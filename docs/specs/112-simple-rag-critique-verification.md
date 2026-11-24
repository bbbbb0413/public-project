---
id: SPEC-112
title: 단순 RAG 질의의 답변 근거 비평(Critique) 연동 및 검증 완료 배지 오표시 방지
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
이 제품의 핵심 목표는 올려 둔 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 최근 배송된 SPEC-108에서는 신뢰도가 80% 이상이고 출처가 확보된 답변에 대해 화면 상단에 `[✓ 지식베이스 근거 검증 완료]`(`isGroundedVerified`) 배지를 부여하여 검증된 신뢰 답변임을 시각적으로 보증하도록 개선하였다.
그러나 현재 단순 질문 RAG 처리 흐름에는 다음과 같이 검증 체계를 근본적으로 무력화하고 환각 답변을 검증된 답변으로 오인하게 만드는 치명적인 결함이 존재한다:
첫째, `public-python-server/src/ai_service/rag/application/ask_use_case.py:92-113` (`AskUseCase.execute`)에서는 하이브리드 검색 청크(`chunks`)가 존재할 때, 실제 생성된 LLM 답변에 대한 비평(Critique)이나 근거 일치성 평가를 전혀 거치지 않고 단순히 검색된 청크의 최고 유사도 점수(`max(c.score for c in chunks)`)를 답변의 `confidence`로 둔갑시킨다. 또한 결여 정보(`missing`)를 무조건 빈 배열(`[]`)로 고정하여 클라이언트에 스트리밍한다.
둘째, `public-python-server/src/ai_service/rag/rag_composition.py:149-174` (`build_rag_composition`)에서 `CritiqueGeneratorService` 인스턴스를 생성하지만, 이는 오직 `AgenticAskUseCase`에만 주입될 뿐 `AskUseCase`에는 전달되지 않는다. 이로 인해 에이전틱 경로가 아닌 단순 질문 경로에서는 생성 답변이 문서 근거에 부합하는지 판정하는 서비스가 연결조차 되어 있지 않다.
셋째, `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:194-209` (`AskRequestedConsumer._process`)에서는 단순 질문 스트림 시작 시 전달된 `_CONFIDENCE_PREFIX` 메타데이터를 그대로 `last_confidence`와 `last_missing`에 저장하고, 224-235행에서 질의 완료(`publish_done`) 및 세션 영속화(`session.append_turn`)에 그대로 전달한다. 즉, LLM이 실제 답변 토큰을 생성하기도 전에 검색 점수로 확정된 신뢰도가 답변 전체의 최종 신뢰도로 굳어진다.
넷째, `public-front/src/components/AiService.tsx:1816-1824` (`AiService`)에서는 백엔드에서 전달된 `confidence >= 0.8`과 결여 항목 부재(`!hasMissing`) 조건만 만족하면 `isGroundedVerified`를 참으로 판정한다. 이로 인해 단순 질문에서 검색 청크 점수가 0.8 이상이기만 하면, LLM이 문서를 무시하고 환각 답변을 하거나 "문서에서 정보를 찾을 수 없습니다"라고 답하더라도 화면에는 당당하게 `[✓ 지식베이스 근거 검증 완료]` 배지가 노출된다.
결과적으로 검색 점수와 답변 신뢰도가 혼동되어 사용자에게 잘못된 안전 확신(False Sense of Verification)을 주는 상태이며, 이는 SPEC-108의 검증 완료 배지 도입 성과를 지우는 결함이다. 따라서 단순 RAG 질의에서도 실제 생성된 답변을 바탕으로 비평(Critique)을 수행하여 진짜 근거 일치성 신뢰도와 결여 항목을 산출하고, 이를 바탕으로 검증 배지 및 경고가 올바르게 작동하도록 개선해야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/rag_composition.py`의 `build_rag_composition`에서 생성된 `critique_generator`(`CritiqueGeneratorService`)를 `AskUseCase`에도 주입할 수 있도록 생성자 매개변수 및 DI 구성을 확장한다.
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`의 `AskUseCase.execute`에서 검색 즉시 `max(c.score)`를 신뢰도로 둔갑시켜 방출하던 조기 `__CONFIDENCE` 발행을 제거하고, 검색된 출처(`__SOURCES`)만 선제적으로 방출한다.
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`의 `AskUseCase.execute`에서 LLM 토큰 스트리밍이 완료된 후, 검색된 청크(`chunks`)와 최종 생성 답변(`answer`)에 대해 `self._critique_generator.generate(command.question, answer, chunks, tenant)`를 호출하여 실제 답변 근거 일치성 비평을 수행한다.
- [ ] 비평 수행 결과 산출된 실제 신뢰도(`critique.get_confidence()`)와 결여 항목(`critique.get_missing()`)을 `__CONFIDENCE` 스트림 메타데이터로 방출하여 `AskRequestedConsumer`가 이를 수집하도록 한다.
- [ ] 비평 생성 실패(예외 발생 또는 타임아웃) 시 안전한 기본값(`confidence = 0.0`, `missing = []`)으로 폴백하여 검증되지 않은 답변에 높은 신뢰도가 부여되는 것을 원천 차단한다.
- [ ] 검색된 청크가 0건인 경우(`not chunks`)에는 비평 호출을 건너뛰고 기존과 동일하게 `confidence = 0.0`, `missing = []`를 안전하게 반환한다.
- [ ] `public-front/src/components/AiService.tsx`에서 단순 RAG 질의 완료 시 비평을 통해 산출된 실제 신뢰도와 결여 항목이 AI 메시지 버블에 반영된다.
- [ ] 비평 결과 신뢰도가 0.8 이상이고 `missing`이 빈 배열인 경우에만 `isGroundedVerified`가 활성화되어 `[✓ 지식베이스 근거 검증 완료]` 배지가 표시된다.
- [ ] 단순 질의라도 비평 결과 신뢰도가 60% 미만으로 떨어진 경우(`confidence < 0.6`) `[답변 신뢰도 낮음]` 경고 배너 및 `[질문 구체화]` 액션이 정상 연동되고 검증 배지는 노출되지 않는다.
- [ ] 단순 질의 비평 결과 결여 항목(`missing`)이 존재하는 경우 `[추가 확인 필요]` 배지와 항목별 `[질문하기]` 버튼이 정상 노출된다.
- [ ] 단순 RAG 비평 연동 및 신뢰도/결여 정보 산출과 프론트엔드 검증 배지 노출 정상화를 검증하는 단위 테스트를 추가 및 갱신한다.

## 비요구사항 (Out of scope)
- 복합 에이전틱 질의(`AgenticAskUseCase`)의 다회차 반복 탐색 루프(`iterations`, `refining`)나 버짓 관리 로직을 수정하지 않는다.
- 하이브리드 검색기(`hybrid_search_use_case.py`)의 검색 알고리즘이나 임베딩 모델 가중치를 변경하지 않는다.
- 사용자 답변 평가(`AnswerFeedback.tsx`) 폼의 별점 입력 방식이나 불만족 사유 태그 목록을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역(`PaymentHistory.tsx`) 컴포넌트를 수정하지 않는다.

## 백엔드
`public-python-server/src/ai_service/rag/rag_composition.py`
- `build_rag_composition`에서 `critique_generator = CritiqueGeneratorService(llm_gateway_service, settings)`를 `AskUseCase` 인스턴스 생성 시 전달한다.

`public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `AskUseCase.__init__`에 `critique_generator: CritiqueGeneratorService | None = None` 선택적 매개변수를 추가한다.
- `execute` 메서드에서:
  - 검색 직후 `chunks`가 있을 때 `max(c.score)` 기반으로 `__CONFIDENCE`를 발행하던 로직을 제거한다.
  - 토큰 스트리밍 루프(`async for token in ...`) 종료 후 모인 전체 답변(`answer = "".join(collected)`)과 `chunks`를 바탕으로 `critique = await self._critique_generator.generate(command.question, answer, chunks, tenant)`를 실행한다.
  - 비평 성공 시 `yield f"__CONFIDENCE:{json.dumps({'confidence': critique.get_confidence(), 'missing': critique.get_missing()})}"`를 방출한다.
  - 비평 실패 또는 `critique_generator` 부재 시 `confidence = 0.0, missing = []` 안전 폴백을 방출한다.
  - `not chunks`인 경우 기존대로 `confidence = 0.0, missing = []`를 방출한다.

