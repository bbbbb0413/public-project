---
id: SPEC-090
title: AI 답변 북마크 시 메모(note) 입력 모달 연동 및 보관함 메모 표시 개선
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 사용자가 AI 답변 메시지 하단의 [★ 보관] 버튼(`public-front/src/components/AiService.tsx:1520-1530` (`AiService`))을 클릭하면, `public-front/src/components/AiService.tsx:212-227` (`handleBookmarkAnswer`) 핸들러가 사용자 입력 절차 없이 즉시 `createBookmark({ sessionId, turnIndex })`를 호출하여 북마크를 생성한다.
그러나 백엔드 스키마 `public-python-server/src/ai_service/rag/schemas.py:497-503` (`CreateBookmarkIn`)와 게이트웨이 DTO `public-server/apps/gateway/src/ai/proxy/dto/save-bookmark-in.dto.ts:4-19` (`SaveBookmarkInDto`), 프론트엔드 API 타입 `public-front/src/api/ai.ts:292-296` (`CreateBookmarkIn`)에는 이미 선택적 메모 필드인 `note?: string`이 정의되어 있으며, 백엔드 서비스 `public-python-server/src/ai_service/rag/service.py:87-96` (`BookmarkService.create_bookmark`)에서도 `note` 필드를 정상 저장하도록 구현되어 있다.
또한 사이드바 북마크 목록 `public-front/src/components/AiService.tsx:1204-1210` (`AiService`)에서도 `{b.note && <span className="session-item-note">{b.note}</span>}`를 통해 보관된 메모를 렌더링하도록 뷰 코드가 마련되어 있다.
그럼에도 불구하고 화면상에 메모를 입력할 수 있는 사용자 인터페이스(모달 또는 다이얼로그)가 전혀 존재하지 않아, 이미 배송된 백엔드/게이트웨이 계약과 프론트엔드 렌더링 코드가 미완성 상태로 방치되어 있다. 사용자는 답변을 보관하면서 자신이 왜 이 답변을 저장했는지, 어떤 핵심 맥락이나 확인 사항이 있는지 메모를 남길 수 없어 "답변을 믿을 수 있게 만들고 스스로 검증·활용한다"는 제품 방향의 가치를 온전히 누리지 못하고 있다.

