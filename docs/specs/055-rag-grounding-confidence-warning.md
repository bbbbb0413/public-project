---
id: SPEC-055
title: 단발성 RAG 질의 신뢰도 산출 및 근거 문서 미참조 답변 경고 표시
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 복합(에이전틱) 질의(`AgenticAskUseCase`)는 비평(Critique) 루프를 통해 산출된 신뢰도(`confidence`)와 누락 항목(`missing`)을 클라이언트에 전달하고 대화 세션에 영속화하지만, 단순·단발성 RAG 질의(`AskUseCase`)는 신뢰도와 누락 정보 산출 로직이 누락되어 있어 완료 이벤트 및 대화 세션에 신뢰도 데이터가 `None`으로 기록된다.

- `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)에서 하이브리드 검색(`_hybrid_search.execute`) 결과 청크(`chunks`)가 존재할 때 출처 정보(`__SOURCES`)만 방출하고, 검색 유사도 점수를 기반으로 한 신뢰도 계산 및 청크가 0건일 때의 누락 메타데이터 산출 로직이 없다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 142-154, 196-203` (`AskRequestedConsumer._process`)에서 `complexity != "complex"`인 단순 질문 경로 실행 시 `last_confidence`와 `last_missing`이 `None`으로 유지되어, `:187-193`의 세션 턴 영속화(`session.append_turn`) 및 `:196-203`의 완료 이벤트(`publish_done`)에 신뢰도 데이터가 담기지 않고 유실된다.
- `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)에서 AI 메시지 버블 렌더링 시 `msg.confidence`가 `undefined`이면 신뢰도 배지가 표시되지 않으며, 검색된 출처가 없거나 신뢰도가 0점인 경우에도 경고 없이 일반 답변과 동일하게 노출되어 사용자가 모델의 일반 지식 기반 생성 답변인지 지식베이스 기반 답변인지 식별할 수 없다.

이는 "근거가 부족한 답과 충분한 답이 화면에서 구분되는가"라는 이번 분기 핵심 제품 방향에 직결되는 결함이다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`에서 하이브리드 검색 결과 청크가 존재할 때 최고 유사도 점수(`max(c.score)`)를 기반으로 기본 검색 신뢰도(`confidence`)를 계산하고, 청크가 0건인 경우 `confidence = 0.0` 및 `missing = ["관련 지식베이스 문서를 찾지 못함"]` 메타데이터를 전달한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`의 단순 질의 처리 경로에서 산출된 `confidence`와 `missing`을 `last_confidence`, `last_missing`에 바인딩하여 세션 턴 저장(`session.append_turn`) 및 완료 이벤트(`publish_done`) 페이로드에 반영한다.
- [ ] `public-front/src/components/AiService.tsx`에서 AI 메시지의 출처(`sources`)가 없거나 빈 배열(`[]`)이고 `confidence === 0` 또는 `missing`에 문서 미발견 안내가 존재하는 경우 메시지 상단/하단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너/배지를 시각적으로 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`에서 단순 RAG 질의 완료 시 전달받은 신뢰도(0~1 수치)가 신뢰도 배지(`confidence-badge`, 0~100%)로 정상 렌더링된다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 단순 질의 메시지의 신뢰도 배지 및 근거 미참조 경고 배너가 정상 렌더링된다.
- [ ] `public-front/src/components/AiService.css`에 근거 부재 경고 배지/배너 스타일(`ungrounded-warning`)을 추가한다.
- [ ] 백엔드 신뢰도 산출 로직 및 프론트엔드 근거 미발견 경고 배너 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 하이브리드 검색 알고리즘(`HybridSearchUseCase`) 및 임베딩 모델 자체의 교체나 가중치 수정.
- 에이전틱 RAG(`AgenticAskUseCase`)의 비평 및 리파이닝 루프 알고리즘 변경.
- 대화 세션 DB 스키마 마이그레이션 (기존 `TurnRecord` 스키마에 이미 `confidence`, `missing`, `sources` 필드가 지원됨).

