---
id: SPEC-052
title: 에이전틱 RAG 다회차 검색 출처(Sources) 갱신 및 누적 전달
status: ready
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
에이전틱 RAG(`AgenticAskUseCase`)는 복잡한 질의에 대해 초기 검색 결과로 답변을 생성한 뒤, 비평(Critique)을 거쳐 정보가 부족하다고 판단되면 쿼리를 재작성(`QueryRefinerService.refine`)하여 2회차 이상의 검색과 답변 생성을 반복한다.
그러나 현재 백엔드 구현에서는 첫 번째 반복(`iteration == 1`)에서만 검색 청크의 출처(`__SOURCES`)를 방출하고, 이후 반복에서 새롭게 검색된 문서 청크들은 출처 목록에 반영되지 않고 버려지고 있다.
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:78-90` (`AgenticAskUseCase.execute`)에서 `if iteration == 1 and chunks:` 조건으로 인해 1회차에 검색된 청크에 대해서만 `__SOURCES` 프리픽스 문자열을 방출한다. 2회차 이후 비평을 통해 추가 검색된 청크(`chunks`)는 LLM 프롬프트 컨텍스트 구성에는 사용되지만, 출처 이벤트로는 클라이언트에 방출되지 않는다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:167-175` (`AskRequestedConsumer._process`)에서 `__SOURCES` 문자열이 도착할 때마다 `self._publisher.publish_sources`를 호출하여 Redis 스트림으로 방출하고 `sources` 변수에 할당한다. 하지만 UseCase가 1회차 이후 출처를 방출하지 않으므로, `:187-194`에서 세션 영속화(`session.append_turn`) 시 최종 세션 데이터에도 1회차 출처만 저장된다.
- `public-python-server/src/ai_service/rag/schemas.py:34-36` (`TurnValue`), `:50-54` (`ConversationTurn.of_assistant`)에 `sources` 필드가 지원되어 세션에 보존될 수 있음에도, 백엔드 UseCase의 조기 차단으로 인해 실제 참조된 최신 청크들이 세션 이력에서 누락된다.
- `public-front/src/components/AiService.tsx:90` (`AiService`) 및 `public-front/src/api/ai.ts:488-490` (`askQuestionStream`)은 SSE의 `sources` 이벤트를 수신하여 `setCurrentSources`로 상태를 갱신하도록 이미 준비되어 있으나, 백엔드에서 2회차 이후 갱신된 출처 이벤트를 발행하지 않아 최종 답변에 사용된 전체 근거 문서 목록을 화면에서 확인할 수 없다.

이는 "사용자가 답변의 근거를 스스로 확인할 수 있어야 한다"는 핵심 제품 원칙을 훼손한다. 사용자는 에이전트가 2회차 이상 재검색하여 최종 완성한 답변을 읽으면서도, 실제 답변 작성에 활용된 추가 참고 문서를 확인할 수 없어 답변의 신뢰성을 검증하기 어렵다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`에서 매 반복(`iteration`)마다 새롭게 검색된 청크들을 누적(또는 병합)하여 중복 없이 전체 출처 목록(`SourceRef`)을 생성하고, 청크가 추가되거나 갱신될 때 최신 출처 목록을 `__SOURCES:{json}` 형태로 방출한다.
- [ ] 여러 회차에 걸쳐 동일한 문서의 동일한 청크(`documentId` 및 `chunkIndex` 기준)가 검색된 경우 중복을 제거하고, 더 높은 유사도 점수(`score`)가 있다면 최신 점수로 갱신한다.
- [ ] 검색된 각 청크의 본문 요약(`snippet`)은 기존과 동일하게 최대 300자로 제한되고 PII 마스킹(`_secret_pii_scanner.mask`)이 적용된다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`에서 갱신된 `__SOURCES` 이벤트를 정상 수신하여 최신 전체 출처 목록으로 `publish_sources`를 발행하고 세션 턴(`session.append_turn`)에 최종 누적 출처를 저장한다.
- [ ] 2회차 이상 반복 검색이 발생하는 에이전틱 RAG 질의에 대해 다회차 검색 출처가 누적되어 방출되고 세션에 저장됨을 검증하는 백엔드 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 프론트엔드 UI 컴포넌트나 스타일시트 수정 (프론트엔드는 이미 `sources` 이벤트 수신 및 렌더링을 지원함).
- 하이브리드 검색 알고리즘이나 리랭커 가중치 로직의 변경.
- 비평기(`CritiqueGeneratorService`) 및 쿼리 리파이너(`QueryRefinerService`) 로직 수정.
- 단발성 비에이전틱 질의(`AskUseCase`)의 출처 처리 변경.

## 백엔드
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`:
  - 루프 진입 전 누적 청크/출처 컬렉션(예: `accumulated_sources: dict[str, dict[str, Any]]`)을 초기화한다.
  - 매 반복에서 `chunks`를 순회하며 청크 고유 키(`${documentId}:${chunkIndex}`)를 기반으로 중복 제거 및 누적한다.
  - 해당 반복에서 새로운 청크가 추가되거나 출처 목록이 갱신된 경우, 전체 누적 출처 목록을 직렬화하여 `yield f"__SOURCES:{json.dumps(sources)}"` 방출한다.
- `public-python-server/tests/unit/rag/test_agentic_ask_use_case.py`:
  - 1회차에 문서 A 검색 -> 2회차에 쿼리 리파이닝 후 문서 B 검색 시 최종 방출된 출처 목록에 문서 A와 문서 B가 모두 포함되어 있는지 검증하는 단위 테스트 추가.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 실행 시 1회차 검색에서 `doc-1 (chunk 0)`이 검색되고 비평 결과 정보 부족으로 2회차에 `doc-2 (chunk 1)`이 추가 검색되었을 때
  When `AgenticAskUseCase.execute` 스트림을 소비할 때
  Then 1회차에는 `doc-1` 출처가 방출되고, 2회차 검색 직후 `doc-1`과 `doc-2`가 모두 포함된 누적 출처 목록이 `__SOURCES` 이벤트로 방출된다.
- Given 1회차와 2회차 검색에서 동일한 `doc-1 (chunk 0)`이 중복 검색되었을 때
  When `AgenticAskUseCase.execute` 스트림을 소비할 때
  Then 중복된 출처 항목이 생성되지 않고 단일 출처 항목으로 병합되어 방출된다.
- Given 1회차 검색 청크가 0건이고 2회차 검색에서 청크가 발견된 경우
  When `AgenticAskUseCase.execute` 스트림을 소비할 때
  Then 2회차 검색 시점에 발견된 청크들의 출처 목록이 `__SOURCES` 이벤트로 정상 방출된다.
- Given 검색된 청크의 본문에 전화번호 등 개인정보(PII)가 포함된 경우
  When 누적 출처 목록이 방출될 때
  Then 모든 반복 회차의 출처 snippet에 대해 PII 마스킹(`[REDACTED_...]`)이 누락 없이 적용된다.

## 참고
- 에이전틱 RAG 반복 루프 및 출처 방출 지점: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:67-90` (`AgenticAskUseCase.execute`)
- 컨슈머 출처 이벤트 발행 및 세션 저장: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:167-175`, `:187-194` (`AskRequestedConsumer._process`)
- 프론트엔드 출처 이벤트 수신 핸들러: `public-front/src/api/ai.ts:488-490` (`askQuestionStream`)
