---
id: SPEC-091
title: RAG 질문 스트리밍 실패 시 원인별 세부 에러 메시지 노출 및 상단 에러 배너 덮어쓰기 수정
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 사용자가 질문을 전송하고 스트리밍 답변을 수신하는 과정에서 오류가 발생하면, `public-front/src/components/AiService.tsx:710-732` (`startQuestionStreaming`) 내부의 에러 콜백에서 `errorDetails` 변수를 계산한다. 이 변수는 `error instanceof Error ? error.message : typeof error === 'string' ? error : '답변 수신 도중 에러가 발생했습니다.'` 형태로 오류 객체로부터 구체적인 오류 원인을 추출하려 시도한다.
그러나 바로 다음 행인 `public-front/src/components/AiService.tsx:718` (`startQuestionStreaming`)에서 상단 공통 에러 배너 상태에 `setErrorMsg('답변 수신 도중 에러가 발생했습니다.')`를 하드코딩으로 강제 대입한다. 이로 인해 화면 상단 공통 에러 배너(`public-front/src/components/AiService.tsx:876` (`AiService`))에는 가드레일 정책 차단, 네트워크 타임아웃, 토큰 초과 등 실제 발생 원인과 무관하게 항상 동일한 일반 문구만 노출된다.
더불어 `public-front/src/api/ai.ts:544-547` (`askQuestionStream` 내 `handleEvent`) 및 `public-front/src/api/ai.ts:557-564` (`askQuestionStream` 내 `run`의 `catch` 블록)을 살펴보면, 잡 생성 HTTP 호출(`POST /ai/rag/jobs`)이 실패할 경우 백엔드가 반환한 응답 본문(예: `error.response?.data?.message` 또는 `error.response?.data?.error`)을 참조하지 않고 Axios의 일반 상태 코드 문자열(`error.message`)만 전달하거나, SSE 스트림 `error` 이벤트의 페이로드가 객체 형태일 때 방어적 문자열 추출이 미흡하다.
또한 백엔드 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:87-101` (`AskRequestedConsumer._process`, `_consume_loop`)에서는 프롬프트 인젝션 및 보안 정책 위반 시 `"요청이 보안 정책에 의해 차단되었습니다."`라는 명확한 사유를 이벤트로 발행하고 있으나, 화면 상단 배너에는 이 내용이 전달되지 않는다.
마지막으로 사용자가 실패한 질문을 [다시 시도](`public-front/src/components/AiService.tsx:762-780` (`handleRetryQuestion`))하거나 새로운 질문을 전송(`public-front/src/components/AiService.tsx:749-758` (`handleSendQuestion`))할 때 기존의 상단 에러 배너 상태(`errorMsg`)를 초기화하지 않아, 새로운 요청이 진행 중이거나 성공한 후에도 이전의 에러 배너가 화면에 계속 남아있는 시각적 불일치가 발생한다.

## 요구사항
- [ ] 질문 스트리밍 실패 시(`startQuestionStreaming` 내 에러 콜백), 상단 공통 에러 배너(`setErrorMsg`)에 하드코딩된 단일 문구 대신 추출된 세부 에러 메시지(`errorDetails`)를 전달하여 반영한다.
- [ ] 에러 메시지 추출 로직을 개선하여, 잡 생성 HTTP 요청 실패 시 Axios 응답 데이터 내 상세 메시지(`response?.data?.message` 또는 `response?.data?.error`)가 존재하면 이를 최우선으로 반영한다.
- [ ] SSE 스트림 에러 이벤트(`event.type === 'error'`) 수신 시 `event.data`가 JSON 문자열이거나 객체인 경우(`error` 또는 `message` 속성 포함) 안전하게 텍스트를 추출하여 전달한다.
- [ ] 에러 원인이 명시되지 않았거나 빈 문자열인 경우에만 안전한 기본 폴백 문구("답변 수신 도중 에러가 발생했습니다.")를 사용한다.
- [ ] 새로운 질문 전송 시(`handleSendQuestion`), 이전에 남아있던 상단 에러 배너를 즉시 초기화(`setErrorMsg('')`)한다.
- [ ] 실패한 질문에 대해 [다시 시도] 버튼 클릭 시(`handleRetryQuestion`), 이전에 남아있던 상단 에러 배너를 즉시 초기화(`setErrorMsg('')`)한다.
- [ ] 인라인 에러 메시지 버블(`msg.errorMessage`)과 화면 상단 에러 배너(`errorMsg`)가 동일한 세부 에러 메시지를 일관되게 표시하도록 동기화한다.
- [ ] 보안 가드레일 차단 메시지("요청이 보안 정책에 의해 차단되었습니다."), API 응답 에러 메시지, 일반 네트워크 오류 메시지의 노출과 재시도 시 배너 초기화를 검증하는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드(`public-python-server`)의 가드레일 정책(`RagContentValidator`) 검사 규칙이나 시크릿 키 마스킹 정규식 자체는 변경하지 않는다. 기존 백엔드가 발행하는 에러 메시지 규격을 그대로 소비한다.
- 게이트웨이(`public-server`)의 `JobController` 및 `JobStreamController`의 REST/SSE 엔드포인트 명세나 gRPC 인터페이스는 수정하지 않는다.
- 에러 발생 시 클라이언트 단에서 자동으로 지수 백오프 재요청을 수행하는 자동 재시도 로직은 도입하지 않으며, 사용자가 직접 [다시 시도] 또는 [질문 복원]을 누르는 기존 수동 흐름을 유지한다.
- 지식베이스 파일 업로드나 결제 내역 조회 등 AI 질문 스트리밍 이외 도메인의 에러 처리 로직은 건드리지 않는다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `startQuestionStreaming` 내부의 에러 콜백(`onError`)에서 `setErrorMsg(errorDetails)`를 호출하여 상단 에러 배너에 구체적인 오류 원인을 반영
  - 에러 객체로부터 메시지를 추출하는 헬퍼 함수(`getErrorMessage(error: unknown): string`)를 구성하거나 `errorDetails` 계산 시 Axios 에러 응답(`isAxiosError`, `error.response?.data?.message`, `error.response?.data?.error`)을 검사하여 상세 메시지 우선 채택
  - `handleSendQuestion` 진입부 및 `handleRetryQuestion` 진입부에 `setErrorMsg('')`를 호출하여 이전 상단 에러 배너를 즉시 클리어
- `public-front/src/api/ai.ts`:
  - `askQuestionStream` 내부의 SSE `handleEvent`에서 `event.type === 'error'` 처리 시, `event.data`가 JSON 객체 또는 JSON 문자열인 경우 `data.message` 또는 `data.error`를 안전하게 추출하여 `onError(new Error(extractedMessage))`로 전달
  - `askQuestionStream`의 `run` 함수 `catch` 블록에서 Axios 에러 발생 시 상세 메시지를 보존하여 `onError`로 전달
- `public-front/src/components/AiService.test.tsx`:
  - 가드레일 차단 이벤트(`"요청이 보안 정책에 의해 차단되었습니다."`) 수신 시 상단 에러 배너와 인라인 에러 버블에 동일하게 해당 메시지가 노출되는지 검증
  - HTTP 400 등 잡 생성 API 에러 발생 시 백엔드 응답 메시지가 상단 에러 배너 및 인라인 버블에 정확히 노출되는지 검증
  - 에러가 표시된 상태에서 사용자가 [다시 시도] 또는 새 질문을 전송할 때 상단 에러 배너(`errorMsg`)가 깨끗이 초기화되는지 검증
  - 상세 메시지가 없는 알 수 없는 에러 발생 시 기본 안내 문구("답변 수신 도중 에러가 발생했습니다.")로 안전하게 폴백되는지 검증

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 보안 정책에 위배되는 질문을 전송하여 백엔드로부터 `"요청이 보안 정책에 의해 차단되었습니다."`라는 에러 이벤트를 수신했을 때, When 스트리밍 에러 처리가 완료되면, Then 상단 공통 에러 배너와 실패 메시지 인라인 버블 모두에 `"요청이 보안 정책에 의해 차단되었습니다."` 텍스트가 정확히 노출된다.
- Given RAG 잡 생성 API 호출 시 백엔드로부터 HTTP 400 상태와 함께 `{ message: "질문 내용이 유효하지 않습니다." }` 응답을 받았을 때, When 에러 핸들러가 동작하면, Then 상단 공통 에러 배너와 인라인 에러 버블에 하드코딩된 문구 대신 `"질문 내용이 유효하지 않습니다."` 메시지가 렌더링된다.
- Given 스트리밍 실패로 상단 에러 배너에 에러 메시지가 표시되어 있는 상태에서, When 사용자가 해당 에러 버블의 [다시 시도] 버튼을 클릭하거나 하단 입력창에 새 질문을 입력하고 전송하면, Then 즉시 상단 에러 배너가 화면에서 사라지고(`errorMsg === ''`) 재시도 스트리밍이 시작된다.
- Given SSE 에러 이벤트나 API 에러 객체에 구체적인 원인 메시지가 누락되었거나 빈 문자열(`""`)로 전달되었을 때, When 에러 처리가 실행되면, Then 에러 배너와 인라인 버블에 안전한 기본 문구인 `"답변 수신 도중 에러가 발생했습니다."`가 폴백으로 표시된다. (경계 케이스: 상세 사유 누락 또는 빈 값)
- Given 스트리밍 도중 일부 텍스트가 누적 수신되다가 네트워크 단절(`"네트워크 연결이 끊어졌습니다."`)이 발생했을 때, When 에러가 처리되면, Then 지금까지 수신된 텍스트는 인라인 버블에 그대로 보존되고 상단 에러 배너 및 인라인 에러 안내 영역에 `"네트워크 연결이 끊어졌습니다."`가 일관되게 표시된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:710-732` (`startQuestionStreaming`)
- 상단 에러 배너 렌더링부: `public-front/src/components/AiService.tsx:876` (`AiService`)
- 질문 전송 및 재시도 핸들러: `public-front/src/components/AiService.tsx:749-758` (`handleSendQuestion`), `public-front/src/components/AiService.tsx:762-780` (`handleRetryQuestion`)
- 스트림 이벤트 파싱 및 에러 방출부: `public-front/src/api/ai.ts:544-547` (`askQuestionStream` 내 `handleEvent`), `public-front/src/api/ai.ts:557-564` (`askQuestionStream` 내 `run` catch 블록)
- 백엔드 에러 이벤트 발행 구현: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:87-101` (`AskRequestedConsumer._process`, `_consume_loop`)
- 게이트웨이 스트림 전달부: `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39-50` (`JobStreamController.stream`)
- 이전 관련 작업: SPEC-073 (`docs/specs/073-qa-retry-failed-answer.md`, AI 답변 생성 실패 시 인라인 에러 표시 및 질문 재시도 지원)