## 백엔드
- `public-python-server/src/ai_service/rag/application/ask_use_case.py`:
  - `AskUseCase.execute`에서 검색된 `chunks`가 있을 경우 `max([c.score for c in chunks])`를 계산하여 `__CONFIDENCE:{"confidence": ..., "missing": []}` 형태의 메타데이터 방출 또는 반환 규격 확장.
  - `chunks`가 비어있는 경우 `__CONFIDENCE:{"confidence": 0.0, "missing": ["관련 지식베이스 문서를 찾지 못함"]}` 방출.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `AskUseCase` 스트림에서 전달된 신뢰도/누락 메타데이터를 파싱하여 `last_confidence` 및 `last_missing`에 저장.
  - `session.append_turn` 및 `publish_done`에 신뢰도와 누락 정보 전달.
- `public-python-server/tests/unit/rag/test_ask_use_case.py`:
  - 검색 청크가 있을 때와 청크가 0건일 때의 신뢰도 및 누락 메타데이터 방출 검증 단위 테스트 추가.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `chatLog`의 AI 메시지 렌더링 영역(`chat-message ai`)에서 `msg.confidence === 0`이거나 `msg.missing`에 문서 미발견 안내가 있는 경우 근거 미발견 경고 배너(`ungrounded-warning-banner`) 표시.
  - 단발성 질의 완료 시 `onDone` 콜백에서 받은 `confidence`와 `missing`을 `chatLog` AI 메시지에 반영.
- `public-front/src/components/AiService.css`:
  - `.ungrounded-warning-banner` 클래스 스타일 정의 (경고 테마 색상 및 안내 아이콘/텍스트).
- `public-front/src/components/AiService.test.tsx`:
  - 근거 문서가 없거나 신뢰도가 0인 AI 메시지에 대해 근거 미발견 경고 배너가 렌더링되는지 검증하는 단위 테스트 추가.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 등록된 지식베이스 문서와 관련된 질문을 단순 질의로 요청했을 때
  When 백엔드가 검색 청크(최고 유사도 0.85)를 기반으로 답변을 생성하고 완료 이벤트를 발행하면
  Then 프론트엔드 AI 메시지 버블에 신뢰도 85% 녹색 배지(`confidence-high`)가 정상 렌더링된다.
- Given 지식베이스에 아무런 문서가 없거나 질문과 일치하는 검색 청크가 0건일 때 (경계 케이스)
  When 단순 RAG 질문을 전송하여 답변이 완료되면
  Then 백엔드는 `confidence: 0.0` 및 `missing: ["관련 지식베이스 문서를 찾지 못함"]`을 발행하고, 프론트엔드 AI 메시지 상단에 "근거 문서 미발견 (모델 일반 지식 기반 답변)" 경고 배너가 렌더링된다.
- Given 과거에 단발성 질의로 저장된 대화 세션을 사이드바에서 선택하여 불러왔을 때
  When 세션 상세 데이터가 로드되면
  Then 각 AI 답변 메시지에 저장된 신뢰도 배지 및 근거 미참조 경고 배너가 원래 상태대로 복원되어 렌더링된다.

## 참고
- 고쳐야 할 자리 (단발성 질의 실행 및 출처 방출): `public-python-server/src/ai_service/rag/application/ask_use_case.py:90-108` (`AskUseCase.execute`)
- 고쳐야 할 자리 (단발성 질의 컨슈머 메타데이터 처리): `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:114-115, 196-203` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리 (신뢰도 배지 및 메시지 버블 렌더링): `public-front/src/components/AiService.tsx:1104-1132` (`AiService`)
- 관련 정의 (에이전틱 신뢰도 산출 참조): `public-python-server/src/ai_service/rag/application/critique_generator_service.py:100-112` (`CritiqueGeneratorService._parse_critique`)
