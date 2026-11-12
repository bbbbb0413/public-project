---
id: SPEC-103
title: 답변 생성 스트리밍 중 새 대화 시작 및 세션 전환 시 이전 작업 취소 연동과 안전장치 추가
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하며, 사용자가 AI 답변을 실시간으로 확인하고 검증할 수 있도록 돕는 것이다. 이를 위해 답변 생성 중단 UI 및 백엔드 취소 감지(SPEC-007, SPEC-015), 대화 세션 복원(SPEC-011), 대화 세션 키워드 검색(SPEC-035) 등이 배송되었다. 그러나 사용자가 답변 생성(스트리밍) 도중 다른 대화를 탐색하거나 새 대화를 시작하는 주요 흐름에서 심각한 상태 오염 및 안전장치 부재 결함이 존재한다.
첫째, `public-front/src/components/AiService.tsx:412-424` (`handleNewChat`)을 보면 새 대화 시작 시 `sessionId`와 `chatLog` 등의 화면 상태만 초기화할 뿐, 현재 실행 중인 활성 스트림 취소 핸들러(`activeStreamCancelRef.current`)를 호출하지 않고 `isStreaming` 상태도 해제하지 않는다. `public-front/src/api/ai.ts:510-523` (`cancel`) 및 게이트웨이 `public-server/apps/gateway/src/ai/job/job.controller.ts:74-88` (`JobController.cancelJob`)을 통해 백엔드 취소 API(`DELETE /ai/jobs/:jobId`)를 호출하여 불필요한 LLM 토큰 생성을 중단하는 파이프라인이 이미 마련되어 있음에도 불구하고 새 대화 진입 시에는 이 안전장치가 전혀 연동되지 않는다. 이로 인해 백그라운드에서 실행되던 이전 스트림의 SSE 이벤트가 계속 수신되어 비워진 새 대화창에 이전 답변 토큰이 갑자기 타이핑되며 나타나고, 스트리밍 완료 콜백(`AiService.tsx:835-858`)이 실행되어 새 대화창의 첫 턴으로 이전 답변이 강제 주입되는 심각한 화면 오염이 발생한다.
둘째, `public-front/src/components/AiService.tsx:457-485` (`handleLoadSession`)을 보면 사이드바 대화 세션이나 보관함 아이템을 클릭해 다른 세션을 불러올 때도 이전 스트리밍 작업을 취소하지 않는다. 이로 인해 스트리밍 진행 중 사이드바에서 다른 대화를 클릭하면 새로 불러온 세션의 대화 로그(`chatLog`) 끝에 이전 질문의 답변이 추가되어 서로 다른 세션의 대화 내용이 뒤섞이는 데이터 왜곡이 발생한다.
셋째, `public-front/src/components/AiService.tsx:1287-1294` (`AiService`)의 새 대화 시작 버튼(`btn-new-chat`), `public-front/src/components/AiService.tsx:1341-1346` (`AiService`)의 세션 목록 아이템(`session-item`), `public-front/src/components/AiService.tsx:1384-1389` (`AiService`)의 보관함 아이템(`bookmark-item`), `public-front/src/components/AiService.tsx:1362-1370` (`AiService`)의 세션 삭제 버튼(`btn-delete-session`) 모두 스트리밍 진행 중(`isStreaming === true`)일 때 클릭을 차단하는 비활성화(`disabled`) 처리가 누락되어 있다. 사용자가 답변 생성이 완료되기 전에 실수로 사이드바의 버튼을 누르는 즉시 위에서 기술한 대화 오염 및 의도치 않은 세션 삭제 레이스 컨디션이 유발된다.
따라서 새 대화 시작 및 세션 로드 시 활성 스트림을 즉시 취소하도록 연동하고, 스트리밍 진행 중 사이드바의 대화 전환 및 삭제 인터랙션에 안전장치(비활성화)를 적용하여 대화 데이터의 무결성과 안정적인 사용자 경험을 보장해야 한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `handleNewChat` 실행 시, 활성 스트림 취소 핸들러(`activeStreamCancelRef.current`)가 존재하면 이를 즉시 호출하여 백엔드 잡 취소(`DELETE /ai/jobs/:jobId`) 및 SSE 구독 중단을 실행한다.
- [ ] `handleNewChat` 실행 시 `activeStreamCancelRef.current = null`과 `setIsStreaming(false)`를 설정하여 스트리밍 관련 상태를 완전히 초기화하고, 취소된 이전 스트림의 콜백(`onMessage`, `onDone`, `onError`)이 새 대화창의 `chatLog`나 `streamingAnswer`를 오염시키지 않도록 차단한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleLoadSession` 실행 시, 활성 스트림 취소 핸들러(`activeStreamCancelRef.current`)가 존재하면 세션 상세 조회를 수행하기 전에 이를 먼저 호출하여 이전 작업을 안전하게 취소하고 `isStreaming` 상태를 해제한다.
- [ ] `public-front/src/components/AiService.tsx`의 사이드바 새 대화 시작 버튼(`btn-new-chat`)에 `disabled={isStreaming}` 속성 및 비활성화 스타일을 적용하여 스트리밍 중 중복/의도치 않은 클릭을 방지한다.
- [ ] `public-front/src/components/AiService.tsx`의 사이드바 세션 목록 아이템(`session-item`) 및 보관함 아이템(`bookmark-item`)에 스트리밍 진행 중(`isStreaming === true`) 클릭 시 세션 로드를 무시하는 가드를 추가하고, 비활성화 CSS 클래스(`disabled`)를 적용한다.
- [ ] `public-front/src/components/AiService.tsx`의 사이드바 세션 삭제 버튼(`btn-delete-session`)에 `disabled={isStreaming}` 속성을 적용하여 스트리밍 진행 중 대상 세션 삭제 시도를 차단한다.
- [ ] 스트리밍 취소 후 새 대화로 전환되었을 때 `sessionId`가 `null`인 상태에서 이전 스트림 완료 이벤트가 새 세션을 자동 생성(`fetchSessions()`)하거나 엉뚱한 턴을 영속화하지 않도록 방어 로직을 유지한다.
- [ ] 스트리밍 중 새 대화 시작 시 취소 호출 및 상태 초기화, 세션 전환 시 이전 스트림 취소, 스트리밍 진행 중 사이드바 액션 비활성화 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 작업 취소 API(`DELETE /ai/jobs/:jobId`) 규격이나 Kafka 컨슈머의 취소 감지 로직을 수정하지 않는다. (SPEC-015에서 기배송된 규격을 유지)
- 세션 상세 조회 API(`GET /rag/sessions/{id}`) 및 세션 목록 조회 API의 규격을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 관리자 화면(`UserManagement.tsx`, `LlmMonitor.tsx`) 컴포넌트의 동작을 수정하지 않는다.
- AI 답변 마크다운 렌더링 스타일이나 기존 인라인 중단 버튼(`chat-cancel-btn`)의 동작을 변경하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- `handleNewChat` (라인 412~424) 보완:
  - `activeStreamCancelRef.current`가 존재하면 `activeStreamCancelRef.current()`를 호출한다.
  - `activeStreamCancelRef.current = null;`로 참조를 정리한다.
  - `setIsStreaming(false);`를 호출하여 스트리밍 상태 플래그를 정상 초기화한다.
- `handleLoadSession` (라인 457~485) 보완:
  - 함수 진입 시 `activeStreamCancelRef.current`가 존재하면 먼저 `activeStreamCancelRef.current()`를 호출하고 `activeStreamCancelRef.current = null;`, `setIsStreaming(false);`를 설정한다.
  - 스트리밍 중단 후 대상 세션 상세(`getSessionDetail(sid)`)를 안전하게 불러와 `chatLog`를 교체한다.
- `startQuestionStreaming` 콜백 가드:
  - 활성 스트리밍 실행 회차를 식별할 수 있는 토큰/ID(예: `streamingId` 또는 플래그)를 활용하여, 취소되거나 세션이 교체된 이후 도착하는 잔여 콜백(`onMessage`, `onDone`)이 현재 `chatLog`나 `streamingAnswer`를 덮어쓰지 않도록 무시 처리한다.
- 사이드바 새 대화 버튼 UI 개선 (라인 1287~1294):
  - `<button className="btn-new-chat" onClick={handleNewChat} disabled={isStreaming} title="새 대화 시작">` 형태로 `disabled={isStreaming}` 속성을 바인딩한다.
- 사이드바 세션 및 보관함 아이템 인터랙션 안전장치 (라인 1341~1346, 1384~1389):
  - 아이템 클릭 핸들러에 `if (isStreaming) return;` 가드를 적용한다.
  - `className`에 `isStreaming ? ' disabled' : ''`를 추가하여 마우스 포인터 스타일(`cursor: not-allowed`) 및 시각적 비활성화 스타일을 부여한다.
- 사이드바 세션 삭제 버튼 안전장치 (라인 1362~1370):
  - `<button className="btn-delete-session" disabled={isStreaming} onClick={(e) => handleDeleteSessionClick(s, e)} ...>` 형태로 `disabled={isStreaming}`을 적용한다.

`public-front/src/components/AiService.css`
- `.btn-new-chat:disabled`: 비활성화 시 투명도(`opacity: 0.5`), `cursor: not-allowed` 스타일 추가.
- `.session-item.disabled`: 스트리밍 중 세션/북마크 항목에 대해 `opacity: 0.6`, `cursor: not-allowed`, hover 효과 억제 스타일 추가.
- `.btn-delete-session:disabled`: 비활성화 시 클릭 차단 및 투명도 스타일 추가.

`public-front/src/components/AiService.test.tsx`
- 스트리밍 진행 중 `handleNewChat`이 호출되었을 때 `activeStreamCancelRef`가 정상 호출되고 `isStreaming`이 `false`로 전환되는지 검증하는 단위 테스트 추가.
- 스트리밍 진행 중 `handleLoadSession`이 호출되었을 때 이전 스트림 취소가 선행되고 새 세션 데이터로만 `chatLog`가 구성되는지 검증하는 단위 테스트 추가.
- `isStreaming === true`일 때 새 대화 버튼 및 세션 삭제 버튼이 `disabled` 상태인지, 세션 아이템 클릭 시 `handleLoadSession`이 트리거되지 않는지 검증하는 단위 테스트 추가.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 질문을 전송하여 답변이 스트리밍 중인 상태(`isStreaming === true`)일 때, When 사용자가 새 대화 시작 버튼(`btn-new-chat`)을 클릭하거나 `handleNewChat`이 호출되면, Then 활성 스트림 취소 핸들러(`cancel()`)가 호출되고 백엔드 잡 취소 요청이 전송되며, `isStreaming`이 `false`로 정리되고 새 대화창에 이전 질문의 잔여 답변이 노출되지 않아야 한다.
- Given 답변 스트리밍이 진행 중인 상태일 때, When 사용자가 사이드바의 다른 대화 세션을 클릭하여 `handleLoadSession`을 호출하면, Then 진행 중이던 이전 스트림이 즉시 취소되고, 대상 세션의 대화 내용만 화면에 표시되며 이전 스트리밍 답변이 대상 세션 로그에 섞이지 않아야 한다.
- Given 질문에 대한 답변이 실시간으로 생성되는 중(`isStreaming === true`)일 때, When 사이드바 UI를 확인하면, Then 새 대화 시작 버튼(`btn-new-chat`)과 세션 삭제 버튼(`btn-delete-session`)이 비활성화(`disabled`) 상태여야 하며, 세션 목록 및 보관함 아이템 클릭 시 세션 로드가 차단되어야 한다.
- Given 스트리밍이 정상 완료된 상태(`isStreaming === false`)이거나 아직 대화가 시작되지 않은 초기 상태(경계 케이스)일 때, When 새 대화 버튼이나 세션 목록 아이템을 클릭하면, Then 버튼이 활성화되어 정상적으로 새 대화창이 열리거나 해당 세션이 오류 없이 로드되어야 한다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:412-424` (`handleNewChat`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:457-485` (`handleLoadSession`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1287-1294` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1341-1346` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1362-1370` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1384-1389` (`AiService`)
- 관련 정의나 기존 구현: `public-front/src/components/AiService.tsx:774-794` (`handleCancelStreaming`)
- 관련 정의나 기존 구현: `public-front/src/api/ai.ts:510-523` (`cancel`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/job/job.controller.ts:74-88` (`JobController.cancelJob`)
