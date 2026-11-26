---
id: SPEC-116
title: 에이전틱 RAG 예산 소진 시 비평 생략 안내 및 검증 완료 배지 오표시 방지와 상태 영속화
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
에이전틱 RAG(Agentic RAG) 질의응답 흐름은 복잡한 질문에 대해 검색-생성-비평-정제 루프를 반복 수행하며 답변 품질을 점진적으로 검증하고 향상한다. 그러나 현재 시스템은 토큰 수나 경과 시간 예산을 초과하여 루프가 중단될 때, 비평 검증이 누락된 중간 산출물이 마치 정상 검증된 답변처럼 왜곡되어 방출되는 심각한 결함이 존재한다.
`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:183-201` (`AgenticAskUseCase.execute`)에서는 생성 중 또는 생성 직후 예산이 소진(`budget_exhausted` 또는 `command.budget.is_exhausted`)되면 비평 단계(`_critique_generator.generate`)를 건너뛰고 즉시 `return`한다. 이 과정에서 1회차일 때는 단순히 검색된 청크의 최고 유사도 점수를 신뢰도로 삼아 `last_confidence = max(c.score for c in chunks)` 및 `last_missing = []`로 설정하고, 2회차 이상일 때는 직전 회차의 비평 신뢰도를 그대로 유지한 채 2회차 생성 답변에 대한 비평은 아예 수행하지 않는다.
이로 인해 `public-front/src/components/AiService.tsx:1844-1852` (`AiService`)의 검증 완료 배지 판정 로직(`isGroundedVerified`)에서 치명적인 오표시가 발생한다. 1회차 예산 소진 시 청크 유사도 점수가 0.8 이상이면, 실제 LLM 생성 답변에 대한 근거 비평을 거치지 않은 불완전 산출물임에도 불구하고 `✓ 지식베이스 근거 검증 완료` 배지가 화면에 그대로 노출된다. 이는 SPEC-108 및 SPEC-112에서 구축한 답변 근거 검증의 신뢰성을 정면으로 왜곡한다.
또한 `public-python-server/src/ai_service/rag/schemas.py:31-41` (`TurnValue`) 및 `public-python-server/src/ai_service/rag/schemas.py:391-403` (`SessionTurnOut`)에 예산 소진 여부(`is_budget_exhausted`) 필드가 없고, `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:224-236` (`AskRequestedConsumer._process`)와 `public-front/src/api/ai.ts:466-471` (`AskDoneMeta`)에도 해당 메타데이터가 전달되지 않아, 프론트엔드와 사용자는 해당 답변이 예산 초과로 비평이 중단된 미완성 답변인지 전혀 알 수 없다.

