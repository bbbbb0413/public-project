---
id: SPEC-011
title: 대화 세션 복원 시 출처 및 신뢰도 정보 복원
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 RAG 대화 세션 저장 및 복원 시, 사용자와 어시스턴트의 텍스트 대화 내용만 저장되고 참고한 출처 문서(`sources`) 및 신뢰도 평가 결과(`confidence`, `missing`)는 영속화되지 않는다.
`public-python-server/src/ai_service/rag/domain/model/conversation_session.py:12-27` 및 `53-84` 에서 `TurnRecord` 와 `ConversationSession.append_turn` 은 오직 `role`, `content`, `created_at` 만 관리하고 있어 RAG 응답의 메타데이터를 담을 수 없다.
또한 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-174` 에서 RAG 답변 스트리밍 완료 후 `session.append_turn(message.question, full_response)` 만 호출하여 생성 시점에 수집된 `sources` 가 세션에 전달되지 않고 유실된다.
`public-python-server/src/ai_service/rag/infrastructure/persistence/conversation_session_repository_impl.py:56-85` 와 `public-python-server/src/ai_service/rag/presentation/dto.py:9-15` 역시 턴 레코드의 텍스트 정보만 MongoDB에 직렬화/역직렬화하고 API 응답으로 반환한다.
이로 인해 프론트엔드 `public-front/src/components/AiService.tsx:113-132` 에서 과거 세션을 선택하여 `handleLoadSession` 을 실행할 때, `public-front/src/api/ai.ts:48-60` 의 `SessionTurn` 에 `sources`, `confidence`, `missing` 정보가 없어 기존 대화의 신뢰도 뱃지와 출처 문서 목록이 복원되지 않고 빈 화면으로 표시되는 불편이 있다.
따라서 세션 턴 모델과 저장소에 출처 목록 및 평가 메타데이터를 선택적 필드로 포함하고, 프론트엔드에서 세션 복원 시 이를 채팅 로그 및 출처 뷰에 반영할 수 있도록 개선해야 한다.

## 요구사항
- [ ] Python 백엔드의 `ConversationTurn` 값 객체 및 `TurnRecord` 모델에 `sources`, `confidence`, `missing` 선택적 필드를 추가한다
- [ ] `ask_requested_consumer.py` 에서 RAG 스트리밍 완료 후 세션에 어시스턴트 턴을 추가할 때 수집된 출처 목록(`sources`)을 함께 전달하여 저장한다
- [ ] MongoDB 세션 저장소(`conversation_session_repository_impl.py`)에서 턴의 `sources`, `confidence`, `missing` 필드를 안전하게 저장하고 복원한다
- [ ] 세션 상세 조회 DTO(`SessionDetailOut`, `TurnOut`) 및 프론트엔드 타입(`SessionTurn`)에 `sources`, `confidence`, `missing` 필드를 추가하여 클라이언트로 전달한다
- [ ] 프론트엔드 `AiService.tsx` 의 `handleLoadSession` 에서 세션 복원 시 각 메시지의 `confidence`, `missing` 을 `chatLog` 에 매핑하고, 마지막 어시스턴트 메시지의 `sources` 를 `currentSources` 에 복원한다
- [ ] 출처나 신뢰도 메타데이터가 없는 기존 세션 데이터(하위 호환성)를 불러와도 오류 없이 기본 텍스트 메시지로 정상 렌더링된다
- [ ] 변경된 도메인 모델, 영속화 매핑, 프론트엔드 세션 복원 동작에 대한 단위 테스트를 작성한다

## 비요구사항 (Out of scope)
- 게이트웨이(`public-server`) 수정: 게이트웨이 프록시 컨트롤러는 JSON 응답을 그대로 중계하므로 별도 스키마 수정 없이 동작한다
- 대화 세션 목록(`get_sessions`) API 페이로드 확장: 세션 목록에서는 제목과 수정일시만 보여주므로 상세 턴 메타데이터를 목록 API에 포함하지 않는다
- 복원된 과거 세션의 개별 턴별 독립적 출처 탭 UI 재구성: 현재 UI 구조에 맞추어 마지막 AI 턴의 출처 목록을 `currentSources` 에 연동하며 전체 레이아웃을 대규모 개편하지 않는다

## 백엔드
`public-python-server` 에서 다음 파일들을 수정한다.
- `src/ai_service/rag/domain/vo/conversation_turn.py`: `TurnValue` 및 `ConversationTurn` 에 `sources: list[dict[str, Any]] | None`, `confidence: float | None`, `missing: list[str] | None` 필드를 추가하고 팩토리 메서드(`of_assistant`) 및 `restore` 를 확장한다 (`public-python-server/src/ai_service/rag/domain/vo/conversation_turn.py:10-35`).
- `src/ai_service/rag/domain/model/conversation_session.py`: `TurnRecord` 및 `ConversationSession.append_turn` 에 어시스턴트 턴의 메타데이터(`sources`, `confidence`, `missing`) 파라미터를 추가한다 (`public-python-server/src/ai_service/rag/domain/model/conversation_session.py:12-27`).
- `src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`: 스트리밍 도중 파싱된 `sources` 를 보관하여 턴 저장 시 `session.append_turn(message.question, full_response, sources=sources)` 형태로 전달한다 (`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-174`).
- `src/ai_service/rag/infrastructure/persistence/conversation_session_repository_impl.py`: `_to_domain` 및 `_to_record` 에서 `sources`, `confidence`, `missing` 필드가 있을 때 MongoDB 문서와 상호 변환되도록 매핑을 추가한다 (`public-python-server/src/ai_service/rag/infrastructure/persistence/conversation_session_repository_impl.py:56-85`).
- `src/ai_service/rag/presentation/dto.py`: `TurnOut` DTO에 `sources`, `confidence`, `missing` 선택적 필드를 추가한다 (`public-python-server/src/ai_service/rag/presentation/dto.py:9-15`).

## 프론트엔드
`public-front` 에서 다음 파일들을 수정한다.
- `src/api/ai.ts`: `SessionTurn` 인터페이스에 `sources?: SourceRef[]`, `confidence?: number`, `missing?: string[]` 필드를 추가한다 (`public-front/src/api/ai.ts:48-54`).
- `src/components/AiService.tsx`: `handleLoadSession` 에서 세션 상세 데이터를 불러올 때 `detail.turns` 의 `confidence`, `missing` 을 `chatLog` 에 반영하고, 가장 마지막 AI 턴에 존재하는 `sources` 를 `setCurrentSources` 에 설정하여 화면 하단에 출처 목록이 나타나도록 구현한다 (`public-front/src/components/AiService.tsx:113-132`).

## 수용 기준 (Acceptance Criteria)
- Given 출처 문서 2개(`doc-1`, `doc-2`)가 포함된 RAG 답변이 세션에 저장된 상황 When 사용자가 세션 목록에서 해당 세션을 클릭하여 복원하면 Then 채팅 화면에 이전 대화 내용과 함께 신뢰도 뱃지 및 참고 문서 목록 2개가 정상 표시된다
- Given 과거 버전에서 생성되어 `sources` 나 `confidence` 필드가 없는 레거시 세션 데이터(경계 케이스) When 세션을 복원하면 Then 오류 발생 없이 텍스트 메시지만 정상 표시되고 출처 목록은 빈 상태로 유지된다
- Given 사용자가 질문만 입력하고 백엔드 처리가 중단되거나 출처 검색 결과가 0건인 대화 턴 When 세션을 저장 및 복원하면 Then `sources` 가 빈 배열 또는 `None` 으로 안전하게 처리된다
- Given 세션에 여러 턴의 질문과 답변이 누적된 상황 When 세부 조회를 수행하면 Then 각 어시스턴트 턴별로 독립적인 생성 시점의 메타데이터가 올바르게 반환된다

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/domain/model/conversation_session.py:12-27`
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/domain/vo/conversation_turn.py:10-35`
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-174`
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/persistence/conversation_session_repository_impl.py:56-85`
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/presentation/dto.py:9-15`
- 고쳐야 할 자리: `public-front/src/api/ai.ts:48-54`
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:113-132`
