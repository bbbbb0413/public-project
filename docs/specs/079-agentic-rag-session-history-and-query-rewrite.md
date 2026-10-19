---
id: SPEC-079
title: 에이전틱 RAG 경로의 대화 세션 이력 복원 및 후속 질문 재작성 연동
status: done
targets: [python-server]
stages: [backend, qa]
priority: high
---
## 배경 / 문제
사용자가 기존 대화 세션에서 후속 질문을 이어갈 때 프론트엔드 `public-front/src/components/AiService.tsx:610` (`handleSendQuestion`)은 이미 `sessionId`가 존재하면 서버가 세션 저장소에서 이전 대화 이력을 읽도록 `prevChatLog`를 빈 배열(`[]`)로 비워 전송하며, `public-front/src/api/ai.ts:476-484` (`askQuestionStream`) 역시 `conversationHistory`를 생략하고 `sessionId`만 요청 본문에 담아 보낸다.

단순 질문(`complexity != "complex"`) 경로는 이 계약을 정상적으로 처리하지만, 복합 질문(`complexity == "complex"`)을 처리하는 에이전틱 RAG 경로에는 세션 이력 복원과 후속 질문 재작성(`ConversationalQueryRewriter`) 단계가 모두 빠져 있어 2번째 턴부터 이전 대화 맥락이 통째로 유실된다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:149-158` (`AskRequestedConsumer._process`)의 단순 질문 경로는 `AskCommand`에 `session_id=session.get_session_id() if session else None`을 전달하고, `public-python-server/src/ai_service/rag/application/ask_use_case.py:76-88` (`AskUseCase.execute`)에서 `self._session_repo.find_by_id(command.session_id)`로 이전 대화 이력(`session.get_history()`)을 복원한 뒤 `ConversationalQueryRewriter.rewrite`로 대명사·생략 표현을 보완한다.
- 반면 에이전틱 RAG 경로인 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:130-141` (`AskRequestedConsumer._process`)은 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:104` (`AskRequestedConsumer._process`)에서 이미 세션을 조회했음에도 `AgenticAskCommand`에 `session_id`를 넘기지 않고 `conversation_history=message.conversation_history`(`sessionId`가 있을 때 항상 `None`)만 전달한다.
- 또한 `public-python-server/src/ai_service/rag/application/command/agentic_ask_command.py:11-21` (`AgenticAskCommand`)에는 `session_id` 필드가 없고, `public-python-server/src/ai_service/rag/rag_composition.py:156-164` (`build_rag_composition`)와 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:37-111` (`AgenticAskUseCase.__init__`, `AgenticAskUseCase.execute`)에도 `session_repo`를 통한 세션 이력 복원 및 `ConversationalQueryRewriter`를 통한 후속 질문 재작성 로직이 연결되어 있지 않다.

그 결과 사용자가 대화 세션의 2번째 질문부터 비교·분석 등 복합 후속 질문(예: "위에서 말한 정책과 절차를 비교해 줘")을 입력하면, 검색 쿼리 재작성이 수행되지 않아 엉뚱한 문서를 검색하고 LLM 프롬프트(`_build_messages`)에도 이전 대화 이력이 빈 상태로 전달된다. 이는 이번 분기 핵심 방향인 "답변을 믿을 수 있게 만든다"에서 다회차 에이전틱 RAG 답변의 근거 검색 정확도와 문맥 일관성을 무너뜨리는 결함이므로 즉시 바로잡아야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/command/agentic_ask_command.py`의 `AgenticAskCommand` 데이터클래스에 `session_id: str | None = None` 선택적 필드를 추가한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`의 `AskRequestedConsumer._process`에서 `AgenticAskCommand` 생성 시 `session_id=session.get_session_id() if session else None`을 전달한다.
- [ ] `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`의 `AgenticAskUseCase.__init__`에 `session_repo: ConversationSessionRepository | Any | None = None` 및 `query_rewriter: ConversationalQueryRewriter | None = None` 선택적 매개변수를 추가하고, `public-python-server/src/ai_service/rag/rag_composition.py`의 `build_rag_composition`에서 `session_repo`와 `query_rewriter`를 주입한다.
- [ ] `AgenticAskUseCase.execute`에서 `command.conversation_history`가 비어 있고 `command.session_id`와 `self._session_repo`가 존재할 때 `await self._session_repo.find_by_id(command.session_id)`를 호출하여 세션이 존재하면 `session.get_history()`로 `conversation_history`를 복원한다.
- [ ] `AgenticAskUseCase.execute`에서 복원된(또는 전달받은) `conversation_history`가 존재하고 `self._query_rewriter`의 `is_follow_up(command.question, conversation_history)`가 참(`True`)이면, `await self._query_rewriter.rewrite(command.question, conversation_history)`로 재작성된 질문을 초기 검색 쿼리(`current_query`)와 비평(`_critique_generator.generate`)·쿼리 정제(`_query_refiner.refine`)의 기준 질문으로 사용한다.
- [ ] `AgenticAskUseCase.execute`의 `_build_messages` 호출 시 복원된 `conversation_history`를 전달하여 모든 반복 회차의 LLM 프롬프트에 이전 대화 맥락이 포함되도록 한다.
- [ ] `session_id`에 해당하는 세션이 저장소에 없거나 이전 턴이 빈 배열(`[]`)인 경우, 또는 지시어·대명사가 없는 독립 질문인 경우에도 오류 없이 원본 `command.question`으로 정상 동작해야 한다.
- [ ] 에이전틱 RAG의 세션 이력 복원, 후속 질문 재작성, 컨슈머 `session_id` 전달을 검증하는 백엔드 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 프론트엔드(`public-front`) 및 NestJS 게이트웨이(`public-server`) 코드는 이미 `sessionId`를 전송하고 전달하도록 구현되어 있으므로 수정하지 않는다.
- `ConversationalQueryRewriter`의 지시어 목록(`FOLLOW_UP_INDICATORS`)이나 재작성 프롬프트, 또는 `QueryComplexityRouter`의 질문 복잡도 판별 기준 자체는 변경하지 않는다.
- 하이브리드 검색 알고리즘(`HybridSearchUseCase`)이나 리랭커 구성은 변경하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/command/agentic_ask_command.py`:
  - `AgenticAskCommand`(`agentic_ask_command.py:11-21`)에 `session_id: str | None = None` 필드를 추가한다.
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`:
  - `AgenticAskUseCase.__init__`(`agentic_ask_use_case.py:37-56`)에 `session_repo: ConversationSessionRepository | Any | None = None`과 `query_rewriter: ConversationalQueryRewriter | None = None`을 기본값 `None`으로 추가하여 기존 테스트와의 하위 호환성을 유지한다.
  - `AgenticAskUseCase.execute`(`agentic_ask_use_case.py:57-161`) 진입 시 `AskUseCase.execute`(`ask_use_case.py:76-88`)와 동일하게 `conversation_history`를 복원하고, 후속 질문인 경우 `search_question = await self._query_rewriter.rewrite(command.question, conversation_history)`로 재작성하여 `current_query = search_question`으로 초기화한다.
  - 루프 내 `self._build_messages`(`agentic_ask_use_case.py:109-111`)에 복원된 `conversation_history`를 전달하고, `self._critique_generator.generate`(`agentic_ask_use_case.py:143-145`)와 `self._query_refiner.refine`(`agentic_ask_use_case.py:159`)에는 문맥이 보완된 `search_question`을 전달한다.
