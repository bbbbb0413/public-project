---
id: SPEC-082
title: AI 답변 및 대화 북마크(보관함) API 구현 및 영속화 연동
status: ready
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
SPEC-029를 통해 대화 기록 검색 및 답변 보관(북마크) 기능이 기획 및 배송되었으나, 현재 프론트엔드와 게이트웨이에서 호출하는 개별 답변 보관함 API가 백엔드에 구현되어 있지 않아 기능이 완전히 마비된 상태다.
프론트엔드 `public-front/src/components/AiService.tsx:203-218` (`handleBookmarkAnswer`)에서는 사용자가 AI 답변 버블의 '★ 보관' 버튼을 클릭하면 `createBookmark({ sessionId, turnIndex })`를 호출하여 보관함 저장을 요청하고, 사이드바의 '보관함' 탭(`sidebarTab === 'bookmarks'`)에서 `fetchBookmarks`를 통해 저장된 보관 목록을 조회한다.
프론트엔드 API 클라이언트 `public-front/src/api/ai.ts:338-358` (`getBookmarks`)는 `GET /ai/rag/bookmarks`, `POST /ai/rag/bookmarks`, `DELETE /ai/rag/bookmarks/:bookmarkId` 엔드포인트를 호출하며, `BookmarkItem` 인터페이스(`id`, `sessionId`, `turnIndex`, `title`, `content`, `note`, `createdAt`)를 기대한다.
게이트웨이 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:124-171` (`RagSessionProxyController`) 및 DTO `public-server/apps/gateway/src/ai/proxy/dto/save-bookmark-in.dto.ts:4-19` (`SaveBookmarkInDto`)에서는 인증된 사용자의 `userId`를 파라미터로 주입하여 Python 백엔드의 `rag/bookmarks`로 요청을 프록시하고 있다.
그러나 Python 백엔드 `public-python-server/src/ai_service/rag/router.py:20-33` (`get_sessions`)에는 세션 단위 엔드포인트(`/rag/sessions`)만 정의되어 있고 `/rag/bookmarks` 엔드포인트 및 저장 모델이 전혀 존재하지 않아, 사용자가 답변을 보관하거나 보관함 탭을 열 때마다 HTTP 404 Not Found 오류가 발생하고 화면에 에러 배너가 노출된다.
사용자가 중요한 답변을 보관하고 언제든 다시 찾아볼 수 있는 신뢰할 수 있는 지식 탐색 경험을 보장하기 위해, Python 백엔드에 북마크 영속화 계층 및 REST API 엔드포인트를 구현해야 한다.

## 요구사항
- [ ] `public-python-server`에 북마크 도메인 모델 및 Pydantic 스키마(`BookmarkItemOut`, `CreateBookmarkIn`)를 정의한다.
- [ ] `CreateBookmarkIn`은 `sessionId: str`, `turnIndex: int | None = None`, `note: str | None = None`을 수신한다.
- [ ] `BookmarkItemOut`은 `id: str`, `sessionId: str`, `turnIndex: int | None`, `title: str | None`, `content: str | None`, `note: str | None`, `createdAt: datetime` 필드를 반환한다.
- [ ] MongoDB에 북마크 데이터를 저장하고 조회·삭제하는 리포지토리(`BookmarkRepository`)를 구현한다.
- [ ] 북마크 등록 시 `turnIndex`가 제공되면 해당 세션 턴의 답변 내용을 `content`로 채우고 세션 제목 또는 질문을 `title`로 추출하여 저장한다.
- [ ] 북마크 등록 시 `turnIndex`가 생략되면 세션 자체의 제목을 `title`로 사용하여 세션 단위 북마크로 저장한다.
- [ ] 북마크 조회 시 요청자 `userId`에 해당하는 북마크만 최신 생성일 역순으로 정렬하여 반환하고, `page`, `limit`, 검색어(`q` 또는 `keyword`) 필터링을 지원한다.
- [ ] 북마크 삭제 시 본인 소유(`userId`)의 북마크만 삭제할 수 있도록 소유권 검증(IDOR 방어)을 수행하며, 존재하지 않거나 타인의 북마크인 경우 404를 반환한다.
- [ ] `public-python-server/src/ai_service/rag/router.py` 또는 신규 라우터에 `GET /rag/bookmarks`, `POST /rag/bookmarks`, `DELETE /rag/bookmarks/{bookmark_id}` 엔드포인트를 등록한다.
- [ ] 북마크 등록/조회/삭제 및 사용자 격리 검증을 수행하는 단위 및 E2E 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 프론트엔드 UI 디자인 변경이나 신규 컴포넌트 추가는 진행하지 않는다. (이미 구현된 프론트 연동 사용)
- 게이트웨이의 프록시 라우트나 DTO 변경은 진행하지 않는다. (기존 게이트웨이 계약 준수)
- 세션 단위 기존 북마크 플래그(`isBookmarked`) 필드 구조나 엔드포인트(`/rag/sessions/{id}/bookmark`)는 변경하지 않는다.
- 북마크 항목에 대한 태그 분류나 폴더 생성 기능은 추가하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/schemas.py`:
  - `CreateBookmarkIn` Pydantic 모델 정의 (`sessionId: str`, `turnIndex: int | None = None`, `note: str | None = None`).
  - `BookmarkItemOut` Pydantic 모델 정의 (`id: str`, `sessionId: str = Field(alias="sessionId")`, `turnIndex: int | None = Field(default=None, alias="turnIndex")`, `title: str | None = None`, `content: str | None = None`, `note: str | None = None`, `created_at: datetime = Field(alias="createdAt")`).
