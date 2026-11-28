---
id: SPEC-117
title: RAG 질의 시 대화 맥락 재작성 검색어(rewrittenQuery) 투명성 제공 및 세션 영속화 연동
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품은 올려 둔 문서를 근거로 질문에 답하는 도구이며, "답변을 그대로 믿으라고 요구하지 않고 사용자가 스스로 확인할 수 있어야 한다"는 대원칙을 지닌다. 이번 분기의 집중 과제는 "답변을 믿을 수 있게 만든다"이며, 지금까지 추론 과정 노출(SPEC-005), 답변 근거 미리보기(SPEC-006), 출처 및 신뢰도 복원(SPEC-011), 근거 미참조 경고(SPEC-059), 에이전틱 다회차 정제 쿼리 추적(SPEC-088), 근거 검증 배지(SPEC-108) 등이 단계적으로 배송되었다.
그러나 다회차 대화 세션에서 후속 질문을 처리할 때, 시스템이 어떤 맥락으로 검색 질의를 재구성하여 지식베이스를 조회했는지 사용자에게 전혀 공개되지 않는 중대한 투명성 결함이 존재한다:
첫째, `public-python-server/src/ai_service/rag/application/ask_use_case.py:84-90` (`AskUseCase.execute`)를 보면 단순 RAG 질의 시 대화 이력이 존재하고 후속 질문으로 판정되면 `self._query_rewriter.rewrite`를 호출해 문맥이 보강된 검색 질문(`search_question`)을 생성한다. 하지만 이 재작성 질의는 하이브리드 검색(`self._hybrid_search.execute`) 인자로만 쓰인 뒤 즉시 버려지며, 클라이언트로 방출되는 스트리밍 이벤트나 응답 메타데이터 어디에도 담기지 않는다.
둘째, `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:77-85` (`AgenticAskUseCase.execute`)를 보면 에이전틱 RAG 질의 역시 대화 맥락을 반영한 재작성 질의(`search_question`)를 생성하지만, 1회차 `searching` 진행 이벤트 발행 시 `next_query`에 이를 전달하지 않고 `current_query`로만 넘겨 은폐된다. SPEC-088에서 2회차 이후의 정제 질의(`refined_queries`)는 추적되도록 개선되었으나, 정작 1회차의 첫 대화 맥락 재작성 질의는 누락되어 있다.
셋째, `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:210-222` (`AskRequestedConsumer._process`) 및 `public-python-server/src/ai_service/rag/schemas.py:52-61` (`ConversationTurn.of_assistant`)를 보면 RAG 질의 완료 후 세션에 어시스턴트 턴을 영속화할 때 `rewritten_query` 매개변수가 전무하여 MongoDB 세션 저장소에 재작성 검색어가 기록되지 않는다. 또한 `publish_done` 완료 이벤트 데이터에도 `rewrittenQuery`가 누락되어 있다.
넷째, `public-front/src/components/AiService.tsx:56-67` (`ChatMessage`) 및 `public-front/src/components/AiService.tsx:1886-1915` (`AiService`)를 보면 프론트엔드 메시지 모델과 화면 렌더링부 어디에도 재작성 검색어에 대한 상태나 표시 영역이 없다.
이로 인해 사용자가 "그 문서의 3조 내용은 뭐야?", "앞서 언급한 수수료율은?"과 같은 후속 질문을 했을 때, AI가 이전 대화를 올바르게 이해하고 검색했는지 아니면 엉뚱한 키워드로 지식베이스를 검색하여 잘못된 근거를 가져왔는지 사용자가 스스로 검증할 방법이 없다.
따라서 대화 맥락으로 재작성된 검색 질문(`rewrittenQuery`)을 진행 이벤트 및 완료 메타데이터에 포함하고, 세션 턴에 영속화하며, 프론트엔드 AI 답변 버블 및 실시간 진행 상태에 명확히 시각화하여 답변의 검색 근거 투명성을 보장해야 한다.
## 요구사항
- [ ] `AskUseCase.execute`에서 후속 질문 재작성이 발생한 경우(`search_question != command.question`), 재작성된 검색어 메타데이터(예: `__REWRITTEN_QUERY:{query}`)를 스트림 초기에 방출한다.
- [ ] `AgenticAskUseCase.execute`에서 후속 질문 재작성이 발생한 경우, 1회차 `searching` 단계의 `_emit_progress` 호출에 `next_query=search_question`을 전달하거나 재작성 검색어 정보를 콜백으로 전달한다.
- [ ] `AskRequestedConsumer._process`에서 단순 및 에이전틱 RAG 질의 중 수집된 `last_rewritten_query`를 `publish_done` 완료 이벤트 페이로드(`rewrittenQuery`)에 포함하여 발행한다.
- [ ] Python 백엔드의 `TurnValue`, `TurnRecord`, `ConversationTurn`, `SessionTurnOut` 모델 및 `ConversationSession.append_turn` 메서드에 선택적 필드 `rewritten_query: str | None = None` (alias: `rewrittenQuery`)를 추가한다.
- [ ] `ConversationSessionRepository`(`repository.py`)의 `_to_record` 및 `_to_domain`에서 `rewrittenQuery` 필드를 MongoDB에 안전하게 저장하고 복원하며, 기존 필드가 없는 레거시 세션 데이터에 대해 `None`으로 안전하게 폴백한다.
- [ ] 프론트엔드 `public-front/src/api/ai.ts`의 `AskDoneMeta`, `SessionTurn`, `AgentProgress` 인터페이스에 선택적 속성 `rewrittenQuery?: string;`을 추가한다.
- [ ] 프론트엔드 `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `rewrittenQuery?: string;`을 추가하고, 스트리밍 완료(`onDone`) 및 세션 복원(`handleLoadSession`) 시 AI 메시지 객체에 매핑한다.
- [ ] `AiService.tsx`의 AI 메시지 버블 상단 또는 메타 영역에 `msg.rewrittenQuery`가 존재하고 원래 사용자 질문과 다를 때, 문맥 반영 검색어 배지(예: `[문맥 반영 검색: "{msg.rewrittenQuery}"]`)를 렌더링한다.
- [ ] 실시간 스트리밍 진행 중 인디케이터(`agent-progress-indicator`)에서 1회차 검색 시 재작성 검색어가 있을 경우 이를 시각적으로 표시한다.
- [ ] 백엔드 재작성 쿼리 방출/영속화 및 프론트엔드 문맥 반영 검색어 배지 렌더링을 검증하는 단위 테스트를 추가한다.
## 비요구사항 (Out of scope)
- `ConversationalQueryRewriterService`의 LLM 프롬프트 템플릿이나 질문 재작성 알고리즘 자체를 변경하지 않는다.
- 대화 세션 목록 검색(`GET /rag/sessions`)의 검색 대상 필드에 `rewrittenQuery`를 추가하거나 인덱스를 변경하지 않는다.
- 답변 평가(`AnswerFeedback.tsx`) 폼 내부의 컴포넌트 구조나 피드백 API 스키마를 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제(`Payment.tsx`) 컴포넌트를 수정하지 않는다.
## 백엔드
`public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `AskUseCase.execute`:
  - `search_question`이 원래 `command.question`과 다를 경우 `yield f"__REWRITTEN_QUERY:{search_question}"` 메타 접두사 스트림을 방출한다.

