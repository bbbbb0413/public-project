---
id: SPEC-057
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 질의응답 시스템은 에이전틱 질의(`complexity == "complex"`)에 대해서만 비평(Critique) 단계를 거쳐 신뢰도(`confidence`)와 누락 항목(`missing`)을 산출하고 완료 이벤트(`done`) 및 대화 세션에 영속화하고 있다.
반면 대다수의 일반 질문이 거치는 단발성 비에이전틱 질의(`AskUseCase`)의 경우, `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색을 수행하여 청크가 검색되더라도 검색 유사도 기반의 신뢰도를 전혀 산출하지 않으며, 청크가 0건 검색된 경우에도 빈 출처 상태로 LLM 생성을 그대로 진행한다.
이로 인해 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-155` (`AskRequestedConsumer._process`)에서 단발성 질의의 `last_confidence` 및 `last_missing`은 항상 `None`으로 유지되어 완료 이벤트(`publish_done`)에 신뢰도 정보가 누락되고 세션 턴(`session.append_turn`)에도 신뢰도가 저장되지 않는다.
결과적으로 프론트엔드 `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서 SPEC-010 및 SPEC-018을 통해 구현된 신뢰도 배지가 일반 단발성 질문에서는 전혀 렌더링되지 않으며, 검색된 근거 문서가 전혀 없는 상태에서 생성된 모델 일반 지식 기반 답변임에도 사용자에게 아무런 경고나 알림이 제공되지 않아 답변의 출처와 신뢰성을 오인하게 만드는 문제가 발생한다.

## 요구사항
- [ ] `AskUseCase.execute`에서 하이브리드 검색 결과(`chunks`)가 존재할 때 검색된 청크들의 점수(최고 유사도 점수 `max(c.score)`)를 기반으로 기본 검색 신뢰도(`confidence`)를 계산하고, 청크가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 반환 또는 방출하도록 지원한다.
- [ ] `AskRequestedConsumer._process`의 단순 질의 처리 경로에서 산출된 `confidence`와 `missing`을 `last_confidence` 및 `last_missing`에 정상 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 포함한다.
- [ ] 프론트엔드 `AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우, 해당 메시지 버블 상단 또는 하단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] 프론트엔드 `AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)가 신뢰도 배지(`confidence-badge`, 0~100%)로 정상 렌더링된다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 단순 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.
- [ ] `AiService.css`에 근거 부재 경고 배지/배너 스타일(`ungrounded-warning`)을 추가한다.

## 비요구사항 (Out of scope)
- 단발성 질의에 에이전틱 다회차 비평/개선 루프를 강제로 도입하거나 추가 LLM 호출을 발생시키는 작업
- RAG 하이브리드 검색 알고리즘 및 임베딩 모델 자체의 변경이나 재학습
- RAGAS 평가 패널의 지표 계산 로직 변경

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `AskUseCase.execute` 실행 시 하이브리드 검색 청크가 있으면 최고 청크 유사도 점수를 기반으로 `confidence`를 도출하고, 청크가 없으면 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]`을 반환/전달한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- `AskRequestedConsumer._process`에서 `AskUseCase`의 신뢰도 및 누락 메타데이터를 `last_confidence`, `last_missing` 변수에 할당하여 `session.append_turn`과 `publish_done`에 전달한다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`
- AI 메시지 버블 렌더링 영역(`chat-message ai`)에서 `sources`가 비어있고 `confidence === 0`이거나 `missing`에 문서 미발견 관련 안내가 있을 때 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 요소를 렌더링한다.
- 단발성 질의 완료 시 `onDone` 페이로드의 `confidence`를 반영하여 신뢰도 배지가 정상 출력되도록 한다.
- `public-front/src/components/AiService.css`
- 근거 미발견 경고 배너(`ungrounded-warning`)에 대한 시각적 경고 스타일(노란색/주황색 계열 테두리 및 배경)을 정의한다.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 등록되어 있고 단발성 질의를 수행했을 때
  When 검색 결과 청크가 1건 이상 반환되어 답변 생성이 완료되면
  Then 완료 이벤트 및 메시지 버블에 산출된 신뢰도 배지(예: 85%)가 표시되고 근거 미참조 경고 배너는 노출되지 않아야 한다.
- Given 지식베이스에 관련 문서가 전혀 없거나 검색 결과가 0건일 때
  When 단발성 질의를 수행하여 답변 생성이 완료되면
  Then 신뢰도는 0%로 계산되고, 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시되어야 한다.
- Given 과거 대화 세션에 근거 문서 없이 생성된 답변 턴이 저장되어 있을 때
  When 해당 세션을 사이드바에서 선택하여 복원(`handleLoadSession`)하면
  Then 해당 AI 메시지 버블에 신뢰도 0% 배지와 근거 미참조 경고 배너가 정상 렌더링되어야 한다.

## 참고
- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-155` (`AskRequestedConsumer._process`)
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
