---
id: SPEC-065
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 시스템에서 복합 질의(Agentic RAG)의 경우 비평 루프(`CritiqueGeneratorService`)를 거쳐 신뢰도(`confidence`)와 누락 정보(`missing`)를 계산하여 완료 이벤트 및 세션에 영속화하고 프론트엔드에 배지로 노출하고 있다.
그러나 단순 단발성 RAG 질의의 경우 다음과 같은 문제점이 존재한다.
- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색을 수행하지만, 검색된 청크의 유사도 점수를 기반으로 한 신뢰도 산출 및 검색 결과가 0건일 때의 누락 메타데이터 생성이 누락되어 있다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:138-155, 196-203` (`AskRequestedConsumer._process`)에서 단순 질의 처리 경로(`complexity != "complex"`)의 경우 `last_confidence`와 `last_missing`이 `None`으로 유지되어 `publish_done` 이벤트 페이로드와 `session.append_turn`에 신뢰도 데이터가 누락된다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서 AI 메시지의 `sources`가 없거나 빈 배열이고 신뢰도가 산출되지 않은 경우, 모델의 사전 학습 지식(일반 지식) 기반 답변인지 지식베이스 기반 답변인지 구분하는 시각적 안내가 제공되지 않아 사용자가 할루시네이션(환각) 위험을 인지하기 어렵다.

## 요구사항
- [ ] `AskUseCase`에서 하이브리드 검색 결과(`chunks`)가 존재할 때 검색된 청크들의 점수(최고 유사도 점수 `max(c.score)`)를 기반으로 검색 신뢰도(`confidence`)를 계산하여 전달한다.
- [ ] `AskUseCase`에서 검색 결과가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `AskRequestedConsumer._process`의 단순 질의 처리 경로에서 `AskUseCase`가 산출한 `confidence`와 `missing`을 `last_confidence` 및 `last_missing`에 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 반영한다.
- [ ] `AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)를 신뢰도 배지(`confidence-badge`, 0~100%)로 시각화하여 렌더링한다.
- [ ] `AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우, 해당 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 렌더링한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 단순 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.
- [ ] 백엔드 신뢰도 산출 로직 및 프론트엔드 근거 미발견 경고 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 복합 질의(Agentic RAG)의 기존 비평(Critique) 프롬프트 및 신뢰도 산출 알고리즘 수정
- 새로운 벡터 검색 알고리즘 및 임베딩 모델 교체
- 다중 사용자 협업 또는 채팅 웹소켓 메시지 전송 로직 수정

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`

## 프론트엔드
- `public-front/src/components/AiService.tsx`
- `public-front/src/components/AiService.css`

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 등록되어 있고 단발성 질의가 실행될 때, When 최고 유사도 점수가 0.85인 청크가 검색되면, Then 완료 이벤트 및 세션에 `confidence: 0.85`가 기록되고 프론트엔드에 `85%` 신뢰도 배지가 표시된다.
- Given 지식베이스에 질문과 일치하는 문서가 없을 때, When 단발성 질의가 실행되어 검색 청크가 0건이면, Then `confidence: 0.0` 및 `missing: ["관련 지식베이스 문서를 찾지 못함"]`이 기록되고 프론트엔드에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시된다.
- Given 근거 문서 없이 일반 지식으로 답변된 과거 세션을 불러올 때, When 세션 목록에서 해당 세션을 클릭하면, Then 복원된 AI 메시지 버블에 근거 미발견 경고 배너가 깨짐 없이 정상 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:138-155, 196-203` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
