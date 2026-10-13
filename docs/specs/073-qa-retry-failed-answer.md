---
id: SPEC-073
title: AI 답변 생성 실패 시 인라인 에러 표시 및 질문 재시도 지원
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 프론트엔드 RAG 대화 화면(`public-front/src/components/AiService.tsx`)은 질문 스트리밍 중 오류 발생 시 실패 상태를 대화창 내에 보존하지 못하고 사용자의 질문 내용까지 유실시키는 결함이 있다.

구체적인 문제 지점은 다음과 같다:
- `public-front/src/components/AiService.tsx:647-654` (`handleSendQuestion`)에서 사용자가 질문을 입력하고 전송하면 `setQuestion('')`으로 입력창이 즉시 비워지고 `chatLog`에 사용자 메시지만 추가된다.
- `public-front/src/components/AiService.tsx:683-690` (`handleSendQuestion`)에서 백엔드 네트워크 순단, LLM 타임아웃, 예산 초과 등으로 인해 `askQuestionStream`의 `onError` 콜백이 호출되면 상단 공통 배너(`setErrorMsg`)에만 에러를 노출하고 스트리밍 상태를 즉시 종료한다. 이때 `public-front/src/api/ai.ts:520-536` (`askQuestionStream`)가 에러 이벤트 또는 예외로부터 구체적인 오류 정보(`Error`)를 전달함에도 불구하고, 상단 배너에 고정 문자열 하나만 띄운 채 실패한 AI 메시지 버블을 `chatLog`에 일절 추가하지 않는다.
- 이로 인해 `public-front/src/components/AiService.tsx:1093-1166` (`AiService`)에서 렌더링되는 채팅 영역에는 질문 말풍선만 남고 AI 응답이 아예 존재하지 않는 비정상적인 상태가 된다. 상단 배너를 보지 못한 사용자는 AI가 답변을 멈추었거나 질문이 씹힌 것으로 오인하게 된다.
- 더욱이 사용자가 정성껏 작성했던 질문 텍스트가 입력창에서 이미 지워졌기 때문에, 사용자는 실패한 질문을 복사해 두지 않았다면 처음부터 다시 작성해야 하는 심각한 불편을 겪는다.
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 인터페이스에는 에러 상태를 구분할 수 있는 필드가 정의되어 있지 않으며, 메시지 버블 내에 즉시 다시 전송할 수 있는 [다시 시도] 버튼이나 질문 내용을 입력창으로 되돌리는 복원 기능이 전혀 존재하지 않는다.
- 이 개선은 "답변을 믿을 수 있게 만든다"는 이번 분기 제품 방향에서 시스템 응답 오류를 투명하게 표시하고 실패한 질문을 유실 없이 복구하여 사용자와 시스템 간의 신뢰를 유지하는 데 직결된다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `isError?: boolean` 및 `errorMessage?: string` 선택적 필드를 추가한다.
- [ ] `handleSendQuestion`의 `onError` 콜백에서 에러 발생 시 스트리밍 중단 상태로 끝나지 않고, 실패한 해당 질문 바로 뒤에 에러 상태를 나타내는 AI 메시지(`{ sender: 'ai', text: accumulated || '답변을 생성하지 못했습니다.', isError: true, errorMessage: errorDetails }`)를 `chatLog`에 추가한다.
- [ ] 스트리밍 진행 중 일부 텍스트(`accumulated`)가 수신되다가 오류가 발생한 경우, 이미 수신된 텍스트를 버리지 않고 에러 메시지 버블 내에 보존하여 사용자가 중간 생성 내용을 확인할 수 있도록 한다.
- [ ] 에러 상태인 AI 메시지 버블(`isError === true`) 내에 직관적인 [다시 시도] 버튼을 렌더링한다.
- [ ] 사용자가 [다시 시도] 버튼을 클릭하면, 직전 실패했던 동일한 질문 내용으로 `handleSendQuestion` 흐름을 재실행하여 답변 생성을 다시 요청한다.
- [ ] 에러 상태인 AI 메시지 버블 내에 [질문 복원] 버튼을 함께 제공하여, 클릭 시 실패한 질문 텍스트를 하단 입력창(`question`)에 다시 채우고 포커스하여 사용자가 질문을 수정 후 재전송할 수 있도록 한다.
- [ ] 재시도 중이거나 스트리밍 진행 중에는 [다시 시도] 버튼이 비활성화(`disabled`)되어 중복 요청을 방지한다.
- [ ] 정상적으로 생성 완료된 AI 메시지(`isError !== true`)에는 재시도/복원 버튼이 노출되지 않는다.