## 요구사항
- [ ] AI 답변 버블 하단 액션 영역의 `★ 보관` 버튼(`btn-bookmark-answer`) 클릭 시, 즉시 API를 호출하지 않고 답변 보관 메모 입력 모달(`bookmark-note-modal`)을 표시한다.
- [ ] 보관 모달에는 대상 답변의 요약 미리보기(텍스트 최대 150자 내외 및 말줄임표 처리), 메모 입력 `textarea` 필드, 실시간 글자 수 카운터, [취소] 버튼, [보관하기] 제출 버튼을 렌더링한다.
- [ ] 메모 입력창에 적절한 안내 문구(`placeholder="보관 사유나 참고 메모를 입력하세요 (선택 사항)"`)를 제공한다.
- [ ] 메모 입력은 선택 사항(optional)으로 처리하며, 메모를 전혀 입력하지 않거나 공백만 입력한 상태에서도 [보관하기] 버튼을 통해 정상적으로 북마크를 저장할 수 있다.
- [ ] 메모 입력 최대 길이를 200자로 제한하고, 실시간 글자 수 카운터(예: `0 / 200자`)를 통해 초과 입력을 방지한다.
- [ ] 모달 내 [보관하기] 버튼 클릭 시, `createBookmark({ sessionId, turnIndex, note: note.trim() || undefined })`를 호출하여 메모가 입력된 경우 `note` 필드를 함께 전송한다.
- [ ] 보관 요청이 진행되는 동안 [보관하기] 버튼 텍스트를 "저장 중..."으로 변경하고 확인/취소 버튼을 모두 비활성화(`disabled`)하여 중복 제출을 차단한다.
- [ ] 보관 완료 시 모달을 닫고, 성공 토스트 알림("답변이 보관함에 저장되었습니다.")을 표시하며, 사이드바가 북마크 탭(`sidebarTab === 'bookmarks'`)인 경우 `fetchBookmarks()`를 호출하여 목록을 갱신한다.
- [ ] 보관 API 실패 시 모달 내 또는 상단 에러 배너(`setErrorMsg`)에 "답변 보관에 실패했습니다." 오류 메시지를 표시하고 사용자가 재시도할 수 있도록 지원한다.
- [ ] 모달 상단 닫기 버튼(`✕`), 하단 [취소] 버튼, 모달 바깥 배경 오버레이 클릭, 키보드 `Escape` 키 입력 시 모달을 닫고 입력 중인 메모 상태를 초기화한다.
- [ ] 사이드바 북마크 목록(`session-item bookmark-item`)의 메모 영역(`session-item-note`)에 전체 메모 내용을 확인할 수 있도록 `title={b.note}` 툴팁 속성을 부여한다.
- [ ] 답변 보관 모달 열기/닫기 인터랙션, 메모 입력 및 글자 수 카운터 검증, 메모 포함/미포함 저장 API 호출 파라미터 검증을 다루는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드(`public-python-server`) 및 게이트웨이(`public-server`)의 북마크 데이터 스키마 변경이나 신규 엔드포인트 추가는 하지 않는다. 기존 `POST /ai/rag/bookmarks` 및 `SaveBookmarkInDto`의 `note` 필드를 그대로 활용한다.
- 이미 저장된 기존 북마크의 메모 내용을 수정(편집)하거나 북마크 메모만 별도로 부분 수정하는 신규 REST API/UI는 이번 작업에 포함하지 않는다. (수정 필요 시 기존 보관 삭제 후 재등록 흐름 유지)
- 세션 단위 북마크 토글(`handleToggleBookmarkSession`)에 대한 메모 입력 기능은 포함하지 않으며, 개별 답변 단위 보관(`handleBookmarkAnswer`)에 한정한다.
- 북마크 메모에 대한 서식 지정(Markdown/Rich Text) 기능은 제공하지 않으며, 순수 텍스트(Plain Text)로만 제한한다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - 보관 모달 열림 상태(`bookmarkModalTarget: { turnIndex: number; text: string } | null`), 메모 입력 상태(`bookmarkNote: string`), 보관 저장 진행 상태(`isSavingBookmark: boolean`) 관리
  - `handleBookmarkAnswer`를 수정하여 즉시 API 호출 대신 `setBookmarkModalTarget({ turnIndex, text })`로 모달을 열도록 변경
  - 모달 닫기 핸들러(`handleCloseBookmarkModal`) 및 실제 저장 핸들러(`handleConfirmBookmarkAnswer`) 구현
  - 답변 보관 메모 모달 마크업(`modal-overlay`, `bookmark-note-modal`, `modal-header`, `modal-body`, `modal-footer`) 추가 및 `Escape` 키 이벤트 리스너 연동
  - 사이드바 북마크 목록 렌더링부(`session-item-note`)에 `title={b.note}` 속성 부여
- `public-front/src/components/AiService.css`:
  - 보관 메모 모달 전용 레이아웃 스타일, 미리보기 인용 블록(`bookmark-preview-snippet`), 메모 입력창 textarea 스타일, 글자 수 카운터(`bookmark-char-count`) 스타일 추가