- `public-python-server/src/ai_service/rag/rag_composition.py`:
  - `build_rag_composition`(`rag_composition.py:156-164`)에서 `AgenticAskUseCase` 생성 시 `session_repo=session_repo`, `query_rewriter=query_rewriter`를 주입한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `AskRequestedConsumer._process`(`ask_requested_consumer.py:130-141`)에서 `AgenticAskCommand` 생성 시 `session_id=session.get_session_id() if session else None`을 추가한다.
- `public-python-server/tests/unit/rag/test_agentic_ask_use_case.py` 및 `public-python-server/tests/unit/rag/test_ask_requested_consumer.py`:
  - 세션 이력 복원 후 후속 질문 재작성 및 LLM 메시지 이력 포함 여부, 컨슈머의 `session_id` 전달 여부를 검증하는 단위 테스트를 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 기존 대화 세션(`session-1`)에 이전 대화 턴(`[{"role": "user", "content": "프리미엄 요금제 안내해 줘"}, {"role": "assistant", "content": "월 29,000원입니다."}]`)이 저장되어 있고 `conversation_history`가 `None`인 상태에서, When 사용자가 후속 복합 질문 `"거기서 환불 조건과 해지 절차를 비교해 줘"`를 `session_id="session-1"`과 함께 요청하면, Then `AgenticAskUseCase.execute`가 세션 저장소에서 이전 대화 이력을 복원하고 `ConversationalQueryRewriter.rewrite`로 재작성된 질문으로 1회차 하이브리드 검색과 비평을 수행하며, LLM 호출 메시지(`_build_messages`)에 이전 대화 턴들이 순서대로 포함된다.
- Given 대화 이력이 복원되었으나 사용자의 복합 질문(`"기본 요금제와 엔터프라이즈 요금제의 차이를 비교해 줘"`)에 후속 질문 지시어가 포함되지 않았을 때(`is_follow_up == False`), When `AgenticAskUseCase.execute`가 실행되면, Then `ConversationalQueryRewriter.rewrite`를 호출하지 않고 원본 질문으로 하이브리드 검색을 수행하되 LLM 호출 메시지에는 복원된 대화 이력이 정상 포함된다.
- Given 전달된 `session_id`에 해당하는 세션이 저장소에 존재하지 않거나(`find_by_id`가 `None` 반환) 갓 생성된 첫 질문 세션이어서 이전 턴 이력이 빈 배열(`[]`)일 때(경계 케이스), When `AgenticAskUseCase.execute`가 실행되면, Then 예외 없이 `conversation_history`를 빈 상태로 처리하고 `ConversationalQueryRewriter.rewrite` 호출 없이 원본 질문으로 검색 및 답변 생성을 완료한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:37-161` (`AgenticAskUseCase.__init__`, `AgenticAskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/command/agentic_ask_command.py:11-21` (`AgenticAskCommand`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:130-141` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/rag_composition.py:156-164` (`build_rag_composition`)
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/rag/application/ask_use_case.py:76-88` (`AskUseCase.execute`), `public-python-server/src/ai_service/rag/application/conversational_query_rewriter_service.py:37-67` (`ConversationalQueryRewriter`), `public-front/src/components/AiService.tsx:610` (`handleSendQuestion`), `public-front/src/api/ai.ts:476-484` (`askQuestionStream`)
