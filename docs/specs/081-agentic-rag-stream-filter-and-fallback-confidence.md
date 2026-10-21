---
id: SPEC-081
title: 에이전틱 RAG 응답 스트리밍 추론 라인 필터링 및 예산 소진 시 신뢰도 산출 연동
status: ready
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
현재 복합 질문을 처리하는 에이전틱 RAG 실행 경로(`AgenticAskUseCase`)에서 단순 RAG 질의 경로(`AskUseCase`)와 비교해 스트리밍 응답 정제와 신뢰도 산출이 누락되는 결함이 존재한다.
단순 RAG 질의인 `public-python-server/src/ai_service/rag/application/ask_use_case.py:32-41` (`_FILTERED_LINE_PATTERNS`, `_is_filtered_line`) 및 `public-python-server/src/ai_service/rag/application/ask_use_case.py:160-182` (`AskUseCase._stream_filtered`)에서는 LLM의 내부 추론 흔적(`Step 1:`, `## Step 2`, `The final answer is`, `In conclusion`, `To summarize`, `\boxed{}`)을 필터링하고 연속 빈 줄을 1개로 제한하여 정제된 토큰만 스트리밍한다.
그러나 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:139-163` (`AgenticAskUseCase.execute`)의 스트리밍 루프에서는 `_stream_filtered` 적용 없이 LLM Gateway에서 방출된 토큰을 즉시 `yield`하고 있어, 복합 질문 응답 시 LLM의 중간 추론 단계나 내부 메타 텍스트가 사용자 화면에 그대로 실시간 노출된다.
또한 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:138-163` (`AgenticAskUseCase.execute`)에서 1회차 답변 생성 도중 또는 직후 토큰/시간 예산이 소진되어 조기 종료(`budget_exhausted` 또는 `command.budget.is_exhausted`)될 때, 비평 단계(`_critique_generator.generate`)를 거치지 못하고 즉시 `return`한다. 이 경우 초기값인 `last_confidence = 0.0`, `last_missing = []`이 그대로 유지된 채 종료된다.
그 결과 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:120-142` (`AskRequestedConsumer._process`)가 수집하는 완료 이벤트(`publish_done`)와 세션 턴(`session.append_turn`)에 신뢰도 `0.0`이 기록되어, 정상적으로 지식 문서를 참조해 답변했음에도 화면에 근거 부재 경고가 발생하거나 0% 위험 배지가 표시되어 사용자가 답변의 진위를 왜곡하여 판단하게 만든다.

## 요구사항
- [ ] 단순 질의(`AskUseCase`)의 스트리밍 라인 필터링 로직(`_FILTERED_LINE_PATTERNS`, `_is_filtered_line`, `_stream_filtered`)을 에이전틱 RAG(`AgenticAskUseCase`)의 토큰 스트리밍 생성기에도 일관되게 적용한다.
- [ ] 에이전틱 RAG 스트리밍 중 `Step 1:`, `## Step 2`, `The final answer is`, `In conclusion`, `To summarize`, `\boxed{}` 등의 중간 추론 흔적 라인이 클라이언트로 방출되지 않아야 한다.
- [ ] 에이전틱 RAG 스트리밍 중 연속된 빈 줄은 최대 1개(`\n`)만 유지되도록 정리되어야 한다.
- [ ] 토큰 스트리밍 필터링 과정에서 기존의 PII/비밀정보 마스킹(`_secret_pii_scanner.mask`)이 온전하게 유지되어야 한다.
- [ ] 에이전틱 1회차에서 예산(시간 또는 토큰) 소진으로 비평을 수행하지 못하고 조기 종료될 때, 검색된 지식 문서 청크(`chunks`)가 존재하면 최고 유사도 점수(`max(c.score for c in chunks)`)를 기반으로 기본 신뢰도를 산출한다.
- [ ] 에이전틱 1회차에서 예산 소진으로 비평을 수행하지 못할 때 `last_missing`에 `["답변 검증 예산 소진으로 상세 비평 생략"]` 안내를 포함하여 원인 불명의 빈 누락 목록 대신 명확한 사유를 전달한다.
- [ ] 2회차 이상 반복 검증 후 예산이 소진된 경우, 이전 회차에서 평가된 유효 신뢰도(`last_confidence`) 및 누락 항목(`last_missing`)을 안전하게 보존하여 반환한다.
- [ ] 검색된 청크가 전혀 없는 상태(`not chunks`)에서 예산 소진 등으로 종료될 경우 신뢰도 `0.0` 및 `["관련 지식베이스 문서를 찾지 못함"]`을 전달한다.
- [ ] 변경된 스트리밍 필터링 및 예산 소진 시 폴백 신뢰도 산출 동작을 검증하는 백엔드 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 하이브리드 검색 알고리즘이나 리랭커, 쿼리 임베딩 로직 자체를 변경하지 않는다.
- 비평기(`CritiqueGeneratorService`)의 프롬프트 템플릿이나 JSON 스키마를 수정하지 않는다.
- 프론트엔드 UI 컴포넌트나 CSS 스타일을 수정하지 않는다.
- 토큰/시간 예산 산정 규칙(`approx_chars_per_token`, 타임아웃 제한 등) 자체를 변경하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/stream_filter.py` (또는 `ask_use_case.py` / `agentic_ask_use_case.py` 공통 모듈)
  - `_FILTERED_LINE_PATTERNS`, `_is_filtered_line`, `filter_token_stream` 유틸리티 함수 분리 또는 공통 재사용 구성.
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
  - `AgenticAskUseCase.execute` 내 `self._llm_gateway.stream(...)` 소비 시 라인 필터링 및 PII 마스킹을 결합한 스트림 필터 적용.
  - 예산 소진 분기(`budget_exhausted or command.budget.is_exhausted`) 진입 시, `last_confidence`가 `0.0`인 경우 `chunks` 존재 여부에 따라 청크 최고 유사도 점수 기반 신뢰도 및 사유(`"답변 검증 예산 소진으로 상세 비평 생략"`) 바인딩 후 진행 이벤트 발행 및 반환.
- `public-python-server/tests/`
  - 에이전틱 RAG 스트리밍 시 CoT 추론 라인 필터링 검증 테스트.
  - 1회차 예산 소진 시 청크 점수 기반 신뢰도 산출 및 missing 사유 검증 테스트.

## 수용 기준 (Acceptance Criteria)
- Given 복합 질문에 대해 LLM Gateway가 `Step 1: 분석\n\n문서 내용에 따르면 정답입니다.\nThe final answer is 정답` 형식으로 토큰을 생성할 때
  When `AgenticAskUseCase.execute`를 통해 스트리밍을 수신하면
  Then `Step 1: 분석`과 `The final answer is 정답` 라인은 클라이언트로 방출되지 않고 정제된 답변 본문만 방출되어야 한다.
- Given 1회차 실행 중 지식 문서 청크 2건(유사도 점수 0.85, 0.72)이 검색되었으나 토큰/시간 예산이 소진되어 조기 종료될 때
  When 스트리밍이 종료되고 최종 progress/done 이벤트 및 세션 저장이 수행되면
  Then 신뢰도(`confidence`)는 0.0이 아닌 청크 최고 점수인 0.85로 산출되고, `missing`에는 `"답변 검증 예산 소진으로 상세 비평 생략"`이 포함되어야 한다.
- Given 1회차에 검색된 지식 문서 청크가 0건(`chunks = []`)이고 예산이 소진되어 조기 종료될 때 (경계 케이스)
  When 스트리밍이 종료되면
  Then 신뢰도(`confidence`)는 0.0으로 유지되고 `missing`에는 `["관련 지식베이스 문서를 찾지 못함"]`이 안전하게 반영되어야 한다.
- Given 1회차 비평에서 신뢰도 0.65가 산출된 후 2회차 생성 중 예산이 소진되어 조기 종료될 때
  When 스트리밍이 종료되면
  Then 이전 회차에서 검증된 신뢰도 0.65와 해당 회차의 `missing` 정보가 손실 없이 유지되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:139-163` (`AgenticAskUseCase.execute`)
- 참조할 기존 필터 구현: `public-python-server/src/ai_service/rag/application/ask_use_case.py:32-41` (`_FILTERED_LINE_PATTERNS`, `_is_filtered_line`)
- 참조할 기존 필터 스트림: `public-python-server/src/ai_service/rag/application/ask_use_case.py:160-182` (`AskUseCase._stream_filtered`)
- 참조할 기존 단순 RAG 신뢰도 산출: `public-python-server/src/ai_service/rag/application/ask_use_case.py:96-118` (`AskUseCase.execute`)
- 컨슈머 진행 및 완료 수신부: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:120-142` (`AskRequestedConsumer._process`)