## 비요구사항 (Out of scope)
- 백엔드(NestJS Gateway, Python AI Service)의 RAG 에러 반환 스키마 변경이나 재시도 큐잉 추가
- 사용자가 명시적으로 중단 버튼을 누른 정상 취소 흐름(`handleCancelStreaming`)의 동작 변경
- 세션 영속화 DB(MongoDB)에 에러 메시지 버블을 영구 저장하는 스키마 마이그레이션 (클라이언트 세션 뷰 레벨 복구에 집중)

## 프론트엔드
- 대상 컴포넌트: `public-front/src/components/AiService.tsx`
- 대상 스타일: `public-front/src/components/AiService.css`
- 주요 변경 사항:
  - `ChatMessage` 인터페이스에 `isError?: boolean`, `errorMessage?: string` 속성 추가
  - `handleSendQuestion` 내 `onError` 콜백 개선:
    - 수신된 에러 객체로부터 메시지를 추출하여 에러 AI 메시지 객체 생성 후 `chatLog`에 추가
    - 마지막 실패 질문(`lastFailedQuestionRef` 또는 턴 기반 참조)을 기록하여 재시도 컨텍스트 확보
  - 재시도 핸들러(`handleRetryQuestion(questionText: string, errorMsgIndex: number)`) 추가:
    - 실패했던 에러 버블을 정리하거나 갱신하고 동일 질문으로 스트리밍 재시작
  - 질문 복원 핸들러(`handleRestoreQuestion(questionText: string)`) 추가:
    - 하단 입력창(`question`) 상태에 실패 질문 복원 및 입력창 포커스
  - AI 메시지 버블 렌더링 분기:
    - `msg.isError`일 때 `.chat-message.ai.error` 클래스 적용
    - 에러 안내 문구(또는 수신된 중간 답변 + 에러 배너) 렌더링
    - [다시 시도], [질문 복원] 액션 버튼 렌더링
  - 스타일 추가(`AiService.css`):
    - `.chat-message.ai.error .message-bubble`: 에러 시각 피드백 테두리 및 배경색 (부드러운 적색 계열)
    - `.btn-retry-answer`, `.btn-restore-question`: 재시도 및 복원 액션 버튼 스타일

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스 대화창에서 질문을 입력하고 전송한 상태에서
  When 백엔드 오류 또는 네트워크 단절로 인해 스트리밍 에러(`onError`)가 발생했을 때
  Then 화면 상단 에러 배너와 함께 채팅 대화창 내 해당 질문 아래에 에러 상태 AI 메시지 버블이 표시되고, 내부에 [다시 시도] 버튼과 [질문 복원] 버튼이 렌더링되어야 한다.
- Given 스트리밍 중 답변 텍스트가 50자 수신된 후 중간에 네트워크가 끊어져 에러가 발생한 경우
  When 에러가 처리되어 화면에 표시될 때
  Then 기수신된 50자의 텍스트가 삭제되지 않고 보존된 상태로 에러 표시와 함께 렌더링되어야 한다.
- Given 답변 생성 실패로 에러 AI 메시지 버블과 [다시 시도] 버튼이 표시된 상태에서
  When 사용자가 [다시 시도] 버튼을 클릭했을 때
  Then 입력창에 질문을 다시 입력하지 않아도 직전 실패한 질문으로 신규 스트리밍 요청이 즉시 시작되어야 한다.
- Given 답변 생성 실패로 에러 AI 메시지 버블과 [질문 복원] 버튼이 표시된 상태에서
  When 사용자가 [질문 복원] 버튼을 클릭했을 때
  Then 하단 입력창에 실패했던 질문 내용이 자동으로 채워지고 입력창에 포커스가 이동해야 한다.
- Given 빈 문자열이거나 공백만 있는 질문인 경우
  When 사용자가 전송을 시도할 때
  Then 전송 및 에러 메시지 생성이 차단되고 기존 동작과 동일하게 무시되어야 한다 (경계 케이스).
- Given 정상적으로 답변 생성이 완료된 AI 메시지 버블인 경우
  When 렌더링될 때
  Then 에러 스타일 및 [다시 시도]/[질문 복원] 버튼이 표시되지 않고 기존의 복사/보관/피드백 액션만 정상 렌더링되어야 한다.

## 참고
- 고쳐야 할 인터페이스 및 핸들러: `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`), `public-front/src/components/AiService.tsx:647-657` (`handleSendQuestion`), `public-front/src/components/AiService.tsx:683-690` (`handleSendQuestion`)
- 고쳐야 할 메시지 버블 렌더링: `public-front/src/components/AiService.tsx:1093-1166` (`AiService`)
- 스트리밍 에러 전달 규약: `public-front/src/api/ai.ts:520-536` (`askQuestionStream`)
