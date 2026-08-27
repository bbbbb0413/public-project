---
id: SPEC-018
title: 대화 세션 영속화 시 신뢰도 메타데이터 저장 누락 수정 및 신뢰도 수준별 배지 시각화
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
이전 작업(SPEC-010, SPEC-011)을 통해 RAG 답변의 신뢰도(`confidence`) 및 누락 정보(`missing`) 전달 체계와 세션 복원 DTO를 구축하였으나, `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-169` (`_process`)에서 답변 생성 완료 후 `session.append_turn` 호출 시 `sources` 만 전달하고 수집된 `confidence` 와 `missing` 인자를 전달하지 않아 세션 DB에 신뢰도 데이터가 누락되는 결함이 존재한다.
세션 모델인 `public-python-server/src/ai_service/rag/schemas.py:162-170` (`ConversationSession.append_turn`)은 `confidence` 및 `missing` 파라미터를 지원하도록 정의되어 있으나, 호출부에서 값을 누락하여 기존 대화 세션을 다시 열었을 때 신뢰도 정보가 복원되지 못하고 소실된다.
또한 프론트엔드 `public-front/src/components/AiService.tsx:640-645` (`AiService`) 및 `public-front/src/components/AiService.css:762-783` (`.confidence-badge`, `.confidence-value`)에서는 신뢰도 수치와 무관하게 고정된 단일 녹색 계열(`#34d399`) 스타일로만 배지를 렌더링한다. 이로 인해 신뢰도가 매우 낮은 답변(예: 30%)도 높은 신뢰도의 답변과 동일한 긍정적 색상으로 표시되어, 사용자가 부정확하거나 근거가 부족한 답변을 직관적으로 식별하기 어렵다.

## 요구사항
- [ ] `public-python-server` 의 `ask_requested_consumer.py` 에서 RAG 스트리밍 완료 후 세션에 턴을 추가할 때 수집된 `last_confidence` 와 `last_missing` 을 `session.append_turn` 에 함께 전달하여 세션 저장소에 영속화한다.
- [ ] 단발성 질의이거나 신뢰도 평가 결과가 없는 경우(`last_confidence` 또는 `last_missing` 이 `None`), `None` 값이 안전하게 전달되어 오류 없이 세션이 저장된다.
- [ ] 프론트엔드 `public-front/src/components/AiService.tsx` 에서 신뢰도 수치에 따라 신뢰도 배지에 수준별 CSS 클래스(`confidence-high`, `confidence-medium`, `confidence-low`)를 부여한다. (80% 이상: high, 60% 이상 80% 미만: medium, 60% 미만: low)
- [ ] 프론트엔드 `public-front/src/components/AiService.css` 에 신뢰도 수준별 색상 테마(high: 녹색, medium: 노랑/황색, low: 주황/적색) 스타일을 추가한다.
- [ ] 세션 목록에서 세션을 불러왔을 때 각 AI 메시지 버블에 영속화된 신뢰도와 확인하지 못한 항목이 올바른 수준별 배지 색상과 함께 화면에 렌더링된다.
- [ ] 백엔드 컨슈머의 세션 메타데이터 영속화 및 프론트엔드 신뢰도 수준별 배지 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- RAG 추론/비평 루프(`agentic_ask_use_case.py`) 내부의 신뢰도 계산 산식 및 임계값 변경
- 세션 단위 외 별도의 사용자 피드백(좋아요/싫어요) 수집 API 추가
- 결제/인증 등 RAG 외 도메인의 상태 변경 및 마이그레이션

## 백엔드
`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- `_process` 메서드에서 스트리밍 종료 후 세션 턴을 기록하는 지점(`session.append_turn`)에 `confidence=last_confidence`, `missing=last_missing` 인자를 전달하도록 수정한다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- AI 메시지 버블 렌더링 시 `msg.confidence` 수치를 판별하여 80% 이상은 `confidence-high`, 60% 이상 80% 미만은 `confidence-medium`, 60% 미만은 `confidence-low` 클래스를 `confidence-badge` 엘리먼트에 추가한다.

`public-front/src/components/AiService.css`
- `.confidence-badge.confidence-high`, `.confidence-badge.confidence-medium`, `.confidence-badge.confidence-low` 및 각 수준별 `.confidence-value` 색상 스타일을 정의한다.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 질의가 실행되어 신뢰도가 85%(0.85)로 계산되고 세션이 연결되어 있을 때
  When 답변 스트리밍이 완료되고 세션 저장이 완료되면
  Then MongoDB 세션 저장소의 마지막 턴 레코드에 `confidence: 0.85` 가 정상 영속화된다.
- Given 세션에 `confidence: 0.45` (45%)인 이전 대화가 저장되어 있을 때
  When 사용자가 해당 세션을 불러오면
  Then AI 답변 버블의 신뢰도 배지에 `confidence-low` 클래스가 적용되어 주황/적색 계열로 렌더링된다.
- Given 세션에 `confidence: 0.70` (70%)인 이전 대화가 저장되어 있을 때
  When 사용자가 해당 세션을 불러오면
  Then AI 답변 버블의 신뢰도 배지에 `confidence-medium` 클래스가 적용되어 노랑/황색 계열로 렌더링된다.
- Given `confidence` 필드가 없는 레거시 세션 턴(`undefined` 또는 `None`)을 불러올 때 (경계 케이스)
  When 화면에 렌더링되면
  Then 신뢰도 배지가 노출되지 않고 답변 텍스트만 정상 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-169` (`append_turn` 호출 누락)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:640-645` (`confidence-badge` 렌더링 로직)
- 고쳐야 할 자리: `public-front/src/components/AiService.css:762-783` (신뢰도 배지 스타일 정의)
- 세션 도메인 모델 정의: `public-python-server/src/ai_service/rag/schemas.py:162-170` (`ConversationSession.append_turn`)
