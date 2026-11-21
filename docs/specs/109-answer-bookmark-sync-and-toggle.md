---
id: SPEC-109
title: AI 답변 보관(북마크) 상태의 대화 세션 턴 연동 및 답변 버블 내 보관 상태 표시·토글 해제 지원
status: done
targets: [front, python-server, server]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하되, 사용자가 답변을 스스로 확인하고 신뢰할 수 있는 답변을 보존·활용할 수 있도록 돕는 도구다. 이를 위해 AI 답변 및 세션 북마크 API와 MongoDB 영속화(SPEC-082), 답변 북마크 시 메모(note) 입력 모달 연동(SPEC-090), 보관함 항목 클릭 시 해당 답변 턴으로의 자동 스크롤 및 하이라이트(SPEC-102), 단건 대화 세션 상세 조회 시 세션 단위 북마크 상태(`isBookmarked`) 전달 및 상단 헤더 보관 연동(SPEC-105) 등이 단계적으로 배송되었다.
그러나 개별 답변 버블에서 답변을 보관하고 확인하는 핵심 상호작용에서 다음과 같은 심각한 상태 단절과 결함이 존재한다:
첫째, `public-front/src/components/AiService.tsx:2130-2139` (`AiService` 내 메시지 액션 렌더링부)를 보면 세션이 존재할 때 AI 답변 버블 하단에 `★ 보관` 버튼(`btn-bookmark-answer`)이 렌더링된다. 하지만 이 버튼은 해당 턴이 이미 보관함에 저장되었는지 여부와 무관하게 항상 `★ 보관` 단일 텍스트와 비활성 스타일로 고정되어 있다. 사용자가 답변을 보관함에 추가했음에도 답변 버블에는 아무런 시각적 변화가 없어, 대화창 내에서 어떤 답변이 보관된 상태인지 전혀 알아챌 수 없다.
둘째, `public-python-server/src/ai_service/rag/service.py:69-97` (`BookmarkService.create_bookmark`) 및 `public-python-server/src/ai_service/rag/repository.py:176-179` (`BookmarkRepository.create`)를 보면, 북마크 등록 시 중복 검증 없이 매번 새로운 문서로 `insert_one`을 실행한다. 프론트엔드 버튼 역시 보관 상태가 표시되지 않으므로, 이미 보관한 답변에 대해 사용자가 `★ 보관` 버튼을 다시 누르면 또다시 메모 입력 모달이 열리고 동일한 세션의 동일 턴에 대한 중복 북마크 레코드가 계속해서 누적 생성되는 데이터 오염이 발생한다.
셋째, `public-front/src/components/AiService.tsx:554-571` (`handleLoadSession`)에서 사이드바 세션 목록을 클릭하여 대화 세션을 복원할 때, 피드백 데이터는 `loadFeedback(detail.sessionId)`로 불러와 각 턴의 평가 상태를 완벽히 복원하지만, 해당 세션에 속한 답변 턴들의 북마크 정보는 전혀 조회하거나 복원하지 않는다. 또한 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:147-156` (`RagSessionProxyController.createBookmark`) 및 Python 백엔드에는 특정 세션에 속한 답변 북마크 목록을 조회하는 엔드포인트(`GET /rag/bookmarks/session/{sessionId}`)가 결여되어 있다.
넷째, 사용자가 이미 보관한 답변을 취소(보관 해제)하려 할 때, 대화창 답변 버블 내에서는 보관 해제 기능이 전혀 제공되지 않는다. 사용자는 반드시 사이드바 보관함 탭으로 이동하여 전체 북마크 목록 중에서 해당 답변을 직접 찾아 `✕` 삭제 버튼을 눌러야만 하는 심각한 사용성 불편을 겪고 있다.
따라서 특정 세션의 답변 턴별 북마크 상태를 조회하는 백엔드 API를 추가하고, 세션 복원 및 보관 등록 시 답변 버블의 버튼을 `★ 보관됨` 활성 상태로 표시하며, 이미 보관된 답변 클릭 시 대화창 내에서 원클릭으로 보관을 해제할 수 있도록 개선해야 한다. 이 개선은 "답변을 믿을 수 있게 만든다"는 분기 방향 아래, 사용자가 검증하고 가치를 인정한 신뢰 답변의 보존 및 관리 경험을 완결 짓는 필수 개선이다.
## 요구사항
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `BookmarkRepository`에 특정 세션에서 사용자가 등록한 턴 단위 북마크 목록을 조회하는 `find_by_session_and_user(session_id: str, user_id: str) -> list[dict[str, Any]]` 메서드를 추가한다.
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `BookmarkRepository.create`에서 동일 사용자, 세션, 턴 인덱스(`userId`, `sessionId`, `turnIndex`)를 가진 북마크가 이미 존재하는 경우, 신규 레코드를 추가 생성하지 않고 기존 문서의 `note`, `content`, `title`, `updatedAt` 필드를 갱신(upsert)하여 반환함으로써 중복 북마크 생성을 방지한다.
- [ ] `public-python-server/src/ai_service/rag/service.py`의 `BookmarkService`에 `get_session_bookmarks(user_id: str, session_id: str) -> list[BookmarkItemOut]` 메서드를 구현한다.
- [ ] `public-python-server/src/ai_service/rag/router.py`에 세션별 북마크 목록 조회 엔드포인트(`GET /rag/bookmarks/session/{session_id}`, response_model=`list[BookmarkItemOut]`)를 추가한다.
- [ ] `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts`에 세션별 북마크 조회를 ai-service-py로 프록시하는 `@Get('bookmarks/session/:sessionId')` 엔드포인트를 추가한다.
- [ ] `public-front/src/api/ai.ts`에 `getSessionBookmarks(sessionId: string): Promise<BookmarkItem[]>` 클라이언트 API 함수를 추가한다.
- [ ] `public-front/src/components/AiService.tsx`에 현재 세션의 턴별 북마크 상태 매핑(`bookmarksByTurn: Record<number, BookmarkItem>`)을 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleLoadSession` 실행 시 `getSessionBookmarks(detail.sessionId)`를 호출하여 세션 내 보관된 턴들을 `bookmarksByTurn`에 동기화한다.
- [ ] `public-front/src/components/AiService.tsx`의 AI 메시지 버블 액션 영역에서, 해당 턴 인덱스(`turnIndex`)가 `bookmarksByTurn`에 존재하는 경우 `btn-bookmark-answer`에 `bookmarked` CSS 클래스를 부여하고 버튼 텍스트를 `★ 보관됨`으로 렌더링한다.
- [ ] 이미 보관된 답변(`bookmarked === true`)의 `★ 보관됨` 버튼 클릭 시 메모 모달을 열지 않고, 해당 북마크 삭제 API(`deleteBookmark(bookmark.id)`)를 호출하여 대화창 내에서 즉각 보관을 해제(삭제)한다.
- [ ] 보관 해제 완료 시 "답변 보관이 해제되었습니다." 토스트 알림을 표시하고, `bookmarksByTurn`에서 해당 턴을 제거하며 버튼을 일반 `★ 보관` 상태로 즉시 복귀시킨다.
- [ ] 새 답변 보관 모달에서 [보관하기] 완료 시, 반환된 북마크 정보를 `bookmarksByTurn`에 즉시 반영하여 해당 턴의 버튼이 즉각 `★ 보관됨` 상태로 전환되도록 한다.
- [ ] 사이드바 보관함 탭에서 북마크 삭제(`handleDeleteBookmarkClick`)가 완료되었을 때, 삭제된 북마크가 현재 열린 세션의 턴인 경우 `bookmarksByTurn`에서도 함께 동기화 제거되어 대화창 버블의 버튼이 `★ 보관`으로 복귀하도록 한다.
- [ ] `handleNewChat` 실행 시 `bookmarksByTurn` 상태를 빈 객체(`{}`)로 초기화한다.
- [ ] `public-front/src/components/AiService.css`에 보관 활성 버튼 스타일(`.btn-bookmark-answer.bookmarked`)을 추가한다 (황색/골드 계열 텍스트 `#facc15`, 배경 `rgba(250, 204, 21, 0.15)`, 테두리 `rgba(250, 204, 21, 0.4)`).
- [ ] 백엔드 세션별 북마크 조회/중복 방지 및 프론트엔드 세션 복원 시 턴별 북마크 표시·토글 해제 동작을 검증하는 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 세션 단위 북마크 토글(`isCurrentSessionBookmarked`, `btn-session-bookmark-header`) 동작이나 DTO 규격을 변경하지 않는다.
- 북마크 메모 입력 모달(`bookmark-note-modal`)의 UI 구조나 글자 수 제한(200자)을 수정하지 않는다.
- 지식베이스 문서 인제스트 또는 벡터 검색 알고리즘을 변경하지 않는다.
- 답변 평가(`AnswerFeedback.tsx`) 폼의 입력 필드나 불만족 사유 태그 목록을 수정하지 않는다.
## 백엔드
`public-python-server/src/ai_service/rag/repository.py`
- `BookmarkRepository`:
  - `find_by_session_and_user(self, session_id: str, user_id: str) -> list[dict[str, Any]]`:
    - 쿼리 조건: `{"sessionId": session_id, "userId": user_id, "turnIndex": {"$ne": None}}`
    - 정렬: `("turnIndex", 1)`
    - 조회된 레코드 목록 반환.
  - `create(self, bookmark_dict: dict[str, Any]) -> str`:
    - `turn_index`가 지정된 답변 북마크인 경우:
      `filter_key = {"sessionId": bookmark_dict["sessionId"], "userId": bookmark_dict["userId"], "turnIndex": bookmark_dict["turnIndex"]}`
      기존 레코드가 존재하면 `$set: {"note": bookmark_dict.get("note"), "content": bookmark_dict.get("content"), "title": bookmark_dict.get("title")}`으로 업데이트하고 해당 문서의 `_id` 문자열을 반환한다.
      존재하지 않는 경우 기존과 동일하게 `insert_one`을 실행한다.

