---
id: SPEC-080
title: 대화 세션 검색 쿼리 파라미터(q) 호환 및 세션 북마크 토글·영속화 연동
status: done
targets: [python-server]
stages: [backend, qa]
priority: high
---
## 배경 / 문제
이미 배송된 대화 기록 검색 및 북마크(SPEC-029, SPEC-035) 작업에서 프론트엔드(`public-front`)와 게이트웨이(`public-server`)는 구현되었으나, 정작 데이터를 저장하고 조회하는 Python 백엔드(`public-python-server`)의 계약이 맞물리지 않아 사이드바 대화 검색과 세션 북마크 토글이 모두 작동하지 않고 있다.
- 프론트엔드 `public-front/src/components/AiService.tsx:141-153` (`fetchSessions`) 및 `public-front/src/api/ai.ts:316-328` (`getSessions`)은 대화 검색어를 `q` 쿼리 파라미터(`params.q = q`)로 전송하고, 게이트웨이 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:35-56` (`RagSessionProxyController.getSessions`) 역시 `@Query('q') q` 또는 `@Query('search') search`를 받아 `params.q`로 전달한다. 그러나 Python 백엔드 `public-python-server/src/ai_service/rag/router.py:16-25` (`get_sessions`)는 `keyword: str | None = Query(default=None)`만 받고 `q` 파라미터를 무시하므로, 사용자가 사이드바 검색창(`public-front/src/components/AiService.tsx:982-1011`, `AiService`)에서 검색어를 입력해도 필터링이 전혀 수행되지 않는다.
- 또한 프론트엔드 `public-front/src/components/AiService.tsx:183-198` (`handleToggleSessionBookmark`) 및 `public-front/src/components/AiService.tsx:1039-1046` (`AiService`)은 각 세션 항목의 북마크 별표 버튼 클릭 시 `public-front/src/api/ai.ts:330-336` (`toggleSessionBookmark`)을 통해 `PATCH /ai/rag/sessions/:sessionId/bookmark`를 호출하고, 게이트웨이 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:70-109` (`RagSessionProxyController.updateSessionBookmark`, `bookmarkSession`, `unbookmarkSession`)가 이를 Python 백엔드의 `/rag/sessions/{sessionId}/bookmark`로 프록시한다.
- 하지만 `public-python-server/src/ai_service/rag/router.py:16-57` (`get_sessions`, `get_session`, `delete_session`)에는 `/rag/sessions/{session_id}/bookmark` 엔드포인트가 아예 없고, `public-python-server/src/ai_service/rag/schemas.py:114-224` (`RestoreProps`, `ConversationSession`)와 `public-python-server/src/ai_service/rag/schemas.py:372-386` (`SessionOut`), `public-python-server/src/ai_service/rag/repository.py:92-142` (`ConversationSessionRepository._to_domain`, `_to_record`)에도 `is_bookmarked`(`isBookmarked`) 필드가 없어 북마크 클릭 시 404 에러가 발생하고 목록 재조회 시 북마크 상태가 유지되지 않는다.

