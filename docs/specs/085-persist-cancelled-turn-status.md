---
id: SPEC-085
title: 대화 세션 복원 시 답변 중단 상태(isCancelled) 영속화 및 미완성 답변 복원
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
`public-front/src/components/AiService.tsx:589-609` (`handleCancelStreaming`)에서는 사용자가 생성 중단 버튼을 클릭했을 때 로컬 메시지 객체에 `isCancelled: true` 플래그를 설정하여 `[생성 중단됨]` 배지 및 미완성 안내를 노출하고 `AnswerFeedback` 평가 폼을 숨기도록 개선되었다(SPEC-074).
그러나 백엔드 이벤트 소비자 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-206` (`AskRequestedConsumer._process`)에서는 스트리밍 중 Redis 취소 플래그(`job:{message.job_id}:cancelled`)를 감지하여 토큰 생성을 중단한 뒤에도, 취소 여부를 무시한 채 수집된 토큰만을 일반 정상 턴으로 세션 저장소(`session.append_turn`)에 영속화한다. 만약 첫 토큰이 수신되기 전에 중단된 경우(`not collected`)에는 세션에 턴 자체가 전혀 기록되지 않는다.
또한 백엔드 도메인 스키마 `public-python-server/src/ai_service/rag/schemas.py:102-111` (`TurnRecord`), `schemas.py:150-173` (`ConversationSession.restore`), `public-python-server/src/ai_service/rag/repository.py:99-121` (`ConversationSessionRepository._to_domain`) 및 `repository.py:124-140` (`ConversationSessionRepository._to_record`) 어디에도 턴의 취소/중단 상태(`is_cancelled`) 필드가 정의되어 있지 않다.
이로 인해 프론트엔드 API 계약 `public-front/src/api/ai.ts:298-306` (`SessionTurn`) 및 세션 복원 함수 `public-front/src/components/AiService.tsx:334-358` (`handleLoadSession`)에서 세션을 다시 불러올 때 중단 플래그가 유실되어, 중단되었던 미완성 답변이 일반 정상 완료 답변으로 둔갑한다. 그 결과 `[생성 중단됨]` 배지가 사라지고 미완성 텍스트에 대해 답변 평가(피드백) 폼이 다시 열려 왜곡된 평가가 수집되는 심각한 결함이 발생한다.

## 요구사항
- [ ] `public-python-server`의 `TurnValue`, `TurnRecord` 모델에 `is_cancelled: bool = False` 속성을 추가하고 `ConversationTurn.of_assistant` 및 `ConversationTurn.restore`에서 이를 다룬다.
- [ ] `ConversationSession.restore` 및 `append_turn` 메서드에 `is_cancelled: bool = False` 매개변수를 지원하여 어시스턴트 턴의 취소 상태를 보존한다.
- [ ] `ConversationSessionRepository._to_record` 및 `_to_domain`에서 `isCancelled` 불리언 필드를 안전하게 직렬화/역직렬화하고, 필드가 없는 기존 레거시 MongoDB 문서에 대해 기본값 `False`로 역호환성을 보장한다.
- [ ] `SessionTurnOut` Pydantic 스키마에 `is_cancelled: bool = Field(default=False, alias="isCancelled")` 필드를 추가하여 세션 상세 조회 API 응답으로 클라이언트에 전달한다.
- [ ] `AskRequestedConsumer._process`에서 Redis 취소 플래그(`job:{job_id}:cancelled`) 감지 시 `was_cancelled = True`를 기록하고, 세션 영속화 시 `session.append_turn`에 `is_cancelled=True`를 전달한다. 수신된 토큰이 없는 상태에서 취소된 경우에도 안내 문구(`"답변 생성이 중단되었습니다."`)와 함께 중단 턴을 세션에 영속화한다.
- [ ] `public-front/src/api/ai.ts`의 `SessionTurn` 인터페이스에 `isCancelled?: boolean` 선택적 필드를 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleLoadSession`에서 세션 복원 시 `isCancelled: t.isCancelled ?? false`를 `chatLog` 메시지 객체로 매핑한다.
- [ ] 세션을 새로고침하거나 대화 목록에서 다시 불러왔을 때 중단된 어시스턴트 메시지 버블에 `[생성 중단됨]` 배지가 유지되고 답변 평가 폼(`AnswerFeedback`)이 노출되지 않아야 한다.

