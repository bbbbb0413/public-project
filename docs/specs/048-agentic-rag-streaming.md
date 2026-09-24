---
id: SPEC-048
title: 에이전틱 RAG 루프 중 실시간 토큰 스트리밍 지원
status: done
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제

현재 에이전틱 RAG(복합 질의 처리) 수행 시 `AgenticAskUseCase.execute`에서 LLM이 생성하는 스트리밍 토큰을 클라이언트로 즉시 방출(`yield`)하지 않고 내부 리스트 `collected`에 전부 취합한 뒤, 비평(Critique) 및 쿼리 개선(Refining) 루프가 완전히 종료된 이후에야 완성된 텍스트를 한 번에 `yield`하고 있다.
`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:98-114, 130` (`AgenticAskUseCase.execute`)

이로 인해 비에이전틱 단발성 질의(`AskUseCase.execute`, `public-python-server/src/ai_service/rag/application/ask_use_case.py:122-135`)에서는 사용자가 실시간으로 글자가 타이핑되는 스트리밍 텍스트를 보며 대기 시간을 체감하지 않는 반면, 에이전틱 RAG 질의 시에는 `searching`, `generating`, `critiquing` 등의 진행 상태 이벤트만 표시되다가 수초에서 십수초 후 완성된 긴 답변이 한꺼번에 화면에 출력되어 심각한 체감 지연과 시스템 멈춤 오인 현상이 발생한다.

또한 백엔드 컨슈머 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:161-180` (`AskRequestedConsumer._process`) 및 게이트웨이 SSE 스트림 `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39-65` (`stream`), 프론트엔드 `public-front/src/components/AiService.tsx:667-671` (`handleSendQuestion`)는 이미 토큰 단위 실시간 스트리밍 처리를 완벽히 지원하고 있으므로, `AgenticAskUseCase`에서 각 반복 회차 생성 단계의 토큰을 실시간으로 `yield`하고 비평 불만족으로 다음 반복으로 넘어갈 때만 스트림 초기화 또는 진행 상태 제어를 맞추면 클라이언트 수정 없이도 부드러운 스트리밍 사용자 경험을 제공할 수 있다.

## 요구사항

- [ ] `AgenticAskUseCase.execute`에서 LLM Gateway로부터 수신되는 각 토큰을 생성되는 즉시 `yield`하여 실시간 토큰 스트리밍을 제공한다.
- [ ] 토큰 스트리밍 중에도 PII/비밀정보 마스킹(`_secret_pii_scanner.mask`) 정책 및 토큰 사용량 버짓(`command.budget`) 검사가 올바르게 유지되어야 한다.
- [ ] 비평(Critique) 결과 신뢰도 기준을 충족하지 못해 추가 검색/재생성(iteration >= 2)으로 넘어갈 경우, 이전 회차의 중간 생성물을 클라이언트가 인지하고 새로운 생성 스트림으로 원활하게 이어받을 수 있도록 처리한다.
- [ ] 작업 중단(`is_cancelled`) 플래그 감지 시 실시간 스트리밍 중 즉시 생성이 중단되고 스트림이 정상 종료되어야 한다.
- [ ] 변경 사항에 대한 백엔드 단위 테스트(Unit Tests)를 작성하여 에이전틱 실행 시 토큰이 제너레이터를 통해 실시간으로 `yield`되는지 검증한다.

## 비요구사항 (Out of scope)

- 프론트엔드 UI 컴포넌트나 CSS 스타일 변경 (이미 토큰 누적 렌더링 지원)
- 게이트웨이 SSE 스트리밍 파이프라인이나 Redis Streams 프로토콜 변경
- 비평(Critique) 프롬프트나 하이브리드 검색 알고리즘 자체의 교체 또는 튜닝
- 단발성 비에이전틱 RAG(`AskUseCase`)의 스트리밍 로직 변경

## 백엔드

- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
  - `AgenticAskUseCase.execute` 내부의 `_llm_gateway.stream` 호출 시 토큰을 버퍼에만 모으지 않고 호출자에게 즉시 `yield`하도록 개선
  - 예산 초과 또는 비평 통과 시 최종 마스킹 텍스트 처리와 실시간 토큰 방출 흐름 정합성 유지
- `public-python-server/tests/rag/test_agentic_ask_use_case.py`
  - 에이전틱 질의 실행 시 토큰이 실시간으로 비동기 이터레이터(`AsyncIterator`)를 통해 순차 방출되는지 검증하는 단위 테스트 추가

## 수용 기준 (Acceptance Criteria)

- Given 복합 질의(`complexity: "complex"`)로 에이전틱 RAG 작업이 요청되었을 때
  When LLM이 답변 토큰을 순차적으로 생성하면
  Then 전체 비평 완료를 기다리지 않고 생성되는 즉시 개별 토큰이 `yield`되어 클라이언트로 스트리밍되어야 한다.
- Given 에이전틱 RAG 실행 중 생성된 토큰 수가 예산 한도(`command.budget.is_exhausted`)에 도달했을 때
  When 한도 초과가 감지되면
  Then 즉시 추가 생성을 멈추고 스트림을 정상 종료해야 한다.
- Given 검색된 문서 청크가 0건(`chunks == []`)이거나 빈 응답인 경계 상황일 때
  When 에이전틱 실행이 시작되면
  Then 에러 없이 빈 스트림 또는 기본 폴백 안내 토큰이 정상적으로 방출되어야 한다.

## 참고

- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:98-114` (`AgenticAskUseCase.execute`)
- 이벤트 발행 및 컨슈머: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:161-180` (`AskRequestedConsumer._process`)
- 프론트 스트리밍 수신 처리: `public-front/src/components/AiService.tsx:667-671` (`handleSendQuestion`)
