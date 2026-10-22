---
id: SPEC-074
title: 답변 생성 중단 시 인라인 중단 상태 보존 및 미완성 안내·질문 복원 지원
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 프론트엔드 RAG 대화 화면(`public-front/src/components/AiService.tsx`)은 스트리밍 답변 생성 도중 사용자가 [중단] 버튼을 클릭했을 때, 중단 시점의 상태를 대화창 내에 적절히 보존하지 못하고 화면과 데이터 상태에 혼선을 초래하는 결함이 있다.

구체적인 문제 지점은 다음과 같다:
- `public-front/src/components/AiService.tsx:626-645` (`handleCancelStreaming`)에서 사용자가 스트리밍 도중 [중단] 버튼을 누르면 `activeStreamCancelRef`를 통해 취소 요청을 보내고 스트림을 중단한다. 이때 토큰이 1개라도 수신되기 전(예: 에이전트의 문서 검색 `searching` 단계나 초기 대기 상태)에 중단을 누르면 `canceledText`가 빈 문자열 `""`이 되어 `if (canceledText)` 조건을 통과하지 못하므로, `chatLog`에 아무런 AI 메시지도 추가되지 않는다. 이로 인해 사용자의 질문 말풍선만 남고 AI 측 응답은 흔적 없이 사라져, 사용자는 생성이 중단된 것인지 요청이 누락된 것인지 알 수 없게 된다.
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 인터페이스에는 중단 상태를 표현하는 플래그(`isCancelled`)가 없어, 일부 텍스트가 수신된 상태에서 중단된 경우에도 `{ sender: 'ai', text: canceledText }` 형태로만 저장된다.
- `public-front/src/components/AiService.tsx:1093-1166` (`AiService`)에서 렌더링될 때, 중간에 문장이 끊긴 불완전한 미완성 답변임에도 불구하고 시각적으로 아무런 중단 배지나 경고 안내가 표시되지 않는다. 이로 인해 사용자가 추후 대화를 다시 보거나 답변을 확인할 때 모델이 완성한 정상 답변으로 오인할 위험이 있다.
- `public-front/src/components/AiService.tsx:1153-1160` (`AiService`)에서 중단된 미완성 메시지 버블 아래에도 `AnswerFeedback` 평가 컴포넌트([이 답변을 평가하기])가 무조건 노출된다. 모델의 추론이나 생성이 완료되지 않고 사용자가 의도적으로 끊은 불완전한 텍스트에 대해 정확도나 유용성 평가를 수집하도록 유도하는 것은 시스템 품질 피드백 데이터를 왜곡시키는 결과를 낳는다.
- 사용자가 답변 생성을 중단하는 주된 원인은 질문을 수정하고 싶거나 오답 흐름을 조기에 감지했기 때문인데, 입력창(`question`)은 이미 비워진 상태이므로 직전 질문 텍스트를 재사용하거나 수정하여 다시 전송하려면 번거롭게 수동 복사·입력해야 하는 불편이 존재한다.
- 이 개선은 "답변을 믿을 수 있게 만든다"는 이번 분기 제품 방향에서 불완전하거나 검증되지 않은 중단 답변을 정상 답변과 시각적으로 명확히 분리하고, 실패나 중단 상황에서도 사용자가 질문을 쉽게 수정·재전송할 수 있도록 지원하는 데 직결된다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `isCancelled?: boolean` 선택적 속성을 추가한다.
- [ ] `handleCancelStreaming` 실행 시, 수신된 텍스트(`canceledText`)가 존재하면 해당 텍스트와 함께 `isCancelled: true` 플래그를 가진 AI 메시지 객체를 `chatLog`에 추가한다.
- [ ] `handleCancelStreaming` 실행 시, 수신된 텍스트가 전혀 없는 초기 상태(`canceledText === ''`)에서 중단되더라도 "답변 생성이 중단되었습니다." 안내 텍스트와 `isCancelled: true` 플래그를 가진 AI 메시지 객체를 `chatLog`에 추가하여 질문만 홀로 남는 현상을 방지한다.
- [ ] 중단된 AI 메시지 버블(`msg.isCancelled === true`) 상단 또는 하단에 `[생성 중단됨]` 시각적 상태 배지 및 미완성 안내 문구를 렌더링한다.
- [ ] 중단된 AI 메시지 버블 내에는 `AnswerFeedback`(답변 평가 폼) 컴포넌트를 렌더링하지 않아 미완성 답변에 대한 피드백 수집 및 데이터 왜곡을 차단한다.
- [ ] 중단된 AI 메시지 버블 내에 [질문 복원] 버튼을 제공하여, 클릭 시 직전 사용자 질문 내용을 하단 입력창(`question`)에 채우고 포커스하여 즉시 수정 후 재전송할 수 있도록 지원한다.
- [ ] 중단된 AI 메시지 버블 내에 [다시 질문] 버튼을 제공하여, 클릭 시 동일한 질문으로 즉시 답변 생성을 재시작할 수 있도록 지원한다.
- [ ] 스트리밍 진행 중(`isStreaming === true`)일 때는 [질문 복원] 및 [다시 질문] 액션 버튼을 비활성화(`disabled`)하여 중복 동시 요청을 방지한다.
- [ ] 정상적으로 생성 완료된 AI 메시지(`msg.isCancelled !== true`)에는 기존과 동일하게 신뢰도 배지, 복사, 보관, 평가 컴포넌트가 유지되며 중단 배지나 복원 버튼은 노출되지 않는다.

