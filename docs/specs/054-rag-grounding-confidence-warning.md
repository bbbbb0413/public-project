---
id: SPEC-054
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 지식베이스 기반 RAG 서비스에서 복합 질의(Agentic RAG)와 달리 단발성(단순) RAG 질의의 경우 신뢰도 및 누락 정보 산출 로직이 누락되어 있다.
`public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서는 하이브리드 검색(`_hybrid_search.execute`)을 수행하여 청크(`chunks`)를 가져오고 출처(`__SOURCES`)를 방출하지만, 검색된 문서의 관련도 점수를 바탕으로 한 신뢰도나 검색 실패 시 누락 사유를 산출하여 전달하지 않는다.
이로 인해 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)에서 비에이전틱 단발성 질의 처리 시 `last_confidence`와 `last_missing`이 계속 `None`으로 유지되어, 스트리밍 완료 이벤트(`publish_done`)에 `done_data`가 비어 전송되고 세션 턴 영속화(`session.append_turn`)에도 신뢰도 데이터가 누락된다.
결과적으로 프론트엔드 `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서는 단발성 질의 답변에 대해 신뢰도 배지(`confidence-badge`)가 렌더링되지 않으며, 검색된 근거 청크가 전혀 없거나 관련도가 매우 낮은 경우에도 경고 없이 모델의 일반 지식 기반 답변이 마치 지식베이스에 근거한 것처럼 노출되어 사용자가 답변의 진위를 신뢰하기 어렵다.

## 요구사항
- [ ] 단발성 RAG 질의 처리 시 검색된 청크의 유사도 점수(`c.score`)를 기반으로 검색 신뢰도(`confidence`)를 산출하고, 검색 결과가 0건일 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]`을 전달한다.
- [ ] 백엔드 메시지 컨슈머(`AskRequestedConsumer`)에서 단순 RAG 질의 수행 후 산출된 `confidence`와 `missing`을 완료 이벤트(`publish_done`) 페이로드와 세션 턴 저장(`session.append_turn`)에 반영한다.
- [ ] 프론트엔드 `AiService`에서 단발성 질의 완료 시 전달받은 신뢰도(0~1 수치)를 신뢰도 배지(0~100%)로 시각화하여 렌더링한다.
- [ ] 프론트엔드 `AiService`에서 AI 메시지의 출처 문서가 없거나(`sources` 미존재 또는 빈 배열), `confidence === 0` 또는 `missing`에 문서 미발견 항목이 존재하는 경우 메시지 상단에 근거 문서 미참조 경고 배너를 렌더링한다.
- [ ] 근거 미참조 경고 배너에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 문구와 경고 안내 스타일을 적용한다.
- [ ] 세션 복원 시에도 단발성 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.

## 비요구사항 (Out of scope)
- 하이브리드 검색 알고리즘 및 RRF(Reciprocal Rank Fusion) 가중치 자체의 변경
- 에이전틱 RAG 루프 비평(Critique) 프롬프트 구조의 변경
- 지식베이스 문서 업로드 및 파싱 파이프라인의 변경
- 다중 사용자 협업 또는 대화 세션 공유 기능 추가

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`:
  - `AskUseCase.execute`에서 하이브리드 검색 결과 `chunks`가 존재하는 경우 청크들의 최고 점수 또는 평균 점수를 정규화하여 신뢰도(`confidence`)를 계산하고, 청크가 0건인 경우 `confidence = 0.0`, `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 스트림 또는 제너레이터 메타 이벤트(`__META:{json}`) 규격으로 전달한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - 단순 질의 경로에서 `AskUseCase`가 방출한 신뢰도 및 누락 메타데이터를 파싱하여 `last_confidence`와 `last_missing`을 갱신한다.
  - 스트리밍 종료 시 `session.append_turn`에 `confidence`, `missing`을 영속화하고, `publish_done` 완료 이벤트 페이로드에 `{ confidence, missing }`을 담아 발행한다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `ChatMessage` 인터페이스의 `confidence`, `missing`, `sources` 데이터를 기반으로 단발성 질의 완료 시 신뢰도 배지가 정상 렌더링되도록 확인 및 보강한다.
  - AI 메시지 버블 내에서 `(!msg.sources || msg.sources.length === 0) && (msg.confidence === 0 || (msg.missing && msg.missing.includes('관련 지식베이스 문서를 찾지 못함')))` 조건에 부합할 경우 `ungrounded-warning` 경고 배너를 렌더링한다.
- `public-front/src/components/AiService.css`:
  - `.ungrounded-warning` 클래스에 눈에 띄는 경고 테두리, 배경색 및 아이콘 스타일을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 질의와 관련된 문서가 등록되어 있을 때, When 사용자가 단발성 질문을 전송하고 생성이 완료되면, Then 응답 버블에 검색 점수를 기반으로 한 신뢰도 배지(예: 신뢰도: 85%)가 표시된다.
- Given 지식베이스에 질의와 관련된 문서가 전혀 없을 때(검색 결과 0건), When 사용자가 단발성 질문을 전송하고 생성이 완료되면, Then 응답 버블 상단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시되고 신뢰도 배지에 0%가 표시된다.
- Given 근거 문서가 미발견된 과거 대화 세션을 다시 불러왔을 때(세션 복원), When 채팅 내역이 렌더링되면, Then 해당 AI 답변 버블에 동일하게 근거 문서 미발견 경고 배너가 유지되어 표시된다.
- Given 단발성 질의 완료 이벤트가 발생했을 때, When `done` 이벤트 페이로드를 검사하면, Then `confidence` 수치 및 `missing` 배열이 포함되어 수신된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
