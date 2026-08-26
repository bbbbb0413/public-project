---
id: SPEC-015
title: 답변 생성 중단 UI 연동 및 백엔드 취소 감지 처리
status: ready
targets: [front, python-server]
stages: [frontend, backend, qa]
priority: high
---

## 배경 / 문제
답변 생성 중단(SPEC-007) 작업이 게이트웨이 엔드포인트에만 반영되고 실제 프론트엔드 UI와 Python 백엔드 스트리밍 루프에는 연동되지 않아, 사용자가 긴 답변 생성을 멈출 수 없고 불필요한 토큰 비용 및 대기 시간이 지속적으로 낭비되고 있다.

1. **프론트엔드 생성 중단 수단 부재**: `public-server/apps/gateway/src/ai/job/job.controller.ts:60-84` 에 `DELETE /ai/jobs/:jobId` 및 `DELETE /ai/rag/jobs/:jobId` 취소 엔드포인트가 열려 있고 `public-server/apps/gateway/src/ai/job/job-store.service.ts:51-57` 에서 Redis `job:{jobId}:cancelled` 키를 기록하도록 구현되어 있다. 그러나 `public-front/src/components/AiService.tsx:644-651` 의 입력 영역에서는 스트리밍 중(`isStreaming === true`) 버튼이 단순히 "답변 중..."으로 비활성화될 뿐 사용자가 생성을 취소할 수 있는 "중단" 버튼이 없다.
2. **프론트엔드 스트리밍 취소 메커니즘 누락**: `public-front/src/api/ai.ts:198-235` 의 `askQuestionStream` 함수는 SSE `fetch` 스트림을 수신하지만 `AbortController` 지원이나 취소 API(`DELETE /ai/jobs/:jobId`) 호출 기능이 없어 사용자가 화면에서 벗어나거나 중단을 원해도 연결을 즉시 끊지 못한다.
3. **Python 백엔드 취소 플래그 미감지**: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:151-159` 의 토큰 스트리밍 루프(`async for chunk in stream:`)에서 Redis에 설정된 취소 플래그(`job:{job_id}:cancelled`)를 확인하지 않는다. 이로 인해 클라이언트가 연결을 끊더라도 백엔드 LLM 및 에이전틱 루프가 끝까지 토큰을 생성하여 리소스가 낭비된다.

이 결함은 이미 배송 완료로 기록된 SPEC-007(답변 생성 중단)의 성과를 지우고 있는 핵심 결함이며, 이번 분기 제품 방향인 "답변을 믿고 제어할 수 있는 경험"에 직접적으로 연결된다.

## 요구사항
- [ ] `public-front/src/api/ai.ts` 의 `askQuestionStream` 에 작업 취소 기능(예: `AbortController` 연동 및 `DELETE /ai/jobs/:jobId` 호출)을 지원하는 취소 핸들러 또는 반환 객체를 추가한다.
- [ ] `public-front/src/components/AiService.tsx` 에서 스트리밍 진행 중(`isStreaming === true`)일 때 전송 버튼 대신 "중단" 버튼을 표시한다.
- [ ] 사용자가 "중단" 버튼을 클릭하면 즉시 스트림 구독을 중단하고 `isStreaming` 상태를 해제하여 입력창을 다시 활성화한다.
- [ ] 중단 시점까지 수신된 스트리밍 텍스트(`accumulated`)는 채팅 로그(`chatLog`)에 그대로 보존하여 사용자가 지금까지 나온 답변을 확인할 수 있도록 한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` 의 스트리밍 토큰 루프에서 Redis 취소 플래그(`job:{job_id}:cancelled`)를 확인하여, 취소 감지 시 토큰 발행 루프를 즉시 조기 종료하고 `publish_done` 을 발행한다.
- [ ] 변경된 프론트엔드 중단 UI/API 동작 및 Python 백엔드 취소 감지 루프에 대한 단위/통합 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 게이트웨이(`public-server`) 수정: `DELETE /ai/jobs/:jobId` 및 `DELETE /ai/rag/jobs/:jobId` 가 이미 정상 구현되어 있으므로 게이트웨이는 수정하지 않는다.
- 중단된 답변에 대한 "이어서 생성하기(Resume)" 기능.
- AI 답변 클립보드 복사 기능: 복사 기능은 이미 `SPEC-008` 로 `AiService.tsx:543-551` 에 완전히 구현 및 배송되어 있으므로 본 작업 범위에 포함하지 않는다.
- 세션 삭제 및 문서 삭제 모달 UI 변경.

