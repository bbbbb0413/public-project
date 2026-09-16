---
id: SPEC-046
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 질의응답 시스템은 문서 검색을 기반으로 답변을 생성하지만, 비에이전틱 단발성 질의(`AskUseCase`) 시 검색 결과의 신뢰도 및 근거 유무 메타데이터를 클라이언트로 온전히 전달하지 않아 사용자가 답변의 근거 충분성을 신뢰하거나 검증하기 어렵다.

- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 `HybridSearchUseCase`를 통해 검색된 `chunks`가 존재할 때만 `__SOURCES` 문자열을 방출하고, 검색 결과가 0건이거나 관련도가 낮은 경우에도 신뢰도 수치나 누락 원인(`missing`) 메타데이터를 일체 생성하지 않고 일반 LLM 모델 지식만으로 답변을 생성한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115`, `:138-155` (`AskRequestedConsumer._process`)에서 단발성 질문(`complexity != "complex"`)의 경우 `last_confidence`와 `last_missing`을 계속 `None`으로 유지하여, `:196-203`의 `publish_done` 완료 통지 및 `:187-193`의 세션 턴 영속화 시 신뢰도 정보가 누락된다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService` / `chat-modal-messages`)에서 `msg.confidence !== undefined`인 경우에만 신뢰도 배지를 렌더링하므로, 대다수의 일반 질의응답이나 지식베이스 문서가 매칭되지 않은 답변(0건 참조)에 대해 신뢰도 배지가 아예 표시되지 않으며 사용자는 해당 답변이 업로드된 문서를 참고했는지 아니면 모델의 자체 환각인지 구분할 수 없다.

이는 "올려 둔 문서를 근거로 질문에 답하되 사용자가 스스로 확인할 수 있어야 한다"는 제품 원칙에 어긋나며, 근거가 부족하거나 없는 답변에 대해 시각적 경고를 명확히 제공하여 답변 신뢰성을 확보해야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`에서 하이브리드 검색 청크가 존재할 때 최고 유사도 점수(`max(c.score)`)를 기반으로 검색 신뢰도를 계산하고, 청크가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 스트림 또는 반환 규격으로 전달한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`의 비에이전틱 경로에서 `AskUseCase`의 검색 신뢰도 및 누락 메타데이터를 수집하여 `last_confidence`, `last_missing`에 저장하고, 세션 턴(`session.append_turn`) 및 완료 이벤트(`publish_done`)에 반영한다.
- [ ] `public-front/src/components/AiService.tsx`에서 AI 메시지의 `sources`가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 있는 경우, 메시지 버블 상단/하단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`에서 단발성 질의의 신뢰도 수치(예: 0.85 -> 85%)가 완료 이벤트 및 세션 복원 시 신뢰도 배지(`confidence-badge`)로 정상 렌더링된다.
- [ ] `public-front/src/components/AiService.css`에 근거 부재 경고 배지/배너 스타일(`ungrounded-warning`)을 추가한다.
- [ ] 백엔드 신뢰도 계산 및 프론트엔드 근거 미발견 경고 배너 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 하이브리드 검색 알고리즘이나 리랭커 가중치 수식을 교체하는 작업.
- 에이전틱 RAG(`AgenticAskUseCase`)의 다중 반복 비평 로직 수정.
- 문서 인제스트 파이프라인의 청킹 및 임베딩 처리 변경.
- 결제 및 인증 등 RAG 질의응답 이외의 도메인 로직 변경.

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`:
  - `HybridSearchResult`의 청크 점수로부터 검색 신뢰도 산출 로직 추가.
  - 검색 결과가 없을 때 `confidence: 0.0` 및 `missing: ["관련 지식베이스 문서를 찾지 못함"]` 정보 전달 지원.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - 단순 질의 스트리밍 완료 시 산출된 신뢰도 및 누락 정보를 `last_confidence`, `last_missing`에 할당.
  - 세션 영속화 및 `done` 이벤트 발행 시 메타데이터 전달.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - AI 메시지 버블 내에 근거 문서 미참조 경고 배너(`ungrounded-warning`) 조건부 렌더링 추가.
  - 단발성 질의 응답의 신뢰도 배지 정상 노출 확인.
- `public-front/src/components/AiService.css`:
  - `.ungrounded-warning` 경고 스타일(주황/황색 계열의 가시성 높은 안내 테두리 및 텍스트) 정의.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스에 없는 내용의 질문을 전송하여 검색 청크가 0건일 때
  When AI 답변 생성이 완료되면
  Then AI 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배지가 표시되고 신뢰도는 0%로 표시된다.
- Given 사용자가 지식베이스에 등록된 내용의 질문을 전송하여 관련 청크(점수 0.85)가 검색되었을 때
  When AI 답변 생성이 완료되면
  Then AI 메시지 버블에 85% 신뢰도 배지(`confidence-high`)와 함께 참고 문서 목록이 정상 렌더링된다.
- Given 과거 레거시 세션 데이터처럼 신뢰도와 출처가 모두 `undefined`인 경우
  When 세션을 불러와 렌더링할 때
  Then 경고 배너가 불필요하게 오작동하지 않고 기존 기본 텍스트 메시지로 안전하게 표시된다.

## 참고
- 단발성 질의 실행 및 출처 방출: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 컨슈머 스트림 소비 및 메타데이터 영속화: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115`, `:138-155`, `:187-203` (`AskRequestedConsumer._process`)
- 프론트엔드 메시지 신뢰도 및 출처 렌더링: `public-front/src/components/AiService.tsx:1104-1132` (`AiService` / `chat-modal-messages`)
