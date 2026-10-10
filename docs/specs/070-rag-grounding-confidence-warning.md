---
id: SPEC-070
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
이 제품의 이번 분기 핵심 목표는 "답변을 믿을 수 있게 만든다"이며, 근거가 부족한 답과 충분한 답이 화면에서 명확히 구분되어야 한다.
지금까지 신뢰도 전달(SPEC-010), 세션 복원 시 신뢰도 복원(SPEC-011), 신뢰도 수준별 배지 시각화(SPEC-018) 기능이 구현되었으나, 이는 복합 질의(에이전틱 RAG) 경로에서만 산출되고 있다.
실제 사용자가 가장 빈번하게 수행하는 기본 단발성 RAG 질의에서는 다음과 같은 결함과 사용성 문제가 존재한다:
- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색 청크(`chunks`)를 기반으로 답변을 생성할 때, 검색 결과 청크들의 유사도 점수를 활용한 기본 신뢰도 및 누락 사유 메타데이터를 산출하거나 방출하지 않는다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-150` (`AskRequestedConsumer._process`) 및 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:185-204` (`AskRequestedConsumer._process`)에서 비에이전틱 단순 질의 경로 실행 시 `last_confidence`와 `last_missing`이 `None`으로 유지되어 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`)에 신뢰도 데이터가 누락된다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서 AI 메시지의 출처(`sources`)가 없거나 신뢰도가 0일 때, 지식베이스 문서를 근거로 생성된 답변인지 모델의 사전 학습 일반 지식에만 의존한 답변인지 구분해 주는 경고 표시가 전혀 없다.
- 이로 인해 지식베이스에 관련 문서가 전혀 없거나 검색 점수가 낮은 상태에서 생성된 환각 위험 답변을 사용자가 신뢰도 높은 근거 기반 답변으로 오인하게 되는 심각한 문제가 발생한다.

## 요구사항
- [ ] `public-python-server`의 `AskUseCase.execute`에서 하이브리드 검색 결과 청크(`chunks`)가 존재할 경우 청크들의 최고 유사도 점수(`max(c.score)`)를 기반으로 검색 신뢰도(`confidence`)를 계산하여 메타데이터로 전달한다.
- [ ] `public-python-server`의 `AskUseCase.execute`에서 검색 결과 청크가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `public-python-server`의 `AskRequestedConsumer._process`의 단순 질의 처리 경로에서 `AskUseCase`가 산출한 `confidence`와 `missing`을 `last_confidence` 및 `last_missing`에 정상 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 반영한다.
- [ ] `public-front/src/components/AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)를 신뢰도 배지(`confidence-badge`, 0~100%)로 시각화하여 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우, 해당 메시지 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] 대화 세션을 다시 불러왔을 때(`handleLoadSession`)에도 영속화된 단발성 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 복원되어 렌더링된다.

## 비요구사항 (Out of scope)
- 하이브리드 검색 알고리즘이나 RRF(Reciprocal Rank Fusion) 가중치 자체를 변경하는 작업
- 에이전틱 RAG 루프 비평 모델(Critique) 프롬프트 및 신뢰도 산출 수식 변경
- 지식베이스 자동 문서 재색인 또는 외부 웹 검색 연동 기능 추가
- 신뢰도 점수 기반 답변 자동 재생성 루프(단발성 질의는 1회 생성 원칙 유지)

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`:
  - 하이브리드 검색 결과(`chunks`)에 대해 검색 신뢰도 계산 (청크 존재 시 최고 점수 기반 정규화된 0.0~1.0 부동소수점, 청크 0건 시 0.0)
  - `__CONFIDENCE:{"confidence": ..., "missing": ...}` 형태의 스트림 제어 이벤트 방출 또는 콜백 지원
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - 단순 질의 스트림 소비 중 신뢰도 메타데이터 감지 및 `last_confidence`, `last_missing` 업데이트
  - `publish_done` 호출 시 `done_data`에 신뢰도 및 누락 목록 포함
  - `session.append_turn` 호출 시 `confidence`와 `missing` 파라미터 전달

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - AI 메시지 버블 내부에서 `msg.confidence` 값이 존재할 때 신뢰도 배지(`confidence-badge`) 렌더링
  - `(!msg.sources || msg.sources.length === 0) && (msg.confidence === 0 || (msg.missing && msg.missing.length > 0))` 조건에 부합할 때 메시지 상단에 경고 배너(`ungrounded-warning`) 렌더링
  - 경고 배너 텍스트: "근거 문서 미발견 (모델 일반 지식 기반 답변)"
- `public-front/src/components/AiService.css`:
  - `.ungrounded-warning` 경고 배너/배지 스타일 추가 (황색/주황색 계열 배경, 경고 아이콘 기호, 둥근 테두리)
- `public-front/src/components/AiService.test.tsx`:
  - 단발성 질의 응답 신뢰도 배지 표시 및 근거 미발견 경고 배너 렌더링 검증 단위 테스트 추가

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 존재하여 검색 청크(점수 0.85)가 발견된 상태에서
  When 사용자가 단발성 질문을 전송하고 생성이 완료되면
  Then AI 답변 버블에 신뢰도 85% 배지가 `confidence-high` 클래스와 함께 녹색으로 표시되어야 한다.
- Given 지식베이스에 질문과 일치하는 문서가 없어 검색 결과가 0건인 상태에서
  When 사용자가 단발성 질문을 전송하고 답변이 생성되면
  Then AI 답변 버블 상단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시되고 신뢰도 0% 배지가 렌더링되어야 한다.
- Given 근거 문서 미참조 경고가 포함된 단발성 질의가 완료된 대화 세션에서
  When 사용자가 사이드바에서 해당 세션을 다시 불러오면(`handleLoadSession`)
  Then 복원된 AI 메시지 버블에 동일하게 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너와 신뢰도 배지가 표시되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-150` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:185-204` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
- 스타일 참고: `public-front/src/components/AiService.css:120-160`