## 백엔드
`public-python-server` 에서 아래 파일을 수정한다.

`src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- `_handle_ask` 메서드의 스트리밍 루프(`async for chunk in stream:`, `151-159행`) 내부에서 Redis 클라이언트(`self._redis.get(f"job:{message.job_id}:cancelled")` 또는 Redis 존재 여부 확인)를 통해 작업 취소 여부를 점검한다.
- 취소 플래그가 설정되어 있으면 추가 토큰 발행을 중단하고 루프를 즉시 탈출한다.
- 취소 탈출 후에도 세션 저장 및 `publish_done(message.job_id, done_data)` 를 안전하게 호출하여 스트림 연결을 정상 종료한다.

## 프론트엔드
`public-front` 에서 아래 파일들을 수정한다.

`src/api/ai.ts`
- `askQuestionStream` 함수가 스트림 중단 함수(`cancel: () => Promise<void>`)를 포함하는 제어 객체를 반환하도록 하거나, 취소 콜백을 등록받아 내부 `AbortController.abort()` 및 `client.delete('/ai/jobs/' + job.jobId)` 를 호출하도록 개선한다 (`198-235행`).

`src/components/AiService.tsx`
- 스트리밍 중단 처리를 위한 `handleCancelStreaming` 함수를 구현한다.
- `isStreaming === true` 일 때 입력 폼의 전송 버튼 영역(`644-651행`)에 "중단" 버튼을 렌더링하고 클릭 시 `handleCancelStreaming` 을 호출한다.
- 중단 호출 시 현재까지 수신된 `streamingAnswer` 텍스트를 `chatLog` 에 AI 메시지로 추가하고, `isStreaming` 을 `false` 로 전환하며 입력 필드를 활성화한다.

`src/components/AiService.css`
- "중단" 버튼에 대한 스타일 클래스(예: `.chat-cancel-btn` 또는 위험/중단 인지용 붉은 계열/경고 톤 스타일)를 정의한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 질문을 전송하여 AI 답변이 실시간으로 스트리밍되는 상태일 때
  When 입력 폼 영역을 확인하면
  Then 비활성화된 전송 버튼 대신 클릭 가능한 "중단" 버튼이 표시된다.
- Given 스트리밍 중 사용자가 "중단" 버튼을 클릭했을 때
  When 중단 요청이 실행되면
  Then SSE 스트림 구독이 즉시 종료되고, `DELETE /ai/jobs/:jobId` API가 호출되며, 입력창이 활성화되고 지금까지 수신된 텍스트가 채팅 기록에 보존된다.
- Given 백엔드에서 LLM 토큰을 연속적으로 발행하고 있는 상황에서 Redis에 취소 플래그(`job:{jobId}:cancelled`)가 설정되었을 때
  When 컨슈머가 다음 토큰을 발행하기 전 플래그를 감지하면
  Then 추가 토큰 생성을 즉시 멈추고 `done` 이벤트를 발행하여 작업을 안전하게 마친다.
- Given 스트리밍이 이미 정상 완료되었거나 에러로 종료된 상태(경계 케이스)에서
  When 사용자가 중단을 시도하거나 중단 요청이 지연 도착하면
  Then 백엔드와 프론트엔드 모두 에러 없이 정상 상태를 유지한다.
- Given 질문 입력창이 비어있거나 스트리밍 중이 아닌 일반 상태(경계 케이스)일 때
  When 화면을 확인하면
  Then "중단" 버튼은 노출되지 않고 일반 "전송" 버튼만 노출된다.

## 참고
- 중단 버튼 추가 위치: `public-front/src/components/AiService.tsx:644-651`
- 프론트엔드 스트림 클라이언트: `public-front/src/api/ai.ts:198-235`
- 백엔드 게이트웨이 취소 엔드포인트: `public-server/apps/gateway/src/ai/job/job.controller.ts:60-84`
- 백엔드 Redis 취소 처리: `public-server/apps/gateway/src/ai/job/job-store.service.ts:51-57`
- Python 컨슈머 토큰 스트리밍 루프: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:151-159`
