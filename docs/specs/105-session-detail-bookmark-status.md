---
id: SPEC-105
title: 단건 대화 세션 상세 조회 시 북마크 상태(isBookmarked) 전달 및 상단 헤더 보관 연동
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 시스템은 대화 세션 북마크 기능(SPEC-080)과 답변 북마크 보관함(SPEC-082, SPEC-090)을 제공하여 사용자가 신뢰할 수 있는 RAG 질의응답 이력을 보관하고 추후 다시 검토할 수 있도록 지원하고 있다.
`public-python-server/src/ai_service/rag/schemas.py:228-237` (`ConversationSession.with_bookmark`)와 `public-python-server/src/ai_service/rag/schemas.py:419-439` (`SessionOut.from_domain`)에 명시된 바와 같이, 도메인 모델과 세션 목록 DTO(`SessionOut`)에는 `isBookmarked` 필드가 정상적으로 영속화되어 전달된다.
하지만 단건 세션의 상세 정보를 반환하는 `public-python-server/src/ai_service/rag/schemas.py:475-492` (`SessionDetailOut.from_domain`) 및 `public-python-server/src/ai_service/rag/router.py:44-59` (`get_session`)에는 `is_bookmarked` 필드가 누락되어 있다.
게이트웨이 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:58-68` (`RagSessionProxyController.getSession`)를 거쳐 클라이언트로 도달하는 `public-front/src/api/ai.ts:317-323` (`SessionDetailOut`) 인터페이스와 `public-front/src/api/ai.ts:369-373` (`getSessionDetail`) 응답에도 해당 필드가 존재하지 않는다.
이로 인해 다음과 같은 결함과 사용성 문제가 발생한다:
- 보관함 탭에서 북마크된 세션을 클릭하여 복원하거나(`public-front/src/components/AiService.tsx:1440`), 사이드바 페이징 범위 밖의 세션을 상세 조회로 불러올 때(`public-front/src/components/AiService.tsx:476-505` (`handleLoadSession`)), 해당 세션이 북마크된 세션인지 클라이언트가 판단할 수 없다.
- 대화 모달 상단 헤더 `public-front/src/components/AiService.tsx:1471-1485` (`chat-modal-header`)에 현재 활성화된 세션의 제목과 북마크 상태 표시가 부재하여, 사용자가 지금 보고 있는 대화가 보관 중인지 즉시 알 수 없고 상단에서 원클릭으로 보관을 등록하거나 해제할 수 없다.
- 사이드바 세션 목록에서 북마크를 토글(`public-front/src/components/AiService.tsx:277-292` (`handleToggleSessionBookmark`))하더라도, 현재 열려 있는 활성 세션 화면의 상태와 유기적으로 동기화되지 않는다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `SessionDetailOut` 모델에 `is_bookmarked: bool = Field(default=False, alias="isBookmarked")` 필드를 추가한다.
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `SessionDetailOut.from_domain` 매핑 메서드에서 `isBookmarked=session.is_bookmarked`를 전달하여 세션 도메인의 북마크 상태를 응답에 포함한다.
- [ ] 레거시 MongoDB 세션 문서이거나 `isBookmarked` 필드가 없는 오래된 세션 데이터에 대해 기본값 `False`로 안전하게 직렬화되도록 보장한다.
- [ ] `public-front/src/api/ai.ts`의 `SessionDetailOut` TypeScript 인터페이스에 `isBookmarked?: boolean;` 속성을 추가한다.
- [ ] `public-front/src/components/AiService.tsx`에 현재 활성화된 세션의 북마크 상태(`isCurrentSessionBookmarked: boolean`)를 관리하는 상태를 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleLoadSession` 실행 시 `detail.isBookmarked` 값을 `setIsCurrentSessionBookmarked(Boolean(detail.isBookmarked))`로 동기화한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleNewChat` 실행 시 `setIsCurrentSessionBookmarked(false)`로 초기화한다.
- [ ] `public-front/src/components/AiService.tsx`의 대화 모달 헤더(`chat-modal-header`)에 현재 활성 세션이 존재할 때(`sessionId`가 존재할 때) 세션 제목(또는 기본 안내)과 함께 세션 북마크 토글 버튼(`btn-session-bookmark-header`)을 렌더링한다.
- [ ] 상단 헤더 북마크 버튼(`btn-session-bookmark-header`) 클릭 시 `handleToggleSessionBookmark`를 호출하여 세션 보관 상태를 토글하고, 토스트 메시지 피드백 및 사이드바 `sessions` 목록 내 해당 세션의 `isBookmarked` 상태를 즉시 동기화한다.
- [ ] 사이드바 세션 목록에서 `handleToggleSessionBookmark`가 실행될 때, 토글 대상 세션 식별자가 현재 활성 세션(`sessionId`)과 일치하면 `isCurrentSessionBookmarked` 상태도 함께 갱신한다.
- [ ] 신규 대화 시작 직후 첫 질문 전송 및 완료되어 새 `sessionId`가 생성되었을 때의 초기 세션 북마크 상태는 `false`로 유지한다.

## 비요구사항 (Out of scope)
- 답변 단위 북마크(보관함 메모 모달 등 SPEC-090, SPEC-102 범위)의 스크롤 이동 로직이나 삭제 확인 모달 구조를 수정하지 않는다.
- 대화 세션 검색 알고리즘이나 쿼리 재작성 등 RAG 검색 파이프라인의 백엔드 처리 로직을 수정하지 않는다.
- 게이트웨이 인증 헤더 검증이나 JWT 세션 유효성 판정 로직을 변경하지 않는다.
- 데이터베이스 마이그레이션 스크립트를 추가하지 않으며 기존 MongoDB 컬렉션 스키마를 강제 변환하지 않는다.

## 백엔드
- 대상 저장소: `public-python-server`
- 파일 경로: `src/ai_service/rag/schemas.py`
- 작업 내용:
  - `SessionDetailOut` Pydantic 모델 정의에 `is_bookmarked: bool = Field(default=False, alias="isBookmarked")` 선언 추가.
  - `SessionDetailOut.from_domain(cls, session: ConversationSession)` 메서드 반환 객체 생성자에 `isBookmarked=session.is_bookmarked` 바인딩 추가.
  - 세션 상세 조회 엔드포인트(`GET /rag/sessions/{session_id}`) 호출 시 JSON 응답에 `isBookmarked: true | false`가 올바르게 직렬화되는지 단위 테스트(`tests/rag/test_schemas.py` 또는 `test_session_service.py`)로 검증.

## 프론트엔드
- 대상 저장소: `public-front`
- 파일 경로: `src/api/ai.ts`, `src/components/AiService.tsx`, `src/components/AiService.css`
- 작업 내용:
  - `src/api/ai.ts`: `SessionDetailOut` 인터페이스에 `isBookmarked?: boolean;` 정의 추가.
  - `src/components/AiService.tsx`:
    - `isCurrentSessionBookmarked` boolean 상태(`useState<boolean>(false)`) 선언.
    - `handleLoadSession`: 세션 상세 조회 결과 `detail`에서 `setIsCurrentSessionBookmarked(Boolean(detail.isBookmarked))` 반영.
    - `handleNewChat`: `setIsCurrentSessionBookmarked(false)` 초기화.
    - `handleToggleSessionBookmark`: 현재 활성 `sessionId`와 일치하는 경우 `setIsCurrentSessionBookmarked(nextStatus)` 동기화.
    - `chat-modal-header`: `chat-modal-title-row` 영역 내에 활성 `sessionId`가 존재할 때 세션 북마크 토글 버튼(`btn-session-bookmark-header`) 렌더링. 북마크 여부에 따라 활성 클래스(`bookmarked`) 및 별표 아이콘(`★` / `☆`), 접근성 레이블(`aria-label`) 제공.
  - `src/components/AiService.css`:
    - `.btn-session-bookmark-header` 스타일 추가 (호버 효과, `.bookmarked` 상태 시 황금색/노란색 강조 테마, 포커스 링).

## 수용 기준 (Acceptance Criteria)
- Given 북마크된 대화 세션(`is_bookmarked=True`)이 저장소에 존재할 때,
  When `GET /rag/sessions/{session_id}` 엔드포인트로 단건 상세 조회를 요청하면,
  Then 응답 JSON 본문에 `isBookmarked: true` 필드가 포함되어 반환되어야 한다.
- Given 북마크되지 않은 대화 세션(`is_bookmarked=False`)이거나 `isBookmarked` 필드가 없는 기존 레거시 세션 문서가 존재할 때,
  When `GET /rag/sessions/{session_id}` 엔드포인트로 단건 상세 조회를 요청하면,
  Then 응답 JSON 본문에 `isBookmarked: false` 필드가 기본값으로 포함되어 반환되어야 한다.
- Given 사용자가 사이드바 보관함 탭이나 세션 목록에서 북마크된 세션을 클릭하여 로드했을 때(`handleLoadSession`),
  When 세션 상세 데이터 수신이 완료되면,
  Then 상단 헤더의 세션 북마크 버튼(`btn-session-bookmark-header`)이 활성화 상태(`★`, `.bookmarked`)로 표시되어야 한다.
- Given 현재 대화창에 세션이 로드되어 있고 상단 헤더에 북마크 버튼이 노출되어 있을 때,
  When 상단 헤더의 북마크 버튼(`btn-session-bookmark-header`)을 클릭하면,
  Then 세션 북마크 토글 API(`toggleSessionBookmark`)가 호출되고 상단 버튼 상태가 반전되며, 사이드바 세션 목록 내 해당 세션의 별표 아이콘 및 북마크 탭 목록이 즉시 일치하도록 갱신되어야 한다.
- Given 신규 대화 상태(`sessionId === null`)이거나 아직 질문을 전송하지 않은 빈 대화 화면일 때,
  When 상단 헤더를 확인하면,
  Then 세션 북마크 버튼이 노출되지 않거나 비활성화되어 유효하지 않은 세션에 대한 북마크 요청을 사전에 차단해야 한다 (경계 케이스).

## 참고
- `public-python-server/src/ai_service/rag/schemas.py:475-492` (`SessionDetailOut.from_domain`)
- `public-python-server/src/ai_service/rag/schemas.py:419-439` (`SessionOut.from_domain`)
- `public-python-server/src/ai_service/rag/schemas.py:228-237` (`ConversationSession.with_bookmark`)
- `public-python-server/src/ai_service/rag/router.py:44-59` (`get_session`)
- `public-front/src/api/ai.ts:317-323` (`SessionDetailOut`)
- `public-front/src/api/ai.ts:369-373` (`getSessionDetail`)
- `public-front/src/components/AiService.tsx:476-505` (`handleLoadSession`)
- `public-front/src/components/AiService.tsx:277-292` (`handleToggleSessionBookmark`)
- `public-front/src/components/AiService.tsx:1471-1485` (`chat-modal-header`)
- `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:58-68` (`RagSessionProxyController.getSession`)