이번 분기 핵심 방향인 "답변을 믿을 수 있게 만든다"에서 사용자가 검증된 과거 RAG 답변 세션을 검색하고 중요 세션을 북마크해 다시 확인하는 흐름이 계약 불일치로 끊겨 있으므로, Python 백엔드의 세션 검색 파라미터 호환과 세션 북마크 영속화·토글 엔드포인트를 연결해야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `RestoreProps` 데이터클래스와 `ConversationSession` 도메인 클래스에 `is_bookmarked: bool = False` 속성을 추가하고, `with_bookmark(self, bookmarked: bool) -> "ConversationSession"`(또는 동등한 상태 변경 메서드)을 제공한다.
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `SessionOut` 모델에 `is_bookmarked: bool = Field(default=False, alias="isBookmarked")` 필드를 추가하고 `SessionOut.from_domain`에서 `session.is_bookmarked`를 매핑한다.
- [ ] `public-python-server/src/ai_service/rag/schemas.py`에 세션 북마크 요청·응답 모델(`ToggleBookmarkIn`: `bookmarked: bool | None = None`, `SessionBookmarkOut`: `session_id: str = Field(alias="sessionId")`, `bookmarked: bool`)을 추가한다.
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `ConversationSessionRepository._to_record`에서 `isBookmarked` 불리언 값을 저장하고, `_to_domain`에서 `bool(record.get("isBookmarked", False))`로 복원하여 기존 MongoDB 레거시 문서와 하위 호환성을 유지한다.
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `ConversationSessionRepository.update`에서 `turns`, `updatedAt`과 함께 `isBookmarked` 필드도 `$set`에 포함하여 갱신한다.
- [ ] `public-python-server/src/ai_service/rag/service.py`의 `SessionService`에 소유자 세션을 조회(`find_by_id_for_user`)한 뒤 북마크 상태를 변경·저장하는 `set_bookmark(self, session_id: str, user_id: str, bookmarked: bool | None = None) -> ConversationSession` 메서드를 추가한다. (`bookmarked`가 `None`이면 현재 `session.is_bookmarked`를 반전하고, 세션이 없거나 타인의 세션이면 `SessionNotFoundError`를 발생시킨다.)
- [ ] `public-python-server/src/ai_service/rag/router.py`의 `get_sessions` 엔드포인트(`GET /rag/sessions`)에 `q: str | None = Query(default=None)` 쿼리 파라미터를 추가하여, `keyword`가 없을 때 `q` 값을 검색 키워드로 `service.get_sessions(user_id, page, limit, effective_keyword)`에 전달한다.
- [ ] `public-python-server/src/ai_service/rag/router.py`에 게이트웨이 프록시(`RagSessionProxyController`)와 맞물리는 `PATCH /rag/sessions/{session_id}/bookmark`, `POST /rag/sessions/{session_id}/bookmark`, `DELETE /rag/sessions/{session_id}/bookmark`(204 No Content) 라우트를 추가하고, 타인 세션이거나 존재하지 않는 세션일 때 기존과 동일하게 HTTP 404(`NOT_FOUND_DETAIL`)를 반환한다.
- [ ] `q` 파라미터 검색 호환, 세션 북마크 영속화·복원(레거시 문서 포함), 소유권 격리 기반 북마크 토글 라우트에 대한 단위 및 E2E 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 프론트엔드(`public-front`)와 게이트웨이(`public-server`)의 세션 검색 및 세션 북마크 호출 코드는 이미 구현되어 있으므로 수정하지 않는다.
- 개별 답변 턴 단위의 별도 보관함 컬렉션 및 `/rag/bookmarks` 엔드포인트 신설은 이번 범위에 포함하지 않는다.
- 대화 세션 제목·본문에 대한 전문 검색(Full-Text Search) 인덱스나 형태소 분석기 도입은 하지 않으며, 기존의 정규식 기반 제목 검색 방식을 유지한다.

## 백엔드
- `public-python-server/src/ai_service/rag/schemas.py`:
  - `RestoreProps`(`schemas.py:114-121`)에 `is_bookmarked: bool = False`를 추가한다.
  - `ConversationSession`(`schemas.py:124-224`) 생성자 및 `create`, `restore`, `append_turn`에서 `is_bookmarked` 상태를 보존하고, 북마크 상태를 변경한 새 세션 인스턴스를 반환하는 `with_bookmark(self, bookmarked: bool) -> "ConversationSession"` 메서드와 `is_bookmarked` 프로퍼티를 추가한다.
  - `SessionOut`(`schemas.py:372-386`)에 `is_bookmarked: bool = Field(default=False, alias="isBookmarked")`를 추가하고 `from_domain`에서 매핑한다.
  - `ToggleBookmarkIn(BaseModel)`(`bookmarked: bool | None = None`) 및 `SessionBookmarkOut(BaseModel)`(`session_id: str = Field(alias="sessionId")`, `bookmarked: bool`) 스키마를 정의한다.
