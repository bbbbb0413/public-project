---
id: SPEC-059
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 지식베이스 기반 질의응답(RAG) 서비스에서 단발성(단순) 질의 처리 시 검색 신뢰도 산출 및 세션 영속화가 누락되어, 근거 문서가 없거나 부족한 상태에서 생성된 일반 모델 지식 기반 답변을 사용자가 신뢰도 높은 근거 기반 답변으로 오인할 위험이 있다.

1. `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서는 하이브리드 검색(`_hybrid_search.execute`)을 통해 문서 청크(`chunks`)를 조회하지만, 검색 결과 청크들의 관련도 점수(`score`)를 기반으로 한 신뢰도 계산을 수행하지 않으며 검색 결과가 0건인 경우에도 누락 사유(`missing`) 메타데이터를 반환하지 않는다.
2. `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-155` (`AskRequestedConsumer._process`)에서 단순 질의(`complexity != "complex"`) 경로 실행 시 `last_confidence`와 `last_missing`이 `None`으로 유지된 채 실행된다.
3. `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:185-203` (`AskRequestedConsumer._process`)에서 대화 턴 저장(`session.append_turn`) 및 완료 이벤트 발행(`publish_done`) 시 `last_confidence`가 `None`이어서 신뢰도 메타데이터가 누락된 채 전달된다.
4. `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서는 `confidence !== undefined`일 때만 신뢰도 배지를 표시하고, 지식베이스에서 근거 청크를 전혀 찾지 못해 모델의 일반 지식으로만 답변이 생성된 경우에 대한 시각적 경고 안내가 없다.

이로 인해 "답변을 믿을 수 있게 만든다"는 제품 방향에서 가장 빈번하게 발생하는 단순 질의 흐름에서 근거가 부족한 답과 충분한 답이 화면에서 구분되지 않는 문제가 발생한다.

## 요구사항
- [ ] `AskUseCase`에서 하이브리드 검색 청크가 존재할 때 최고 유사도 점수(`max(c.score)`)를 기반으로 검색 신뢰도(`confidence`)를 계산하고, 검색 결과가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `AskRequestedConsumer`의 단순 질의 처리 경로에서 `AskUseCase`가 산출한 `confidence`와 `missing`을 `last_confidence`, `last_missing`에 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`)에 반영한다.
- [ ] 프론트엔드 `AiService`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)가 신뢰도 배지(`confidence-badge`, 0~100%)로 정상 렌더링된다.
- [ ] 프론트엔드 `AiService`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우 메시지 상단에 근거 문서 미참조 경고 배너를 렌더링한다.
- [ ] 근거 부재 경고 배너/배지에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 문구와 경고 안내 스타일을 적용한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 단순 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.

## 비요구사항 (Out of scope)
- 에이전틱 RAG 루프(`AgenticAskUseCase`)의 다회차 비평 및 리파이닝 로직 수정은 포함하지 않는다.
- 새로운 검색 알고리즘이나 리랭커 가중치 수식의 변경은 포함하지 않는다.
- 게이트웨이(`public-server`)와 프론트엔드 간의 SSE 프로토콜 규격 변경은 하지 않으며, 기존 `done` 이벤트 메타데이터 형식을 그대로 재사용한다.

## 백엔드
`public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `AskUseCase.execute`에서 검색된 청크가 있을 때 최고 유사도 점수(`max(c.score)`)를 산출하고, 없을 때 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 정보를 반환 스트림 또는 메타데이터로 방출한다.

`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- `AskRequestedConsumer._process`에서 `AskUseCase`로부터 수신된 신뢰도 및 누락 메타데이터를 `last_confidence`, `last_missing` 변수에 할당한다.
- 수집된 신뢰도 정보를 `session.append_turn` 및 `publish_done`에 전달한다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- AI 메시지 버블 내에 `sources`가 비어있거나 `confidence === 0`일 때 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너(`ungrounded-warning`)를 조건부 렌더링한다.
- 단순 질의의 완료 이벤트 및 세션 복원 시 신뢰도 배지(`confidence-badge`)가 올바른 수치(예: 85%) 및 수준별 색상으로 표시되도록 한다.

`public-front/src/components/AiService.css`
- 근거 부재 경고 배너(`ungrounded-warning`)에 대한 시각적 강조 스타일(황색/주황색 경고 테두리, 배경 및 아이콘)을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 업로드된 문서와 관련 없는 질문을 단순 RAG 질의로 전송하여 검색 청크가 0건일 때
  When 답변 생성이 완료되면
  Then 신뢰도는 0%로 계산되고 AI 답변 버블에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 표시된다.
- Given 사용자가 업로드된 문서와 관련된 질문을 전송하여 유사도 점수 0.88의 청크가 검색되었을 때
  When 답변 생성이 완료되면
  Then AI 답변 버블에 신뢰도 88% 배지(`confidence-high`)가 녹색으로 정상 렌더링된다.
- Given 근거 문서 없이 생성되어 신뢰도 0%로 저장된 세션이 존재할 때
  When 사이드바에서 해당 세션을 클릭하여 복원하면
  Then 해당 AI 메시지 버블에 근거 부재 경고 배너와 신뢰도 0% 배지가 그대로 유지되어 복원된다.
- Given 에이전틱 복합 질의(`complexity === "complex"`)가 실행될 때
  When 비평 및 생성이 완료되면
  Then 기존 에이전틱 신뢰도 산출 및 진행 표시 기능이 회귀 없이 정상 동작한다.

## 참고
- 검색 신뢰도 산출 위치: `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 컨슈머 신뢰도 바인딩 및 영속화: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-155` (`AskRequestedConsumer._process`)
- 세션 턴 및 done 이벤트 발행: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:185-203` (`AskRequestedConsumer._process`)
- 프론트엔드 메시지 버블 및 신뢰도 렌더링: `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