`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
- `AgenticAskUseCase.execute`:
  - 후속 질문 재작성이 적용된 경우(`search_question != command.question`), 1회차 `searching` 진행 이벤트의 `next_query_display`에 `search_question`을 할당하여 `_emit_progress`로 방출되도록 한다.

`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- `AskRequestedConsumer._process`:
  - `_REWRITTEN_QUERY_PREFIX = "__REWRITTEN_QUERY:"` 감지 로직을 추가하여 `last_rewritten_query`를 수집한다.
  - 에이전틱 질의의 경우 `on_progress` 데이터에서 1회차 `nextQuery` 또는 `rewrittenQuery`를 `last_rewritten_query`로 수집한다.
  - `session.append_turn` 호출 시 `rewritten_query=last_rewritten_query`를 전달하여 세션에 영속화한다.
  - `publish_done` 호출 시 `done_data`에 `rewrittenQuery=last_rewritten_query`를 포함한다.

`public-python-server/src/ai_service/rag/schemas.py`
- `TurnValue`, `TurnRecord`에 `rewritten_query: str | None = None` 속성을 추가한다.
- `ConversationTurn.of_assistant` 및 `ConversationTurn.restore`, `ConversationSession.append_turn`에 `rewritten_query` 매개변수를 지원한다.
- `SessionTurnOut`에 `rewritten_query: str | None = Field(default=None, alias="rewrittenQuery")` 필드를 추가한다.