`public-python-server/src/ai_service/rag/service.py`
- `BookmarkService`:
  - `get_session_bookmarks(self, user_id: str, session_id: str) -> list[BookmarkItemOut]`:
    - 세션 소유권 확인 (`await self._session_repo.find_by_id_for_user(session_id, user_id)`)
    - `records = await self._bookmark_repo.find_by_session_and_user(session_id, user_id)`
    - `[BookmarkItemOut.from_dict(r) for r in records]` 반환.

`public-python-server/src/ai_service/rag/router.py`
- `@bookmarks_router.get("/session/{session_id}", response_model=list[BookmarkItemOut], response_model_by_alias=True)`:
  - `get_session_bookmarks` 핸들러 구현. `service.get_session_bookmarks(user_id, session_id)` 호출 및 결과 반환.

`public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts`
- `RagSessionProxyController`:
  - `@Get('bookmarks/session/:sessionId')`
  - `@ApiOperation({ summary: '특정 세션의 답변 북마크 목록 조회 (ai-service-py 프록시)' })`
  - `this.aiServicePy.get({ method: 'rag/bookmarks/session/' + sessionId, params: { userId: req.session.uuid } })` 호출.

## 프론트엔드
`public-front/src/api/ai.ts`
- `getSessionBookmarks(sessionId: string): Promise<BookmarkItem[]>`:
  - `GET /ai/bookmarks/session/:sessionId` 호출 및 결과 반환.

