---
id: SPEC-072
title: 에이전틱 RAG 루프 중 실시간 토큰 스트리밍 지원
status: done
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
현재 에이전틱 RAG(복합 질의 처리) 수행 시 `AgenticAskUseCase.execute`에서 LLM이 생성하는 스트리밍 토큰을 클라이언트로 즉시 방출(`yield`)하지 않고 내부 리스트 `collected`에 전부 취합한 뒤, 비평(Critique) 및 쿼리 개선(Refining) 루프가 완전히 종료된 이후에야 완성된 텍스트를 한 번에 `yield`하고 있다.
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:98-114` (`AgenticAskUseCase.execute`)에서 `collected = [token async for token in self._llm_gateway.stream(...)]` 형태로 비동기 제너레이터를 리스트 컴프리헨션으로 모두 소진한 후, 루프 및 비평 검증이 끝난 113줄 및 130줄에서야 `yield self._secret_pii_scanner.mask(raw)`로 한 번에 방출한다.
- 반면 비에이전틱 단발성 질의인 `public-python-server/src/ai_service/rag/application/ask_use_case.py:126-135` (`AskUseCase.execute`)에서는 `async for token in self._llm_gateway.stream(...)`으로 LLM이 생성하는 각 토큰을 즉시 `yield token`하여 프론트엔드로 실시간 전달하고 있다.
- 이로 인해 복합 질의를 사용하는 사용자는 질문 전송 후 `searching`, `generating`, `critiquing` 등의 진행 상태 이벤트만 지켜보며 수 초에서 십수 초 동안 화면에 아무런 글자도 나타나지 않는 긴 대기 시간과 시스템 멈춤 오인을 겪게 된다.
- 또한 백엔드 컨슈머 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:161-184` (`AskRequestedConsumer._process`), 게이트웨이 SSE 엔드포인트 `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39-65` (`JobStreamController.stream`), 프론트엔드 `public-front/src/components/AiService.tsx:667-671` (`handleSendQuestion`)는 이미 토큰 단위의 실시간 SSE 스트리밍 및 누적 렌더링 파이프라인을 온전히 갖추고 있다. `AgenticAskUseCase`에서 토큰을 생성 즉시 방출하도록 개선하면 타 저장소 변경 없이 백엔드 개선만으로 체감 응답 속도를 크게 개선할 수 있다.

## 요구사항
- [ ] `AgenticAskUseCase.execute`에서 각 반복 회차(`iteration`)의 답변 생성 단계 시 LLM Gateway로부터 전달되는 개별 토큰을 즉시 `yield`하여 실시간 토큰 스트리밍을 제공한다.
- [ ] 토큰 스트리밍 과정 중에도 PII 및 기밀정보 마스킹(`_secret_pii_scanner.mask`)이 토큰 단위 또는 방출 단위에 누락 없이 적용되어야 한다.
- [ ] 토큰 생성 중 토큰 사용량 및 경과 시간 예산(`command.budget.is_exhausted`)을 추적하여 한도 초과 시 즉시 생성을 중단하고 조기 종료한다.
- [ ] 비평(Critique) 결과 신뢰도 기준을 만족하지 못해 다음 반복 회차(`iteration >= 2`)로 재검색 및 재생성이 진행되는 경우, 클라이언트가 이전 회차 생성물을 인지하고 새로운 생성 스트림으로 원활하게 이어받을 수 있도록 처리한다.
- [ ] 작업 중단(`is_cancelled`) 플래그 감지 시 실시간 스트리밍 루프가 즉시 중단되고 리소스가 안전하게 정리되어야 한다.
- [ ] 에이전틱 RAG 질의 실행 시 LLM 토큰이 실시간으로 비동기 이터레이터를 통해 순차 방출됨을 검증하는 백엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 프론트엔드 UI 컴포넌트나 CSS 스타일 변경 (이미 토큰 누적 렌더링을 지원함)
- 게이트웨이 SSE 스트리밍 라우터나 Redis Streams 이벤트 프로토콜 규격 변경
- 비평(Critique) 모델 프롬프트나 신뢰도 판정 공식, 하이브리드 검색 알고리즘 교체
- 단발성 비에이전틱 RAG 질의 경로(`AskUseCase`)의 스트리밍 로직 수정

## 백엔드
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
  - `execute` 메서드 내 LLM 응답 수신 루프 개선: `self._llm_gateway.stream(...)`의 토큰을 `collected` 리스트에 누적함과 동시에 호출자에게 즉시 `yield`하도록 변경
  - 스트리밍 중 토큰 길이 기반 토큰 카운팅 및 예산(`command.budget`) 검증 연동
  - 비평 불만족으로 인한 추가 반복 진행 시 스트림 제어 처리
- `public-python-server/tests/rag/test_agentic_ask_use_case.py` (또는 해당 테스트 스위트)
  - `AgenticAskUseCase.execute` 호출 시 제너레이터가 전체 생성이 끝나기 전에 개별 토큰을 순차적으로 `yield`하는지 검증하는 단위 테스트 추가

## 수용 기준 (Acceptance Criteria)
- Given 복합 질의(`complexity: "complex"`)로 에이전틱 RAG 작업이 실행되었을 때
  When LLM Gateway가 답변 토큰을 순차적으로 반환하면
  Then 비평(Critique) 완료를 기다리지 않고 생성되는 각 토큰이 즉시 `yield`되어 클라이언트로 실시간 스트리밍되어야 한다.
- Given 에이전틱 RAG 실행 중 생성된 누적 토큰 수가 설정된 예산 한도(`command.budget.is_exhausted`)에 도달했을 때
  When 한도 초과가 감지되면
  Then 즉시 추가 토큰 생성을 중단하고 스트림이 정상 종료되어야 한다.
- Given 검색된 문서 청크가 0건(`chunks == []`)이거나 빈 응답인 경계 상황일 때
  When 에이전틱 RAG가 실행되면
  Then 예외 에러 없이 안전하게 완료되거나 빈 스트림/안내 토큰이 정상적으로 방출되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:98-114` (`AgenticAskUseCase.execute`)
- 백엔드 이벤트 컨슈머 스트리밍 루프: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:161-184` (`AskRequestedConsumer._process`)
- 게이트웨이 SSE 스트림 구독 진입점: `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39-65` (`JobStreamController.stream`)
- 프론트엔드 스트리밍 수신 처리: `public-front/src/components/AiService.tsx:667-671` (`handleSendQuestion`)
- 단발성 질의 정상 스트리밍 참고 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:126-135` (`AskUseCase.execute`)
