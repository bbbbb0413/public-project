---
id: SPEC-102
title: 보관함 답변 클릭 시 해당 턴 위치 이동·강조 표시 및 보관 항목 삭제 확인 대화상자 추가
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품은 올려 둔 문서를 근거로 질문에 답하고, 사용자가 검증한 신뢰할 수 있는 답변을 스스로 확인하고 활용할 수 있도록 돕는 도구다. 이를 위해 답변 북마크 API(SPEC-082), 세션 북마크 토글(SPEC-080), 답변 보관 메모 모달(SPEC-090) 등이 단계적으로 배송되었다. 그러나 사용자가 보관한 답변을 사이드바 보관함에서 다시 열람하고 관리하는 핵심 흐름에서 두 가지 중대한 사용성 및 안전장치 결함이 존재한다.
첫째, `public-front/src/components/AiService.tsx:1384-1414` (`AiService`)을 보면 사이드바 보관함(`bookmarks`) 목록에서 답변 북마크 아이템(`b`)을 클릭했을 때 단순히 `handleLoadSession(b.sessionId)`만 호출하고 있다. `public-front/src/api/ai.ts:288-296` (`BookmarkItem`)에 명시된 대로 북마크 객체는 어떤 턴을 보관했는지 나타내는 `turnIndex?: number` 필드를 정상적으로 보유하고 있으나, `public-front/src/components/AiService.tsx:457-485` (`handleLoadSession`)에 턴 위치 정보가 전달되지 않는다. 이로 인해 여러 턴이 누적된 긴 대화 세션에서 특정 답변을 보관해 두었더라도 보관함을 통해 대화를 열면 첫 턴부터 화면에 표시되어, 사용자가 보관했던 답변이 어디 있는지 대화창 전체를 수동으로 스크롤하며 대조해 찾아야 하는 큰 불편이 발생한다.
둘째, `public-front/src/components/AiService.tsx:336-345` (`handleDeleteBookmark`)을 보면 보관함 아이템 삭제 버튼(`btn-delete-session` `✕`) 클릭 시 아무런 확인 대화상자 없이 즉시 `deleteBookmark(bookmarkId)` API를 호출하여 영구 삭제를 실행한다. 지식베이스 문서 삭제 확인(SPEC-013) 및 대화 세션 삭제 확인(SPEC-058)에서는 되돌릴 수 없는 영구 삭제 동작에 확인 모달을 두어 안전장치를 마련했으나, 정성껏 메모까지 남겨둔 보관 항목 삭제에는 안전장치가 누락되어 있다. 이로 인해 사용자가 실수로 삭제 버튼을 누르는 즉시 복구할 수 없는 데이터 유실이 발생한다.
셋째, `public-front/src/components/AiService.tsx:1384-1414` (`AiService`)의 보관함 렌더링부에서는 답변 단위 북마크(`b.turnIndex !== undefined`)와 세션 단위 북마크(`b.turnIndex === undefined`) 간의 시각적 구분이 전혀 없다. 사용자는 목록에서 어떤 항목이 특정 답변을 보관한 것인지, 세션 전체를 보관한 것인지 확인하기 어렵다.
따라서 보관함에서 항목 클릭 시 해당 답변 턴으로 자동 스크롤 및 일시적 하이라이트 애니메이션을 제공하고, 삭제 시 확인 대화상자 안전장치를 추가하며, 보관 유형별 배지를 노출하여 답변 보관 경험의 신뢰성과 편의성을 완성해야 한다.