`public-front/src/components/AiService.tsx`
- 상태 관리 추가:
  - `const [bookmarksByTurn, setBookmarksByTurn] = useState<Record<number, BookmarkItem>>({});`
- 세션 북마크 로드 헬퍼:
  - `const loadSessionBookmarks = async (sid: string) => { try { const items = await getSessionBookmarks(sid); const map: Record<number, BookmarkItem> = {}; items.forEach((item) => { if (typeof item.turnIndex === 'number') map[item.turnIndex] = item; }); setBookmarksByTurn(map); } catch { setBookmarksByTurn({}); } };`
- 세션 로드 및 초기화 연동:
  - `handleLoadSession`: `detail` 조회 후 `await loadSessionBookmarks(detail.sessionId);` 호출.
  - `handleNewChat`: `setBookmarksByTurn({});` 호출.
- 보관 등록 완료 연동 (`handleConfirmBookmarkAnswer`):
  - `createBookmark` 응답으로 받은 `newBookmark`를 바탕으로:
    `if (typeof newBookmark.turnIndex === 'number') setBookmarksByTurn(prev => ({ ...prev, [newBookmark.turnIndex!]: newBookmark }));`
- 답변 버블 내 원클릭 보관 해제 핸들러:
  - `const handleToggleAnswerBookmark = async (turnIndex: number, text: string, e?: React.MouseEvent) => { ... }`
  - 해당 턴의 북마크가 이미 존재하면: `await deleteBookmark(existingBookmark.id);` 호출, `setBookmarksByTurn(prev => { const next = { ...prev }; delete next[turnIndex]; return next; });`, `showBookmarkToast('답변 보관이 해제되었습니다.');`, 사이드바 탭이 `bookmarks`인 경우 `fetchBookmarks();` 갱신.
  - 북마크가 없으면 기존 `handleBookmarkAnswer(turnIndex, text, e)` 모달 오픈 흐름 유지.