## 비요구사항 (Out of scope)
- 백엔드 취소 감지 및 Redis 플래그 처리 로직(`ask_requested_consumer.py`, `job.controller.ts`) 변경 (기존 백엔드 취소 계약 유지)
- 세션 영속화 DB(MongoDB) 스키마 변경 (프론트엔드 클라이언트 세션 뷰 레벨 상태 표시 및 편의성 개선에 집중)
- 네트워크 에러나 예외 발생 시의 오류 처리 변경 (이는 SPEC-073의 범위이며, 본 스펙은 사용자가 명시적으로 [중단]을 누른 정상 취소 흐름에 한정)
- 음성 입력이나 파일 첨부 등 신규 입력 수단 추가

## 프론트엔드
- 대상 컴포넌트: `public-front/src/components/AiService.tsx`
- 대상 스타일: `public-front/src/components/AiService.css`
- 주요 변경 사항:
  - `ChatMessage` 인터페이스 확장:
    ```typescript
    interface ChatMessage {
      sender: 'user' | 'ai';
      text: string;
      confidence?: number;
      missing?: string[];
      isCancelled?: boolean;
    }
    ```
  - `handleCancelStreaming` 취소 핸들러 개선:
    - `activeStreamCancelRef` 호출 후 취소 처리
    - `canceledText` 존재 여부에 따라 실제 수신된 텍스트 또는 기본 안내 텍스트("답변 생성이 중단되었습니다.")를 `text`로 바인딩
    - `{ sender: 'ai', text: ..., isCancelled: true }` 메시지를 `chatLog`에 안전하게 추가
    - 마지막 중단된 질문(`lastCancelledQuestion`)을 추적하거나 직전 사용자 턴(`chatLog` 내 직전 `user` 메시지)을 식별하여 복원/재질문 컨텍스트 확보
  - 재질문 및 복원 핸들러 구현:
    - `handleRestoreQuestion(questionText: string)`: 하단 질문 입력창에 `questionText`를 설정하고 입력창(`chatInputRef`)에 포커스 부여
    - `handleRetryQuestion(questionText: string)`: 중단된 질문으로 `handleSendQuestion` 스트리밍 즉시 재호출
  - 메시지 버블 렌더링 분기:
    - `msg.isCancelled`일 때 `.chat-message.ai.cancelled` 클래스 적용
    - `[생성 중단됨]` 배지 렌더링
    - `AnswerFeedback` 노출 차단 (`!msg.isCancelled && sessionId && <AnswerFeedback ... />`)
    - [질문 복원] 및 [다시 질문] 액션 버튼 렌더링
  - 스타일 추가(`AiService.css`):
    - `.chat-message.ai.cancelled .message-bubble`: 중단 상태를 인지할 수 있는 시각적 스타일(차분한 회색/슬레이트 테두리 및 배경)
    - `.cancelled-badge`: 중단 상태 요약 배지(노란색/주황색 계열 배지 텍스트)
    - `.btn-restore-question`, `.btn-retry-question`: 질문 복원 및 재질문 버튼 스타일
