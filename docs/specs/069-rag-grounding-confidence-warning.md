---
id: SPEC-069
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 이번 분기 핵심 목표는 "답변을 믿을 수 있게 만든다"이며, "근거가 부족한 답과 충분한 답이 화면에서 구분되는가" 및 "답이 틀렸을 때 사용자가 그것을 알아챌 수 있는가"가 핵심 해결 과제다.
그러나 현재 비에이전틱 단발성 RAG 질의 흐름에서는 지식베이스 검색 결과가 없거나 관련도가 낮아도 신뢰도 및 누락 사유가 산출되지 않고, 화면에서도 일반 모델 지식 답변과 지식베이스 근거 답변이 구분되지 않는다.
- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색 후 검색된 조각(`chunks`)의 유사도 점수(`score`)를 바탕으로 검색 신뢰도를 계산하지 않으며, 청크가 0건일 때도 신뢰도 및 누락 사유 메타데이터(`confidence = 0.0`, `missing = ["관련 지식베이스 문서를 찾지 못함"]`)를 전달하지 않는다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)에서 단순 질의 처리 시 `last_confidence`와 `last_missing`이 계속 `None`으로 남아 세션 턴 저장(`session.append_turn`) 및 스트리밍 완료 이벤트(`publish_done`)에 신뢰도 데이터가 누락된다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서 AI 메시지의 출처가 없거나(`sources` 미존재 또는 빈 배열) 신뢰도가 0일 때 근거 미참조 경고 배너/배지가 표시되지 않아, 사용자는 답변이 등록된 문서에 기반한 것인지 모델의 사전 학습 지식(환각 가능성 포함)에 의한 것인지 파악할 수 없다.
- `public-front/src/api/ai.ts:428-442` (`askQuestionStream`)의 `onDone` 콜백이 완료 메타데이터(`AskDoneMeta`)를 처리할 수 있도록 준비되어 있으나 백엔드에서 단발성 질의 완료 시 유효한 신뢰도 데이터를 방출하지 않아 프론트엔드에서 활용되지 못하고 있다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`에서 하이브리드 검색 결과(`chunks`)가 존재할 때 검색된 청크들의 최고 유사도 점수(`max(c.score)`)를 기반으로 기본 검색 신뢰도(`confidence`)를 계산하여 전달한다.
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`에서 검색 결과 청크가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`의 단순 질의 처리 경로에서 `AskUseCase`가 산출한 `confidence`와 `missing`을 `last_confidence` 및 `last_missing`에 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 반영한다.
- [ ] `public-front/src/components/AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)를 신뢰도 배지(`confidence-badge`, 0~100%)로 시각화하여 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우, 해당 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] 대화 세션을 다시 불러왔을 때(`handleLoadSession`)에도 영속화된 단발성 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.
- [ ] `public-front/src/components/AiService.css`에 근거 부재 경고 배지/배너 스타일(`ungrounded-warning`)을 추가한다.
- [ ] 백엔드 신뢰도 산출 로직 및 프론트엔드 근거 미발견 경고 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 에이전틱 RAG 루프(`AgenticAskUseCase`)의 다단계 비평(Critique) 모델 프롬프트나 신뢰도 산출 수식 변경
- 하이브리드 검색 알고리즘 자체(BM25/Vector RRF 가중치) 수정 또는 리랭커 모델 교체
- 근거 문서가 없을 때 답변 생성을 원천 차단하는 가드레일 정책 강제 (경고 표시만 제공)

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`:
  - `AskUseCase.execute`에서 하이브리드 검색 후 청크가 있을 경우 최고 유사도 점수를 신뢰도로 산출(`confidence = round(max(c.score for c in chunks), 2)` 등)하고, 청크가 없을 경우 `confidence = 0.0`, `missing = ["관련 지식베이스 문서를 찾지 못함"]`을 이벤트/메타데이터로 전달할 수 있도록 구성
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `AskRequestedConsumer._process`의 단순 질의 경로에서 `AskUseCase`로부터 산출된 `confidence`, `missing` 값을 수집하여 `last_confidence`, `last_missing`에 할당
  - `session.append_turn` 호출 시 `confidence`, `missing` 인자로 전달하여 MongoDB에 저장
  - `publish_done` 호출 시 `done_data`에 `confidence`, `missing`을 포함하여 Redis Streams로 발행

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `handleSendQuestion`의 `onDone` 콜백에서 수신된 `confidence`, `missing`을 신규 AI `ChatMessage` 객체에 저장
  - 메시지 버블(`message-bubble`) 렌더링 시 `sources`가 없고 `confidence === 0` 또는 `missing`에 미발견 안내가 있는 경우 상단 또는 하단에 경고 배너(`.ungrounded-warning`) 노출
  - 유효한 `confidence`가 있는 경우 신뢰도 수준별 배지(`.confidence-badge`) 렌더링
- `public-front/src/components/AiService.css`:
  - `.ungrounded-warning` 경고 배너 스타일 (주황/적색 계열 보더, 배경, 경고 안내 텍스트) 정의

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 등록되어 있고 단발성 질의 검색 결과 청크(최고 점수 0.85)가 반환되었을 때
  When 사용자가 질문을 전송하고 답변 스트리밍이 완료되면
  Then 백엔드 `done` 이벤트 페이로드에 `confidence: 0.85`가 포함되어 전달되고, 프론트엔드 AI 메시지 버블에 신뢰도 배지("신뢰도: 85%")가 녹색(`confidence-high`)으로 렌더링되어야 한다.
- Given 지식베이스에 관련 문서가 전혀 없거나 검색 결과가 0건인 상태에서
  When 사용자가 단발성 질의를 전송하여 답변이 완료되면
  Then 백엔드는 `confidence: 0.0` 및 `missing: ["관련 지식베이스 문서를 찾지 못함"]`을 전달하고, 프론트엔드 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시되어야 한다.
- Given 근거 문서 미발견 경고가 포함된 대화 세션이 저장된 상태에서
  When 사용자가 사이드바에서 해당 세션을 선택하여 복원(`handleLoadSession`)하면
  Then 해당 AI 메시지 버블에 과거 저장된 신뢰도 0% 및 근거 미참조 경고 배너가 그대로 다시 렌더링되어야 한다.
- Given 신뢰도 정보가 없는 과거 레거시 세션 데이터(confidence: undefined, missing: undefined)를 불러왔을 때
  When 세션 상세 화면을 렌더링하면
  Then 오류나 화면 깨짐 없이 일반 텍스트 메시지만 정상 표시되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
- 관련 정의 및 이벤트 발행: `public-python-server/src/ai_service/core/events.py:46-54` (`JobEventPublisher.publish_done`)