- `public-python-server/src/ai_service/rag/repository.py`:
  - `ConversationSessionRepository.update`(`repository.py:68-74`)의 `$set` 대상에 `"isBookmarked": record["isBookmarked"]`를 추가한다.
  - `ConversationSessionRepository._to_domain`(`repository.py:92-114`)에서 `is_bookmarked=bool(record.get("isBookmarked", False))`를 전달하고, `_to_record`(`repository.py:116-142`)에서 `"isBookmarked": session.is_bookmarked`를 포함시킨다.
- `public-python-server/src/ai_service/rag/service.py`:
  - `SessionService`(`service.py:15-35`)에 `set_bookmark(self, session_id: str, user_id: str, bookmarked: bool | None = None) -> ConversationSession`을 추가한다. 내부에서 `await self._repo.find_by_id_for_user(session_id, user_id)`로 소유권을 검증하고, 없으면 `SessionNotFoundError(session_id)`를 던진 뒤 `target_state = (not session.is_bookmarked) if bookmarked is None else bookmarked`로 갱신 후 `await self._repo.update(updated)`를 호출한다.
- `public-python-server/src/ai_service/rag/router.py`:
  - `get_sessions`(`router.py:16-25`)에 `q: str | None = Query(default=None)`을 추가하고 `effective_keyword = keyword if keyword is not None else q`를 `service.get_sessions(user_id, page, limit, effective_keyword)`에 전달한다.
  - `PATCH /{session_id}/bookmark`(`response_model=SessionBookmarkOut`), `POST /{session_id}/bookmark`(`response_model=SessionBookmarkOut`), `DELETE /{session_id}/bookmark`(`status_code=204`) 엔드포인트를 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자 `user-1`의 대화 세션 목록이 존재할 때, When 클라이언트가 `GET /rag/sessions?userId=user-1&page=1&limit=50&q=매출`로 요청하면, Then `get_sessions`가 `q` 파라미터(`"매출"`)를 검색 키워드로 전달하여 제목에 `"매출"`이 포함된 세션 목록만 반환하고 각 항목에 `isBookmarked` 불리언 필드가 포함된다.
- Given 사용자 `user-1` 소유의 세션 `session-1`(`isBookmarked=False`)이 존재할 때, When `PATCH /rag/sessions/session-1/bookmark?userId=user-1`에 `{"bookmarked": true}`(또는 본문 생략)를 보내면, Then 세션의 북마크 상태가 `True`로 저장되고 HTTP 200과 함께 `{"sessionId": "session-1", "bookmarked": true}`를 반환한다.
- Given `isBookmarked` 필드가 없는 기존 레거시 MongoDB 세션 문서가 저장되어 있거나 타인의 세션 ID(`other-session`)로 북마크 변경을 시도할 때(경계 케이스), When 레거시 세션을 조회하면 오류 없이 `isBookmarked: false`로 복원되고, 타인의 세션이나 존재하지 않는 세션에 북마크 변경(`PATCH`/`POST`/`DELETE`)을 요청하면 소유권이 격리되어 둘을 구분할 수 없는 동일한 HTTP 404(`"대화 세션을 찾을 수 없습니다."`)를 반환한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/router.py:16-57` (`get_sessions`, `get_session`, `delete_session`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/service.py:15-35` (`SessionService`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:68-74` (`ConversationSessionRepository.update`), `public-python-server/src/ai_service/rag/repository.py:92-142` (`ConversationSessionRepository._to_domain`, `_to_record`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/schemas.py:114-224` (`RestoreProps`, `ConversationSession`), `public-python-server/src/ai_service/rag/schemas.py:372-386` (`SessionOut`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:35-109` (`RagSessionProxyController.getSessions`, `updateSessionBookmark`, `bookmarkSession`, `unbookmarkSession`), `public-front/src/api/ai.ts:316-336` (`getSessions`, `toggleSessionBookmark`), `public-front/src/components/AiService.tsx:141-198` (`fetchSessions`, `handleToggleSessionBookmark`)
