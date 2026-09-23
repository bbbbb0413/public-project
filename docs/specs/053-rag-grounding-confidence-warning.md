---
id: SPEC-053
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 서비스는 복합 질의(Agentic RAG) 시에만 비평기(Critique) 루프를 통해 신뢰도(`confidence`) 및 누락 사유(`missing`)를 산출하고 있습니다.
반면 단발성 단순 질의(`AskUseCase`)의 경우, 지식베이스 검색 결과가 0건이거나 관련도가 낮은 상태에서 답변을 생성하더라도 신뢰도 점수와 누락 정보가 계산되지 않아 세션 및 완료 이벤트에 메타데이터가 전혀 기록되지 않습니다.
또한 프론트엔드 UI에서는 답변의 근거 문서가 누락되었거나 신뢰도가 바닥(0%)인 경우에도 일반 지식베이스 기반 정상 답변과 동일한 형태로 표시되어 사용자가 모델의 환각(Hallucination) 또는 단순 일반 지식 기반 생성 여부를 구분하기 어렵습니다.

- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`): 하이브리드 검색 후 청크가 없거나 점수가 낮아도 검색 신뢰도 및 근거 유무 메타데이터를 산출하거나 방출하지 않고 단순 토큰 생성으로 직행합니다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`): 비에이전틱 질의 경로에서 `last_confidence`와 `last_missing`이 항상 `None`으로 유지되어 `done` 이벤트 페이로드 및 세션 턴 영속화에 신뢰도 데이터가 누락됩니다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`): AI 메시지 렌더링 시 신뢰도가 `undefined`이거나 출처 목록(`sources`)이 비어 있는 경우에도 근거 미참조에 대한 시각적 경고가 없어 사용자가 답변의 신뢰성을 판단할 수 없습니다.

## 요구사항
- [ ] `AskUseCase`에서 하이브리드 검색 결과 청크들의 유사도 점수를 기반으로 기본 검색 신뢰도(`confidence`)를 산출하고, 검색 결과가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]`을 전달한다.
- [ ] `AskRequestedConsumer`의 단순 질의 처리 경로에서 산출된 `confidence`와 `missing`을 `last_confidence`, `last_missing`에 바인딩하여 세션 턴 저장 및 `done` 이벤트 페이로드에 포함한다.
- [ ] 프론트엔드 `AiService.tsx`에서 AI 메시지의 `sources`가 없거나 빈 배열이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 표시한다.
- [ ] 단발성 질의의 신뢰도 수치가 완료 이벤트 및 세션 복원 시 신뢰도 배지(`confidence-badge`)로 화면에 올바르게 렌더링된다.
- [ ] `AiService.css`에 근거 부재 경고 배너/배지 스타일을 추가한다.
- [ ] 백엔드 신뢰도 산출 로직 및 프론트엔드 근거 미발견 경고 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 단발성 질의에 대해 무거운 LLM 비평(Critique) 프롬프트를 추가 실행하는 것은 응답 지연을 방지하기 위해 수행하지 않는다. (검색 유사도 점수 및 청크 존재 여부 기반 경량 산출로 한정)
- 검색 알고리즘이나 임베딩 모델 자체의 변경은 포함하지 않는다.
- 다중 사용자 협업 또는 채팅방 기능 수정은 포함하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`
  - 하이브리드 검색 결과 `chunks`가 존재하는 경우 최고 유사도 점수(`max(c.score)`) 또는 가중 평균 점수를 기반으로 0.0~1.0 범위의 신뢰도를 계산한다.
  - 검색된 청크가 없는 경우 `confidence = 0.0`, `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 반환 스트림 또는 콜백 형태로 전달한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
  - 단순 질의 스트리밍 과정에서 수집된 신뢰도 및 누락 정보를 `last_confidence`, `last_missing` 변수에 저장한다.
  - 스트리밍 완료 시 `session.append_turn` 및 `publish_done`에 해당 메타데이터를 전달한다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`
  - AI 메시지 버블에서 출처가 없거나 빈 배열이고 신뢰도가 0이거나 미참조 상태인 경우, 답변 상단에 경고 배너(`ungrounded-warning`)를 렌더링한다.
  - 단발성 질의 완료 시 수신된 `confidence` 값에 따라 기존 신뢰도 배지가 정상 출력되도록 연동한다.
- `public-front/src/components/AiService.css`
  - `.ungrounded-warning` 경고 박스 및 배지 스타일을 정의한다.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 전혀 없는 상태에서 사용자가 단순 RAG 질의를 전송했을 때
  When 백엔드가 검색 결과 0건으로 답변을 생성하여 스트리밍을 완료하면
  Then `done` 이벤트 페이로드에 `confidence: 0.0`과 `missing: ["관련 지식베이스 문서를 찾지 못함"]`이 포함되어 전달되고, 프론트엔드 AI 답변 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시된다.
- Given 지식베이스에서 높은 유사도(예: 0.85)로 문서 청크가 검색되어 답변이 생성되었을 때
  When 스트리밍이 완료되면
  Then 신뢰도 배지에 "신뢰도: 85%"와 `confidence-high` 스타일이 정상적으로 표시된다.
- Given 과거에 생성되어 신뢰도나 출처 메타데이터가 `undefined`인 레거시 세션 턴을 불러왔을 때
  When 세션이 복원되면
  Then UI가 깨지거나 오류 없이 기본 답변 텍스트로 안전하게 렌더링된다.

## 참고
- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