## 요구사항
- [ ] `AgenticAskUseCase.execute`에서 시간 또는 토큰 예산 소진(`budget_exhausted` 또는 `command.budget.is_exhausted`)으로 루프가 조기 종료될 때, 비평이 생략되었음을 나타내는 예산 소진 메타데이터(`is_budget_exhausted: True`)를 방출 및 전달한다.
- [ ] 예산 소진으로 비평을 거치지 못한 에이전틱 답변의 경우, 검색 유사도를 근거 신뢰도로 둔갑시키지 않고 안전한 미검증 기본 신뢰도(`confidence = 0.0`)로 처리한다.
- [ ] Python 백엔드의 세션 도메인 모델(`TurnValue`, `ConversationTurn`, `TurnRecord`), DTO(`SessionTurnOut`), 세션 저장소(`ConversationSessionRepository`)에 `is_budget_exhausted: bool = False` (alias: `isBudgetExhausted`) 필드를 추가하여 세션 턴에 영속화하고 세션 상세 조회 시 반환한다.
- [ ] `AskRequestedConsumer._process`에서 에이전틱 RAG 완료 이벤트(`publish_done`)에 `isBudgetExhausted: bool` 필드를 포함하여 발행한다.
- [ ] 프론트엔드 API 규격(`AskDoneMeta`, `SessionTurn`) 및 컴포넌트 모델(`ChatMessage`)에 `isBudgetExhausted?: boolean` 선택적 속성을 추가한다.
- [ ] 프론트엔드 `AiService.tsx`의 `isGroundedVerified` 판정 조건에 `!msg.isBudgetExhausted` 가드를 추가하여, 예산 소진으로 비평이 누락된 답변에 `[✓ 지식베이스 근거 검증 완료]` 배지가 렌더링되지 않도록 보장한다.
- [ ] `msg.isBudgetExhausted === true`인 AI 메시지 버블에 `has-budget-exhausted` CSS 클래스를 적용하고, 상단에 `[답변 검증 미완료 (예산 소진)]` 주의 배지(`budget-exhausted-badge`) 및 인라인 안내 배너(`budget-exhausted-warning`)를 렌더링한다.
- [ ] 인라인 안내 배너 내에 "답변 검증 예산(시간/토큰)이 소진되어 근거 일치성 검증이 완료되지 못했습니다." 안내 문구와 함께 질문을 구체화하여 다시 질문할 수 있는 [질문 구체화] 액션 버튼(`btn-refine-budget-question`)을 제공한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 대화의 `isBudgetExhausted` 상태를 안전하게 복원하여 배지 및 안내 배너를 동일하게 렌더링한다.