## 요구사항
- [ ] 사이드바 보관함(`bookmarks`) 탭의 북마크 아이템 클릭 시, 대상 세션을 불러온 뒤 해당 답변 턴(`b.turnIndex`)의 위치로 부드럽게 자동 스크롤(`scrollIntoView({ behavior: 'smooth', block: 'center' })`)한다.
- [ ] 스크롤 이동된 대상 답변 턴 메시지 버블에 일시적으로 시각적 강조 테두리 및 배경 효과(`highlighted-turn`)를 2초간 적용한 뒤 자동으로 해제한다.
- [ ] 보관함 목록의 삭제 버튼(`✕`) 클릭 시 즉시 삭제 API를 호출하지 않고, 삭제 확인 모달 대화상자(`deletingBookmark`)를 화면에 표시한다.
- [ ] 삭제 확인 모달에 삭제 대상 보관 항목의 제목 또는 내용 미리보기(최대 100자 내외 말줄임표 처리)와 복구 불가능 안내 문구를 노출한다.
- [ ] 삭제 확인 모달에서 "취소" 버튼 클릭, 오버레이 배경 클릭, 또는 `Escape` 키 입력 시 삭제 작업이 취소되고 모달이 닫힌다.
- [ ] 삭제 확인 모달에서 "삭제" 버튼 클릭 시 `deleteBookmark(id)` API를 호출하여 삭제를 진행하고, 진행 중에는 모달 내 버튼을 비활성화(`disabled`)하고 "삭제 중..." 상태 피드백을 제공한다.
- [ ] 삭제 완료 시 모달을 닫고, `bookmarks` 목록 상태에서 해당 항목을 제거하며, "보관 항목이 삭제되었습니다." 토스트 알림을 유지한다.
- [ ] 사이드바 보관함 아이템 렌더링 시 답변 보관(`b.turnIndex !== undefined`) 항목에는 `[답변 보관]` 배지를, 세션 단위 보관(`b.turnIndex === undefined`) 항목에는 `[대화 전체]` 배지를 렌더링하여 직관적으로 구분할 수 있도록 한다.
- [ ] 보관함 답변 클릭 시 턴 스크롤/강조 처리, 보관 삭제 확인 모달 노출/취소/삭제 완료, 보관 유형별 배지 렌더링을 검증하는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 북마크 API(`GET/POST/DELETE /rag/bookmarks`)의 데이터베이스 스키마나 DTO 응답 규격을 변경하지 않는다.
- 대화 세션 상세 조회 API(`GET /rag/sessions/{id}`)의 규격을 변경하지 않는다.
- 답변 보관 메모 모달(`bookmark-note-modal`, SPEC-090)의 구조나 글자 수 제한(200자)을 수정하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역 화면의 동작을 수정하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 상태 및 Ref 추가:
  - `highlightedTurnIndex`: 현재 시각적으로 강조 표시 중인 턴 번호(`number | null`, 기본값 `null`).
  - `highlightTimerRef`: 강조 해제 타이머를 관리하는 `useRef<ReturnType<typeof setTimeout> | null>(null)`.
  - `deletingBookmark`: 현재 삭제 확인 모달이 열린 대상 보관 항목(`{ id: string; title?: string; content?: string } | null`, 기본값 `null`).
  - `isDeletingBookmark`: 삭제 API 요청 진행 중 여부 불리언 상태(`boolean`, 기본값 `false`).
- `handleLoadSession` 확장 (라인 457~485):
  - 시그니처를 `handleLoadSession(sid: string, targetTurnIndex?: number)`로 확장한다.
  - 세션 상세 조회 및 `chatLog` 상태 갱신이 완료된 직후, `targetTurnIndex`가 `number`로 전달된 경우 DOM에서 해당 턴 요소(예: `data-turn-index={turnIndex}` 속성을 가진 메시지 컨테이너)를 조회한다.
  - `element.scrollIntoView({ behavior: 'smooth', block: 'center' })`를 호출하여 화면 중앙으로 스크롤한다.
  - `setHighlightedTurnIndex(targetTurnIndex)`를 설정하고, `highlightTimerRef`를 통해 2000ms(2초) 후 `highlightedTurnIndex`를 `null`로 초기화한다.
- 보관함 아이템 클릭 핸들러 연동 (라인 1388):
  - `onClick={() => handleLoadSession(b.sessionId, b.turnIndex)}`로 수정하여 보관된 턴 인덱스를 함께 전달한다.