- 메시지 버블 액션 렌더링 (`chatLog.map`):
  ```tsx
  {sessionId && (
    <button
      type="button"
      className={`btn-bookmark-answer${bookmarksByTurn[turnIndex] ? ' bookmarked' : ''}`}
      onClick={(e) => handleToggleAnswerBookmark(turnIndex, msg.text || '', e)}
      aria-label={bookmarksByTurn[turnIndex] ? '답변 보관 해제' : '답변 보관'}
      title={bookmarksByTurn[turnIndex] ? '보관 해제' : '답변 보관'}
    >
      {bookmarksByTurn[turnIndex] ? '★ 보관됨' : '★ 보관'}
    </button>
  )}
  ```

`public-front/src/components/AiService.css`
- `.btn-bookmark-answer.bookmarked`:
  - 텍스트 색상: `#facc15`
  - 배경: `rgba(250, 204, 21, 0.15)`
  - 테두리: `1px solid rgba(250, 204, 21, 0.4)`
  - 마우스 호버 시: `background: rgba(250, 204, 21, 0.25)`

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 로그인 상태에서 세션 내 AI 답변에 대해 [보관하기]를 완료했을 때,
  When 모달이 닫히고 저장이 성공하면,
  Then 해당 AI 메시지 버블 하단의 버튼 텍스트가 `★ 보관됨`으로 즉시 변경되고 `bookmarked` CSS 클래스가 적용되어야 한다.
- Given 답변이 이미 보관된 상태(`bookmarksByTurn[turnIndex]` 존재)인 AI 메시지 버블에서,
  When 사용자가 `★ 보관됨` 버튼을 클릭하면,
  Then 메모 입력 모달이 열리지 않고 해당 북마크 삭제 API가 호출되어 즉시 보관이 해제되며, 버튼 텍스트가 `★ 보관`으로 복귀하고 "답변 보관이 해제되었습니다." 토스트 메시지가 표시되어야 한다.
- Given 1번 턴 답변을 이미 보관한 대화 세션이 존재할 때,
  When 사용자가 사이드바 대화 목록에서 해당 세션을 클릭하여 복원(`handleLoadSession`)하면,
  Then 세션 상세 및 턴별 북마크 조회가 완료된 후 1번 턴 AI 메시지 버블의 버튼이 `★ 보관됨` 상태로 올바르게 복원되어 렌더링되어야 한다.
- Given (경계 케이스) 세션 내 보관된 답변이 전혀 없는 세션을 불러왔을 때(`getSessionBookmarks` 결과가 빈 배열),
  When 대화창이 렌더링되면,
  Then 자바스크립트 오류 없이 모든 AI 메시지 버블의 버튼이 기본 `★ 보관` 상태로 안전하게 표시되어야 한다.
- Given (경계 케이스) 동일한 세션과 턴 인덱스에 대해 네트워크 재전송 등으로 북마크 생성 API(`createBookmark`)가 중복 호출되었을 때,
  When 백엔드가 요청을 처리하면,
  Then 새 레코드를 추가 삽입하지 않고 기존 문서를 갱신하여 데이터베이스 내 동일 턴 중복 북마크가 생성되지 않아야 한다.
- Given 사이드바 보관함(`bookmarks`) 탭에서 현재 열려 있는 세션의 답변 북마크를 `✕` 버튼으로 삭제했을 때,
  When 삭제 확인 모달에서 삭제가 완료되면,
  Then 대화창의 해당 답변 버블 버튼 역시 `★ 보관됨`에서 일반 `★ 보관` 상태로 즉시 동기화되어야 한다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2130-2139` (`AiService` 내 메시지 액션 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:554-571` (`handleLoadSession`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/service.py:69-97` (`BookmarkService.create_bookmark`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:176-179` (`BookmarkRepository.create`)
- 관련 정의 및 프록시 입구: `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:147-156` (`RagSessionProxyController.createBookmark`)
- 관련 DTO 및 세션 북마크 구현: `public-python-server/src/ai_service/rag/schemas.py:475-495` (`SessionDetailOut.from_domain`)