`public-python-server/src/ai_service/rag/repository.py`
- `ConversationSessionRepository._to_record`에서 `t.rewritten_query`가 존재할 때 `turn_dict["rewrittenQuery"] = t.rewritten_query`를 저장한다.
- `ConversationSessionRepository._to_domain`에서 `t.get("rewrittenQuery")`로 복원하여 하위 호환성을 보장한다.
## 프론트엔드
`public-front/src/api/ai.ts`
- `AskDoneMeta`, `SessionTurn`, `AgentProgress`에 `rewrittenQuery?: string;` 필드를 선언한다.
- `askQuestionStream`의 `done` 이벤트 수신부에서 `data.rewrittenQuery`를 추출하여 `onDone` 콜백 메타데이터로 전달한다.

`public-front/src/components/AiService.tsx`
- `ChatMessage` 인터페이스에 `rewrittenQuery?: string;` 필드를 추가한다.
- `handleSendQuestion`의 `onDone` 실행 시 전달받은 `finalMeta.rewrittenQuery`를 신규 AI 메시지에 저장한다.
- `handleLoadSession` 실행 시 `detail.turns`의 각 턴에서 `t.rewrittenQuery`를 `ChatMessage` 객체에 매핑하여 복원한다.
- AI 메시지 버블 상단(메타 배지 영역)에 `msg.rewrittenQuery`가 존재하고 직전 사용자 질문과 일치하지 않는 경우:
  - `<div className="rewritten-query-badge" data-testid="rewritten-query-badge">문맥 반영 검색: &quot;{msg.rewrittenQuery}&quot;</div>` 배지를 렌더링한다.
- 스트리밍 진행 중 인디케이터(`agent-progress-indicator`)에서 `lastProgress?.nextQuery`가 존재할 때 정제/재작성 검색어를 명확히 표시한다.

`public-front/src/components/AiService.css`
- `.rewritten-query-badge`: 은은한 청회색/보라 테마 배경, 테두리, 둥근 모서리 및 작은 폰트 크기로 검색 맥락을 깔끔하게 표시하는 스타일 추가.
## 수용 기준 (Acceptance Criteria)
- Given 이전 대화 이력이 존재하는 세션에서 사용자가 "그 법률의 적용 대상은 누구인가요?"라는 후속 질문을 전송했을 때
  When 백엔드가 대화 맥락을 반영하여 검색 질문을 "산업안전보건법 적용 대상"으로 재작성하면
  Then 스트리밍 완료 이벤트(`done`) 및 세션 상세 조회(`turns`) 응답에 `rewrittenQuery: "산업안전보건법 적용 대상"`이 포함된다.
- Given `rewrittenQuery`가 포함된 AI 답변이 화면에 표시될 때
  When 메시지 버블 상단을 확인하면
  Then `[문맥 반영 검색: "산업안전보건법 적용 대상"]` 배지가 표시되어 사용자가 검색된 맥락을 직접 확인할 수 있다.
- Given 이전 대화가 없거나 후속 질문이 아니어서 질문 재작성이 일어나지 않은 첫 단발성 질의인 경우 (경계 케이스)
  When 답변이 생성 완료되면
  Then `rewrittenQuery`는 `None`/`undefined`로 처리되고 화면에 문맥 반영 검색 배지가 노출되지 않는다.
- Given `rewrittenQuery` 필드가 저장되지 않은 과거 세션 데이터(레거시 세션)를 불러왔을 때 (경계 케이스)
  When 세션 상세 조회를 실행하면
  Then 에러 없이 세션이 정상 로드되고 `rewrittenQuery`가 없는 일반 메시지로 정상 렌더링된다.
## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:84-90` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:77-85` (`AgenticAskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:210-222` (`AskRequestedConsumer._process`)
- 관련 정의: `public-python-server/src/ai_service/rag/schemas.py:52-61` (`ConversationTurn.of_assistant`)
- 관련 정의: `public-python-server/src/ai_service/rag/repository.py:109-119` (`ConversationSessionRepository._to_domain`)
- 관련 정의: `public-front/src/components/AiService.tsx:56-67` (`ChatMessage`)
- 관련 정의: `public-front/src/api/ai.ts:466-471` (`AskDoneMeta`)
