---
id: SPEC-066
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 복합 질의(Agentic RAG)의 경우 비평(Critique) 루프를 거치며 신뢰도(`confidence`)와 누락 항목(`missing`)을 산출하여 이벤트와 세션에 저장하고 화면에 배지로 표시한다.
반면 단발성 RAG 질의(`complexity != "complex"`)의 경우 `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색 청크(`chunks`)를 기반으로 출처(`__SOURCES`)만 방출할 뿐 신뢰도와 누락 정보를 일절 산출하지 않는다.
이로 인해 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 187-203` (`AskRequestedConsumer._process`)에서 `last_confidence`와 `last_missing`이 계속 `None`으로 남아 완료 이벤트(`done`) 및 세션 저장소(`session.append_turn`)에 신뢰도 데이터가 누락된다.
또한 `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서는 AI 답변 메시지에 신뢰도 배지가 누락될 뿐만 아니라, 검색 결과가 없어 지식베이스 문서 참조 없이 LLM 자체 일반 지식으로만 답변을 생성한 경우에도 "근거 문서 미참조" 경고 배너나 배지가 전혀 표시되지 않아 사용자가 답변의 근거 신뢰성을 스스로 검증할 수 없는 문제가 있다.

## 요구사항
- [ ] `AskUseCase`에서 하이브리드 검색 결과(`chunks`)가 존재할 때 검색된 청크들의 점수(최고 유사도 점수 `max(c.score)`)를 기반으로 기본 검색 신뢰도(`confidence`)를 계산하여 전달한다.
- [ ] `AskUseCase`에서 검색 결과가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `AskRequestedConsumer._process`의 단순 질의 처리 경로에서 `AskUseCase`가 산출한 `confidence`와 `missing`을 `last_confidence` 및 `last_missing`에 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 반영한다.
- [ ] `AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)를 신뢰도 배지(`confidence-badge`, 0~100%)로 시각화하여 렌더링한다.
- [ ] `AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우, 해당 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 단순 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.

## 비요구사항 (Out of scope)
- 단발성 질의에 비평 LLM(Critique) 호출 루프를 추가하는 것은 처리 지연 및 비용을 유발하므로 수행하지 않는다. (검색된 청크의 점수 기반 계산만 적용)
- 하이브리드 검색 알고리즘 및 RRF 가중치 계산식을 변경하지 않는다.
- 기존 복합 에이전틱 질의(`AgenticAskUseCase`)의 비평 및 신뢰도 산출 로직은 수정하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py` (`AskUseCase.execute`): 검색 결과 청크 점수를 바탕으로 신뢰도 계산 및 메타데이터 이벤트 방출 또는 전달.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` (`AskRequestedConsumer._process`): 단순 질의 경로에서 신뢰도 및 누락 메타데이터 수집, 세션 턴 저장 및 `done` 이벤트 발행 페이로드 연동.

## 프론트엔드
- `public-front/src/components/AiService.tsx` (`AiService`): AI 메시지 버블 내 단발성 질의 신뢰도 배지 렌더링 및 근거 문서 미참조 경고 배너 렌더링.
- `public-front/src/components/AiService.css`: 근거 부재 경고 배너/배지 스타일(`ungrounded-warning` 등) 추가.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 존재하여 검색 결과 청크가 반환된 경우, When 사용자가 단순 질문을 전송하고 생성이 완료되면, Then 답변 버블 하단에 산출된 신뢰도 배지(예: 신뢰도 85%)가 표시된다.
- Given 지식베이스에 일치하는 문서가 없어 검색 청크가 0건인 경우, When 사용자가 질문을 전송하고 답변 생성이 완료되면, Then 신뢰도 0% 배지와 함께 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 메시지 버블에 표시된다.
- Given 근거 문서 미참조 경고가 포함된 대화 세션이 저장된 상태에서, When 세션 목록에서 해당 세션을 클릭하여 복원하면, Then 이전 답변 버블에 신뢰도 배지와 근거 미참조 경고 배너가 동일하게 유지되어 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 187-203` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:78-90` (`AgenticAskUseCase.execute`)