- 보관함 아이템 유형 배지 추가 (라인 1390~1400):
  - `session-item-content` 상단에 `bookmark-type-badge` 요소를 렌더링한다.
  - `b.turnIndex !== undefined`이면 텍스트 `답변 보관`과 클래스 `badge-answer-bookmark`를 부여한다.
  - `b.turnIndex === undefined`이면 텍스트 `대화 전체`와 클래스 `badge-session-bookmark`를 부여한다.
- 보관함 아이템 삭제 클릭 핸들러 수정 (라인 1404):
  - 기존의 즉시 삭제 대신 `onClick={(e) => handleDeleteBookmarkClick(b, e)}`를 호출한다.
  - `handleDeleteBookmarkClick`: `e.stopPropagation(); setDeletingBookmark({ id: b.id, title: b.title, content: b.content });`로 상태를 설정하여 확인 모달을 연다.
- 보관 항목 삭제 확인 모달 핸들러:
  - `handleCancelDeleteBookmark`: `if (isDeletingBookmark) return; setDeletingBookmark(null);`
  - `handleConfirmDeleteBookmark`: `deleteBookmark(deletingBookmark.id)` 호출, `bookmarks` 목록 필터링 제거, 모달 닫기, 성공 토스트 표시, 에러 발생 시 `setErrorMsg` 노출.
- 보관 항목 삭제 확인 모달 UI 렌더링:
  - `deletingBookmark`가 존재할 때 `modal-overlay` 및 `delete-confirm-modal`을 화면에 렌더링한다.
  - 모달 제목: "보관 항목 삭제"
  - 미리보기 텍스트: `deletingBookmark.title || deletingBookmark.content || '선택한 보관 항목'`을 최대 100자로 잘라 노출.
  - 안내 문구: "이 보관 항목을 삭제하시겠습니까? 삭제 후에는 복구할 수 없습니다."
  - [취소], [삭제] 버튼 그룹 배치.
- 메시지 버블 렌더링부 (`public-front/src/components/AiService.tsx:1440-1442`):
  - 각 메시지 버블 또는 래퍼 요소에 `data-turn-index={turnIndex}` 속성을 부여한다.
  - `turnIndex === highlightedTurnIndex`인 경우 `highlighted-turn` CSS 클래스를 부여한다.

`public-front/src/components/AiService.css`
- `.highlighted-turn`: 대상 답변 버블에 일시적으로 강조 테두리 및 펄스 배경 스타일(`border: 2px solid #2563eb; background-color: #eff6ff; transition: all 0.3s ease;`)을 부여한다.
- `.bookmark-type-badge`: 보관함 아이템 내 유형 배지 스타일(글자 크기 11px, 패딩 2px 6px, 둥근 모서리).
- `.badge-answer-bookmark`: 답변 보관 배지 테마(연청색 배경, 진파랑 텍스트).
- `.badge-session-bookmark`: 대화 전체 보관 배지 테마(연회색 배경, 짙은 회색 텍스트).
- `.bookmark-delete-modal`: 보관 항목 삭제 확인 대화상자 레이아웃 및 텍스트 스타일.

`public-front/src/components/AiService.test.tsx`
- 보관함에서 `turnIndex`가 포함된 답변 북마크 클릭 시 `handleLoadSession`이 해당 세션을 불러오고 대상 턴 요소로 스크롤 및 `highlighted-turn` 클래스를 적용하는지 검증하는 단위 테스트.
- `turnIndex`가 없는 세션 전체 북마크 클릭 시 스크롤 및 하이라이트 없이 정상적으로 세션만 로드되는지 검증하는 경계 조건 테스트.
- 보관함 아이템 삭제 버튼(`✕`) 클릭 시 즉시 삭제되지 않고 확인 모달이 열리는지 검증하는 단위 테스트.
- 삭제 확인 모달에서 "취소" 클릭 또는 `Escape` 키 입력 시 모달이 닫히고 삭제 API가 호출되지 않음을 검증하는 단위 테스트.
- 삭제 확인 모달에서 "삭제" 클릭 시 `deleteBookmark` API가 호출되고 목록에서 제거되는지 검증하는 단위 테스트.
- 보관 항목의 유형(`turnIndex` 유무)에 따라 `[답변 보관]` 및 `[대화 전체]` 배지가 올바르게 렌더링되는지 검증하는 단위 테스트.