- `public-front/src/components/AiService.test.tsx`:
  - 답변 버블의 [★ 보관] 버튼 클릭 시 메모 모달 렌더링 검증
  - 메모를 입력하고 [보관하기] 클릭 시 `createBookmark`에 `sessionId`, `turnIndex`, `note`가 전달되는지 검증
  - 메모 없이 [보관하기] 클릭 시 `note: undefined`로 전달되는지 검증
  - 취소 버튼 또는 `Escape` 키 클릭 시 모달이 닫히고 저장이 취소되는지 검증

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 AI 답변을 조회 중인 대화 세션 화면에서, When 특정 AI 답변 버블의 [★ 보관] 버튼(`btn-bookmark-answer`)을 클릭하면, Then 즉시 API가 호출되지 않고 답변 본문 일부가 미리보기로 표시되는 답변 보관 모달(`bookmark-note-modal`)이 화면에 열린다.
- Given 답변 보관 모달이 열려 있는 상태에서, When 메모 입력창에 `"프로젝트 분기 보고서 핵심 근거"`를 입력하고 [보관하기] 버튼을 누르면, Then `createBookmark` 함수가 `{ sessionId, turnIndex, note: "프로젝트 분기 보고서 핵심 근거" }` 객체와 함께 호출되고 모달이 닫히며 "답변이 보관함에 저장되었습니다." 토스트가 노출된다.
- Given 답변 보관 모달이 열려 있는 상태에서, When 메모를 입력하지 않고(빈 값 또는 공백) [보관하기] 버튼을 누르면, Then 에러 없이 `createBookmark` 함수가 `{ sessionId, turnIndex, note: undefined }`로 호출되어 메모 없이 정상적으로 답변이 보관된다. (경계 케이스: 메모 미입력)
- Given 답변 보관 모달이 열려 있는 상태에서, When 메모 입력창에 200자를 초과하는 긴 텍스트를 입력하려고 시도하면, Then 최대 200자까지만 입력이 유지되고 글자 수 카운터에 `200 / 200자`가 표시되며 초과 입력이 차단된다. (경계 케이스: 최대 글자 수 제한)
- Given 답변 보관 모달이 열려 있을 때, When 사용자가 [취소] 버튼을 누르거나 모달 닫기 버튼(`✕`)을 누르거나 모달 바깥 오버레이 영역을 클릭하거나 키보드 `Escape` 키를 누르면, Then API 호출 없이 모달이 닫히고 입력 중이던 메모 내용이 초기화된다. (경계 케이스: 취소 인터랙션)
- Given 답변 보관 API 호출이 네트워크 지연으로 처리 중일 때, When 사용자가 모달을 확인하면, Then [보관하기] 버튼이 "저장 중..."으로 표시되고 확인/취소 버튼이 모두 비활성화되어 중복 클릭이 차단된다.
- Given 메모가 포함된 북마크가 저장되어 사이드바 보관함 목록에 표시될 때, When 사용자가 해당 북마크 항목의 메모(`session-item-note`)에 마우스 커서를 올리면, Then `title` 속성을 통해 메모 원문 전체가 툴팁으로 표시된다.

## 참고
- 게이트웨이 북마크 생성 DTO: `public-server/apps/gateway/src/ai/proxy/dto/save-bookmark-in.dto.ts:4-19` (`SaveBookmarkInDto`)
- 게이트웨이 북마크 프록시 엔드포인트: `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:145-156` (`RagSessionProxyController.createBookmark`)
- 백엔드 북마크 입력 스키마: `public-python-server/src/ai_service/rag/schemas.py:497-503` (`CreateBookmarkIn`)
- 백엔드 북마크 생성 비즈니스 로직: `public-python-server/src/ai_service/rag/service.py:87-96` (`BookmarkService.create_bookmark`)
- 프론트엔드 북마크 API 클라이언트 및 타입: `public-front/src/api/ai.ts:292-296` (`CreateBookmarkIn`) 및 `public-front/src/api/ai.ts:350-355` (`createBookmark`)
- 프론트엔드 기존 답변 보관 핸들러: `public-front/src/components/AiService.tsx:212-227` (`handleBookmarkAnswer`)
- 프론트엔드 답변 버블 보관 버튼: `public-front/src/components/AiService.tsx:1520-1530` (`AiService`)
- 프론트엔드 사이드바 북마크 목록 메모 표시부: `public-front/src/components/AiService.tsx:1204-1210` (`AiService`)