`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- 스트림 토큰 방출 이후 수신되는 `_CONFIDENCE_PREFIX` 페이로드가 `last_confidence`와 `last_missing`에 온전히 바인딩되어 `publish_done` 및 `session.append_turn`에 정상 저장되도록 보장한다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 단순 질의 완료 시 `finalMeta`로부터 전달받은 검증 신뢰도(`confidence`)와 결여 정보(`missing`)를 AI 메시지 객체에 반영한다.
- `isGroundedVerified` 판정 시 실제 비평 검증을 통과한 경우(`confidence >= 0.8 && !hasMissing`)에만 `[✓ 지식베이스 근거 검증 완료]` 배지가 렌더링되도록 기존 검증 상태와 조화롭게 동작함을 확인한다.
- 비평 결과에 따라 신뢰도가 낮거나 결여 항목이 있는 경우 기존의 저신뢰도 배너(`low-confidence-warning`) 및 추가 확인 배지(`missing-warning-badge`)가 올바르게 분기 렌더링되도록 보장한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 단순 질의를 전송하여 유사도 0.9인 청크가 검색되고, 생성된 답변이 청크의 내용을 충실히 반영하여 비평 결과 `confidence = 0.85`, `missing = []`가 산출되었을 때
  When 답변 생성이 완료되어 `done` 이벤트가 프론트엔드에 수신되면
  Then AI 메시지 버블에 신뢰도 85% 배지와 함께 `[✓ 지식베이스 근거 검증 완료]` 배지(`grounded-verified-badge`)가 정상 표시된다.
- Given 사용자가 단순 질의를 전송하여 유사도 0.88인 청크가 검색되었으나, LLM이 청크와 무관한 환각 답변을 생성하여 비평 결과 `confidence = 0.35`, `missing = ["핵심 수치 정보 불일치"]`가 산출되었을 때
  When 답변 생성이 완료되면
  Then AI 메시지 버블에 `[✓ 지식베이스 근거 검증 완료]` 배지는 표시되지 않고, `[답변 신뢰도 낮음]` 경고 배너(`low-confidence-warning`)와 `[추가 확인 필요]` 배지(`missing-warning-badge`)가 렌더링된다.
- Given 사용자가 지식베이스에 존재하지 않는 내용에 대해 단순 질의를 전송하여 검색된 청크가 0건일 때 (경계 케이스)
  When 답변 생성이 완료되면
  Then 비평 호출 없이 `confidence = 0.0`, `missing = []`로 처리되어 "근거 문서 미발견" 경고 배너가 표시되고 검증 완료 배지는 표시되지 않는다.
- Given 비평 생성 서비스(`CritiqueGeneratorService`) 호출 중 LLM 게이트웨이 오류나 파싱 실패가 발생했을 때 (실패/예외 케이스)
  When 단순 질의 생성이 마무리되면
  Then 500 내부 에러로 중단되지 않고 안전 폴백(`confidence = 0.0`, `missing = []`)이 적용되어 검증 완료 배지 오표시가 방지된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:92-113` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/rag_composition.py:149-174` (`build_rag_composition`)
- 비평 수신 및 영속화 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:194-235` (`AskRequestedConsumer._process`)
- 프론트 검증 배지 렌더링부: `public-front/src/components/AiService.tsx:1816-1858` (`AiService`)
- 기존 비평 생성 서비스 구현: `public-python-server/src/ai_service/rag/application/critique_generator_service.py:50-80` (`CritiqueGeneratorService.generate`)