- 테스트 추가(`public-front/src/components/AiService.test.tsx`):
  - 토큰 수신 전(초기 단계) 중단 시 안내 메시지와 함께 중단 AI 버블이 생성되는지 검증
  - 중간 토큰 수신 후 중단 시 기수신된 텍스트와 함께 중단 배지가 표시되는지 검증
  - 중단된 답변 버블에 평가 폼(`AnswerFeedback`)이 표시되지 않는지 검증
  - [질문 복원] 버튼 클릭 시 입력창에 직전 질문이 복원되고 포커스되는지 검증
  - [다시 질문] 버튼 클릭 시 동일 질문으로 새 스트리밍이 트리거되는지 검증

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스 대화창에서 질문을 입력하고 전송한 상태에서
  When 아직 LLM 토큰이 1개도 수신되지 않은 초기 검색/대기 단계에서 [중단] 버튼을 클릭했을 때
  Then 질문 메시지만 홀로 남지 않고, 해당 질문 바로 아래에 "답변 생성이 중단되었습니다." 텍스트와 `[생성 중단됨]` 배지가 포함된 AI 메시지 버블이 생성되어야 한다 (경계 케이스).
- Given 스트리밍 진행 중 답변 텍스트가 40자 수신된 상태에서
  When 사용자가 [중단] 버튼을 클릭했을 때
  Then 수신되었던 40자의 텍스트가 버려지지 않고 보존되며, 메시지 버블에 `[생성 중단됨]` 배지가 함께 렌더링되어야 한다.
- Given 답변 생성이 중단되어 중단된 AI 메시지 버블이 렌더링된 상태에서
  When 메시지 하단 액션 영역을 확인할 때
  Then `AnswerFeedback`(답변 평가 컴포넌트)은 렌더링되지 않아야 하고, [질문 복원] 및 [다시 질문] 버튼이 노출되어야 한다.
- Given 중단된 AI 메시지 버블 내에 [질문 복원] 버튼이 표시된 상태에서
  When 사용자가 [질문 복원] 버튼을 클릭했을 때
  Then 하단 질문 입력창(`question`)에 해당 질문의 텍스트가 자동으로 입력되고 입력창에 포커스가 이동해야 한다.
- Given 중단된 AI 메시지 버블 내에 [다시 질문] 버튼이 표시된 상태에서
  When 사용자가 [다시 질문] 버튼을 클릭했을 때
  Then 입력창을 직접 입력하지 않아도 동일한 질문으로 새로운 RAG 답변 스트리밍 요청이 즉시 시작되어야 한다.
- Given 현재 다른 질문의 스트리밍이 진행 중(`isStreaming === true`)인 경우
  When 기존에 중단되었던 메시지 버블의 [다시 질문] 버튼을 확인할 때
  Then 버튼이 비활성화(`disabled`)되어 동시 중복 요청이 방지되어야 한다.
- Given 정상적으로 생성이 완료된 일반 AI 답변 메시지인 경우
  When 메시지 버블이 렌더링될 때
  Then `[생성 중단됨]` 배지 및 [질문 복원]/[다시 질문] 버튼이 나타나지 않고, 기존의 복사, 보관, 피드백 평가 기능이 정상 노출되어야 한다.

## 참고
- 고쳐야 할 인터페이스 및 취소 핸들러: `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`), `public-front/src/components/AiService.tsx:626-645` (`handleCancelStreaming`)
- 고쳐야 할 메시지 버블 및 액션 렌더링: `public-front/src/components/AiService.tsx:1093-1165` (`AiService`), `public-front/src/components/AiService.tsx:1133-1160` (`AiService`)
- 질문 전송 핸들러 및 입력창: `public-front/src/components/AiService.tsx:647-665` (`handleSendQuestion`), `public-front/src/components/AiService.tsx:1285-1311` (`AiService`)
- 프론트엔드 취소 API 스트림 핸들: `public-front/src/api/ai.ts:428-457` (`askQuestionStream`)
- 백엔드 취소 게이트웨이 엔드포인트: `public-server/apps/gateway/src/ai/job/job.controller.ts:69-93` (`JobController.cancelJob`)
