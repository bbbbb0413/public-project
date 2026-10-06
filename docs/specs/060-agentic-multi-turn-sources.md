---
id: SPEC-060
title: 에이전틱 RAG 다회차 검색 출처(Sources) 갱신 및 누적 전달
status: done
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
현재 복합 질의를 처리하는 에이전틱 RAG(`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:78-90` (`AgenticAskUseCase.execute`))에서는 1회차 검색 시에만 `if iteration == 1 and chunks:` 조건에 따라 검색된 출처(`sources`)를 방출한다.
하지만 에이전틱 RAG는 답변 비평(Critique) 후 신뢰도가 부족할 경우 질의를 재구성(`_query_refiner.refine`, `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:137-138` (`AgenticAskUseCase.execute`))하여 2회차, 3회차 추가 검색을 수행한다.
이 과정에서 새롭게 검색된 문서 청크들은 최종 답변을 생성하는 컨텍스트에는 포함되지만, 클라이언트로 방출되는 출처 목록(`__SOURCES`)에는 갱신되지 않고 버려진다.
결과적으로 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:167-175` (`AskRequestedConsumer._process`) 및 `public-python-server/src/ai_service/rag/schemas.py:48-64` (`ConversationTurn.of_assistant`)를 통해 대화 세션에 저장되는 최종 출처 목록에도 1회차 검색 출처만 영속화되어, 사용자가 다회차 검색을 통해 실제로 인용된 최신 근거 문서를 확인할 수 없게 된다. 이는 "답변을 믿을 수 있게 만든다"는 제품의 핵심 가치와 기존 근거 미리보기(SPEC-006) 성과를 훼손한다.

## 요구사항
- [ ] `AgenticAskUseCase.execute`에서 매 회차(`iteration`)마다 새롭게 검색된 문서 청크들을 누적 및 병합하여 전체 출처 목록(`sources`)을 관리해야 한다.
- [ ] 회차별 검색 결과 중 동일한 문서의 동일한 청크(`documentId`와 `chunkIndex`가 동일)가 중복 검색된 경우, 중복을 제거하고 더 높은 유사도 점수(`score`)를 우선 반영해야 한다.
- [ ] 2회차 이상 반복 검색 시 새로운 청크가 추가되거나 기존 청크의 점수가 갱신되면 최신 전체 출처 목록을 `__SOURCES:{json}` 형식으로 클라이언트에 방출해야 한다.
- [ ] 검색된 각 청크의 본문 요약(`snippet`)은 최대 300자로 제한되고 PII 마스킹(`_secret_pii_scanner.mask`)이 정상 적용되어야 한다.
- [ ] `AskRequestedConsumer._process`에서 다회차에 걸쳐 수신된 최신 `__SOURCES` 페이로드를 바탕으로 세션(`session.append_turn`)에 누적된 최종 출처 목록을 정상 저장해야 한다.

## 비요구사항 (Out of scope)
- 프론트엔드 UI 컴포넌트(`AiService.tsx`)의 출처 렌더링 스타일 수정 (기존 `__SOURCES` 수신 처리 유지).
- 단발성 비에이전틱 질의(`AskUseCase.execute`)의 출처 방출 로직 변경.
- RAG 하이브리드 검색 알고리즘이나 리랭커 가중치 조정.

## 백엔드
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
  - `execute` 메서드 내부에 전체 회차 청크를 누적·병합하는 `accumulated_chunks` 또는 `accumulated_sources` 관리 로직 구현.
  - 청크 병합 시 고유 키 `(documentId, chunkIndex)`를 기준으로 중복 제거 및 최고 점수(`score`) 갱신.
  - 매 `iteration`에서 새 청크가 추가되거나 갱신될 때마다 최신 누적 출처를 `__SOURCES:{json}` 형식으로 `yield`.
- `public-python-server/tests/rag/test_agentic_ask_use_case.py` (또는 해당 테스트 스위트)
  - 2회차 이상 루프 실행 시 1회차와 2회차 검색 출처가 모두 포함된 누적 `__SOURCES` 이벤트 방출 및 중복 제거 검증 단위 테스트 작성.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 복합 질문을 요청하여 에이전틱 RAG 루프가 2회 이상 실행될 때
  When 1회차에서 문서 A(청크 0)가 검색되고, 2회차에서 문서 B(청크 1)가 추가 검색되면
  Then 최종 방출되는 `__SOURCES` 이벤트 및 세션에 저장되는 `sources` 목록에 문서 A와 문서 B의 출처가 모두 포함되어야 한다.
- Given 1회차와 2회차 검색에서 동일한 문서 A의 청크 0이 다시 검색될 때 (1회차 score: 0.75, 2회차 score: 0.88)
  When 출처 목록을 병합하면
  Then 중복 항목 없이 1건만 유지되고 최고 유사도 점수인 0.88이 반영되어야 한다.
- Given 검색 결과 청크(`chunks`)가 0건인 회차가 발생할 때
  When 루프가 진행되면
  Then 이전 회차까지 누적된 출처 목록이 유실되지 않고 안전하게 보존되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:78-90` (`AgenticAskUseCase.execute`)
- 이벤트 처리 및 세션 저장소 반영 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:167-195` (`AskRequestedConsumer._process`)
- 출처 데이터 구조 정의: `public-python-server/src/ai_service/rag/schemas.py:48-64` (`ConversationTurn.of_assistant`)