- `public-python-server/src/ai_service/rag/repository.py`:
  - `BookmarkRepository` 클래스 구현: MongoDB 컬렉션 `rag_bookmarks`에 북마크 CRUD 수행.
  - `create(bookmark_dict: dict[str, Any]) -> str`: 북마크 생성 후 생성된 ID 반환.
  - `find_by_user_id(user_id: str, page: int, limit: int, query: str | None = None) -> list[dict[str, Any]]`: 소유자별 북마크 페이징 및 키워드 검색 조회.
  - `delete_by_id_for_user(bookmark_id: str, user_id: str) -> bool`: 소유자 검증 기반 단건 삭제.
- `public-python-server/src/ai_service/rag/service.py`:
  - `BookmarkService` 구현: `create_bookmark`, `list_bookmarks`, `delete_bookmark` 메서드 제공. 세션 리포지토리를 참조하여 `turnIndex`에 해당하는 턴의 텍스트(`content`) 및 세션 제목(`title`)을 추출하여 보관.
- `public-python-server/src/ai_service/rag/router.py`:
  - `GET /rag/bookmarks`: `user_id`, `page`, `limit`, `q` 파라미터를 받아 북마크 목록 반환.
  - `POST /rag/bookmarks`: `user_id`, `CreateBookmarkIn` 본문을 받아 북마크 생성 후 `BookmarkItemOut` 반환.
  - `DELETE /rag/bookmarks/{bookmark_id}`: `user_id`, `bookmark_id`를 받아 삭제 수행 후 204 No Content 반환.

## 수용 기준 (Acceptance Criteria)
- Scenario: 답변 북마크 생성 성공
  - Given 사용자가 세션 "sess-1"에서 1번째 턴(어시스턴트 답변: "RAG 검색 결과입니다.")을 확인한 상태에서
  - When `POST /rag/bookmarks?userId=user-1`에 `{ "sessionId": "sess-1", "turnIndex": 1 }` 요청을 보내면
  - Then HTTP 200(또는 201)과 함께 `id`, `sessionId="sess-1"`, `turnIndex=1`, `content="RAG 검색 결과입니다."`가 포함된 북마크 항목이 반환된다.

- Scenario: 본인 북마크 목록 조회 및 검색
  - Given "user-1"이 생성한 북마크 2건(제목: "지식베이스 질의", 내용: "파이프라인 안내")이 존재할 때
  - When `GET /rag/bookmarks?userId=user-1&q=파이프라인` 요청을 보내면
  - Then 검색어 "파이프라인"이 포함된 해당 북마크 1건만 목록에 반환된다.

- Scenario: 타인 북마크 소유권 격리 검증 (IDOR 방어)
  - Given "user-2" 소유의 북마크 "bm-user2-99"가 존재할 때
  - When "user-1"이 `DELETE /rag/bookmarks/bm-user2-99?userId=user-1` 요청을 보내면
  - Then HTTP 404 Not Found 응답이 반환되고 해당 북마크는 삭제되지 않는다.

- Scenario: 경계 케이스 — 존재하지 않는 세션 또는 빈 턴 인덱스
  - Given 데이터베이스에 존재하지 않는 세션 "sess-nonexistent"가 주어졌을 때
  - When `POST /rag/bookmarks?userId=user-1`에 `{ "sessionId": "sess-nonexistent", "turnIndex": 0 }` 요청을 보내면
  - Then 시스템은 에러 없이 저장하거나 적절한 기본 제목을 부여하여 안전하게 처리하거나(또는 404 예외 반환), `turnIndex`가 누락된 경우 세션 단위 북마크로 정상 저장된다.

## 참고
- 게이트웨이 프록시 엔드포인트: `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:124-171` (`RagSessionProxyController`)
- 게이트웨이 북마크 DTO: `public-server/apps/gateway/src/ai/proxy/dto/save-bookmark-in.dto.ts:4-19` (`SaveBookmarkInDto`)
- 프론트엔드 북마크 핸들러: `public-front/src/components/AiService.tsx:203-218` (`handleBookmarkAnswer`)
- 프론트엔드 북마크 API 클라이언트: `public-front/src/api/ai.ts:338-358` (`getBookmarks`)
- 백엔드 RAG 라우터 입구: `public-python-server/src/ai_service/rag/router.py:20-33` (`get_sessions`)