## 비요구사항 (Out of scope)
- 사용자 질문 자체의 취소 또는 편집 기록 영속화
- 과거 세션에 저장된 정상 완료 답변을 사후에 중단 상태로 수동 변경하는 관리자 기능
- 중단된 답변에 대한 LLM 재개(이어쓰기) 백엔드 스트리밍 파이프라인 신설

## 백엔드
- `public-python-server/src/ai_service/rag/schemas.py`:
  - `TurnValue` 및 `TurnRecord`에 `is_cancelled: bool = False` 필드 추가
  - `ConversationTurn.of_assistant` 시그니처에 `is_cancelled: bool = False` 매개변수 추가
  - `ConversationTurn` 클래스에 `@property def is_cancelled(self) -> bool` 노출
  - `ConversationSession.restore` 및 `append_turn`에 `is_cancelled: bool = False` 매개변수 지원
  - `SessionTurnOut`에 `is_cancelled: bool = Field(default=False, alias="isCancelled")` 필드 추가 및 `from_domain` 매핑
- `public-python-server/src/ai_service/rag/repository.py`:
  - `ConversationSessionRepository._to_record`에서 `t.is_cancelled`가 참일 때 `turn_dict["isCancelled"] = True` 저장
  - `ConversationSessionRepository._to_domain`에서 `bool(t.get("isCancelled", False))`로 역직렬화
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - 스트리밍 토큰 루프에서 취소 감지 플래그(`was_cancelled: bool = False`) 관리
  - 취소 감지 후 세션 저장 시 수신된 토큰 유무와 관계없이 `is_cancelled=True`로 `session.append_turn` 호출하여 MongoDB에 보존

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `SessionTurn` 인터페이스에 `isCancelled?: boolean` 필드 선언
- `public-front/src/components/AiService.tsx`:
  - `handleLoadSession` 내부 `detail.turns.map` 매핑 로직에 `isCancelled: t.isCancelled ?? false` 반영
  - 세션 복원 시 중단된 메시지가 `msg.isCancelled === true` 분기를 타도록 보장

## 수용 기준 (Acceptance Criteria)
- Given 중단 플래그가 없는 기존 대화 세션 데이터가 MongoDB에 저장되어 있을 때
  When `GET /ai/rag/sessions/:sessionId`로 세션 상세를 조회하면
  Then 모든 턴의 `isCancelled` 필드가 `false`로 기본 반환되고 오류 없이 정상 응답한다.
- Given RAG 스트리밍 생성 도중 취소 요청이 발생하여 중단된 대화 세션이 있을 때
  When 세션을 새로고침하거나 대화 사이드바에서 해당 세션을 다시 불러오면(`handleLoadSession`)
  Then 중단된 어시스턴트 메시지 버블에 `[생성 중단됨]` 배지가 표시되고 답변 평가 폼(`AnswerFeedback`)이 렌더링되지 않는다.
- Given 질문 전송 직후 첫 토큰이 생성되기 전에 스트리밍이 즉시 중단된 세션이 있을 때
  When 해당 세션을 다시 불러오면
  Then 어시스턴트 메시지에 `"답변 생성이 중단되었습니다."` 문구와 함께 `[생성 중단됨]` 배지가 정상 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/schemas.py:102-111` (`TurnRecord`), `schemas.py:150-173` (`ConversationSession.restore`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-206` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:99-140` (`ConversationSessionRepository._to_domain`, `_to_record`)
- 고쳐야 할 자리: `public-front/src/api/ai.ts:298-306` (`SessionTurn`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:334-358` (`handleLoadSession`)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:589-609` (`handleCancelStreaming`), `AiService.tsx:1198-1249`