## 수용 기준 (Acceptance Criteria)
- Given 사이드바 보관함(`bookmarks`)에 `turnIndex: 2`를 가진 답변 북마크 항목이 존재할 때 When 사용자가 해당 보관 항목을 클릭하면 Then 대상 세션이 로드되고 2번 턴의 AI 답변 메시지 요소가 화면 중앙으로 부드럽게 스크롤되며 `highlighted-turn` 강조 클래스가 부여된다.
- Given 답변 북마크 클릭으로 특정 답변에 `highlighted-turn` 강조 효과가 적용되었을 때 When 2초(2000ms)가 경과하면 Then 해당 메시지의 `highlighted-turn` 강조 클래스가 자동으로 제거된다.
- Given (경계 케이스) 사이드바 보관함에 `turnIndex`가 없는 세션 단위 북마크 항목을 클릭했을 때 When 대화 세션이 로드되면 Then 오류 없이 전체 대화 목록이 렌더링되고 특정 턴 강조나 강제 스크롤이 트리거되지 않는다.
- Given 사이드바 보관함 목록의 특정 보관 항목 행을 확인할 때 When 해당 항목이 특정 답변을 보관한 경우(`turnIndex !== undefined`) Then `[답변 보관]` 배지가 표시되고, 세션 전체를 보관한 경우(`turnIndex === undefined`) Then `[대화 전체]` 배지가 표시된다.
- Given 사이드바 보관함 목록의 삭제 버튼(`✕`)이 표시되어 있을 때 When 사용자가 삭제 버튼을 클릭하면 Then 즉시 삭제 API가 호출되지 않고 화면에 "보관 항목 삭제" 확인 모달 대화상자가 표시된다.
- Given 보관 항목 삭제 확인 모달이 열려 있는 상태에서 When 사용자가 "취소" 버튼을 누르거나 모달 바깥 배경을 클릭하거나 `Escape` 키를 누르면 Then 모달이 닫히고 보관 목록에서 항목이 삭제되지 않는다.
- Given (경계 케이스) 보관 항목의 본문 또는 제목이 100자를 초과하는 긴 텍스트일 때 When 삭제 확인 모달을 열면 Then 텍스트가 화면 밖으로 넘치지 않고 최대 100자 내외에서 말줄임표(`...`)로 안전하게 축약되어 안내된다.
- Given 보관 항목 삭제 확인 모달에서 "삭제" 버튼을 클릭했을 때 When 삭제 API 호출이 완료되면 Then 모달이 닫히고 해당 항목이 보관함 목록에서 제거되며 "보관 항목이 삭제되었습니다." 안내 메시지가 표시된다.

## 참고
- 고쳐야 할 자리 (보관함 아이템 렌더링 및 클릭부): `public-front/src/components/AiService.tsx:1384-1414` (`AiService`)
- 고쳐야 할 자리 (보관 항목 즉시 삭제 핸들러): `public-front/src/components/AiService.tsx:336-345` (`handleDeleteBookmark`)
- 고쳐야 할 자리 (세션 상세 로드 핸들러): `public-front/src/components/AiService.tsx:457-485` (`handleLoadSession`)
- 고쳐야 할 자리 (메시지 턴 렌더링 및 인덱스 부여부): `public-front/src/components/AiService.tsx:1440-1442` (`AiService`)
- 관련 정의 (북마크 인터페이스 정의부): `public-front/src/api/ai.ts:288-296` (`BookmarkItem`)
- 세션 삭제 확인 대화상자 참조 구현: `public-front/src/components/AiService.tsx:487-505` (`handleDeleteSessionClick`, `handleConfirmDeleteSession`)