## 비요구사항 (Out of scope)
- 에이전틱 RAG의 기본 예산(최대 반복 횟수, 제한 시간, 최대 허용 토큰 수) 파라미터 값 자체를 변경하거나 사용자 설정 UI를 추가하는 작업은 다루지 않는다.
- 비평 생성기(`CritiqueGeneratorService`)의 프롬프트나 LLM 모델을 교체하는 작업은 포함하지 않는다.
- 단발성 비에이전틱 질의(`AskUseCase`)의 질의 파이프라인이나 쿼리 라우팅 알고리즘은 변경하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`:
  - `AgenticAskUseCase.execute`에서 `budget_exhausted` 또는 `command.budget.is_exhausted` 감지 시, `is_budget_exhausted=True` 플래그를 설정하고 `_emit_progress` 및 `__CONFIDENCE` 스트림 메타데이터에 `{"confidence": 0.0, "missing": [], "isBudgetExhausted": True}` 형태로 방출한다.
  - 1회차 및 다회차 모두 예산 소진 시 비평을 수행하지 못했으므로 임의의 높은 점수를 부여하지 않고 `confidence = 0.0`으로 정규화한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `_process`의 `on_progress` 및 스트림 청크 수신부에서 `isBudgetExhausted` 플래그를 수집한다.
  - 세션 영속화 시 `session.append_turn`에 `is_budget_exhausted` 값을 전달한다.
  - `publish_done` 완료 이벤트 데이터에 `isBudgetExhausted: bool` 필드를 포함한다.
- `public-python-server/src/ai_service/rag/schemas.py`:
  - `TurnValue`, `TurnRecord` 데이터클래스에 `is_budget_exhausted: bool = False`를 추가한다.
  - `ConversationTurn.of_assistant` 및 `ConversationTurn.restore`, `ConversationSession.append_turn`에 `is_budget_exhausted: bool = False` 매개변수를 지원한다.
  - `SessionTurnOut` 모델에 `is_budget_exhausted: bool = Field(default=False, alias="isBudgetExhausted")`를 추가하고 `from_domain` 매핑을 지원한다.
- `public-python-server/src/ai_service/rag/repository.py`:
  - `ConversationSessionRepository._to_record` 및 `_to_domain`에서 `isBudgetExhausted` 필드를 MongoDB에 저장하고 역직렬화한다. 기존 문서에 필드가 없는 경우 `False`로 안전하게 폴백한다.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `AskDoneMeta` 인터페이스에 `isBudgetExhausted?: boolean` 필드를 추가하고 스트림 완료 시 전달한다.
  - `SessionTurn` 인터페이스에 `isBudgetExhausted?: boolean` 필드를 추가한다.
- `public-front/src/components/AiService.tsx`:
  - `ChatMessage` 인터페이스에 `isBudgetExhausted?: boolean` 속성을 추가한다.
  - `handleSendQuestion`의 `onDone` 콜백 수신 시 `meta.isBudgetExhausted`를 새 AI 메시지 객체에 반영한다.
  - `handleLoadSession`에서 세션 복원 시 각 턴의 `t.isBudgetExhausted`를 `chatLog`의 `isBudgetExhausted`로 매핑한다.
  - `isGroundedVerified` 조건에 `!msg.isBudgetExhausted` 가드를 추가한다.
  - `msg.isBudgetExhausted === true`인 경우 메시지 버블에 `has-budget-exhausted` 클래스, 상단 메타 영역에 `budget-exhausted-badge` 배지(`data-testid="budget-exhausted-badge"`), 메시지 상단에 `budget-exhausted-warning` 배너(`data-testid="budget-exhausted-warning"`)를 렌더링한다.
  - `budget-exhausted-warning` 배너에 [질문 구체화] 버튼(`btn-refine-budget-question`, `data-testid="btn-refine-budget-question"`)을 배치하여, 클릭 시 직전 사용자 질문을 입력창에 복원하고 포커스한다.
- `public-front/src/components/AiService.css`:
  - `has-budget-exhausted`, `budget-exhausted-badge`, `budget-exhausted-warning`, `btn-refine-budget-question`에 대한 스타일 규칙을 정의한다.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 질의 실행 중 1회차 또는 다회차에서 시간/토큰 예산이 소진되었을 때
  When 스트리밍이 종료되고 완료(`done`) 이벤트가 발행되면
  Then `done` 이벤트 페이로드에 `isBudgetExhausted: true`가 포함되고, `confidence`는 0.0으로 전달된다.
- Given 예산이 소진되어 `isBudgetExhausted: true`인 AI 메시지가 화면에 렌더링될 때
  When 메시지의 검색 청크 점수가 0.8 이상이거나 이전 회차 신뢰도가 높더라도
  Then `[✓ 지식베이스 근거 검증 완료]`(`grounded-verified-badge`) 배지는 절대 렌더링되지 않고, `[답변 검증 미완료 (예산 소진)]`(`budget-exhausted-badge`) 배지와 `budget-exhausted-warning` 안내 배너가 렌더링된다.
- Given `budget-exhausted-warning` 배너가 표시된 AI 답변이 있을 때
  When 사용자가 배너 내 [질문 구체화](`btn-refine-budget-question`) 버튼을 클릭하면
  Then 해당 답변의 직전 사용자 질문 내용이 하단 질문 입력창에 복원되고 입력창에 포커스가 이동한다.
- Given 세션에 `isBudgetExhausted: true` 속성을 가진 답변 턴이 영속화되어 있을 때
  When 사용자가 사이드바에서 해당 대화 세션을 다시 불러오면(세션 복원)
  Then 해당 답변 버블에 `budget-exhausted-badge` 및 `budget-exhausted-warning` 배너가 동일하게 유지되어 렌더링된다.
- Given [경계 케이스] `isBudgetExhausted` 필드가 전혀 없는 과거 레거시 세션 데이터나 정상 완료된 단발성 질의 턴을 조회할 때
  When 세션 상세 조회 API를 호출하거나 화면에 렌더링하면
  Then `isBudgetExhausted`는 기본값 `false`로 안전하게 취급되어 어떠한 파싱 에러나 불필요한 경고 배너도 노출되지 않는다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:183-201` (`AgenticAskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:224-236` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1844-1852` (`AiService`)
- 관련 도메인 모델 정의: `public-python-server/src/ai_service/rag/schemas.py:31-41` (`TurnValue`)
- 관련 API 클라이언트 인터페이스: `public-front/src/api/ai.ts:466-471` (`AskDoneMeta`)
