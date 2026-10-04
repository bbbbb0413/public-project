---
id: SPEC-058
title: 대화 세션 삭제 확인 대화상자 및 안전장치 추가
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
대화 이력 사이드바에서 세션 삭제 시 확인 절차가 없어 사용자의 오클릭으로 인해 대화 기록이 영구 유실되는 문제가 있다.
`public-front/src/components/AiService.tsx:1020-1026` (`AiService`)에서 사이드바 세션 목록의 삭제 버튼(`btn-delete-session`)을 클릭하면 확인 대화상자 없이 즉시 `handleDeleteSession`이 호출된다.
`public-front/src/components/AiService.tsx:345-354` (`handleDeleteSession`)에서는 `deleteSessionById(sid)`를 직접 호출하여 백엔드 DB에서 세션을 영구 삭제하고, 현재 열려 있는 세션인 경우 `handleNewChat()`으로 화면 상태까지 즉시 초기화한다.
반면 지식베이스 문서의 경우 `public-front/src/components/AiService.tsx:1550-1596` (`AiService`)에서 삭제 확인 모달(`delete-confirm-modal`)을 통해 사용자 확인을 거치도록 안전장치가 구현되어 있으나, 대화 세션 삭제에는 이러한 안전장치가 전무하다.
`public-front/src/api/ai.ts:364-366` (`deleteSessionById`)을 통한 세션 삭제는 RAG 질의응답 이력, 신뢰도 정보, 피드백 등 축적된 대화 데이터를 영구히 제거하므로 오클릭을 방지하고 작업 진행 피드백을 제공하는 확인 대화상자가 필수적이다.

## 요구사항
- [ ] 사이드바 대화 세션 목록의 "삭제" 버튼(`✕`) 클릭 시 즉시 삭제 API를 호출하지 않고 세션 삭제 확인 모달을 화면에 표시한다.
- [ ] 세션 삭제 확인 모달에 삭제 대상 세션의 제목(예: `"{title}" 대화를 삭제하시겠습니까?`) 및 복구 불가능 안내 문구를 노출한다. 세션 제목이 없을 경우 `"선택한 대화를 삭제하시겠습니까?"`를 기본으로 표시한다.
- [ ] 확인 모달에서 "취소" 버튼 클릭, 오버레이 배경 영역 클릭, 또는 `Escape` 키 입력 시 삭제 작업이 취소되고 모달이 닫힌다.
- [ ] 확인 모달에서 "삭제" 버튼 클릭 시 `deleteSessionById(sessionId)` API를 호출하여 세션을 삭제한다.
- [ ] 삭제 요청이 진행되는 동안 모달 내 삭제 버튼 텍스트를 "삭제 중..."으로 변경하고 취소/삭제 버튼을 비활성화(`disabled`)하여 중복 요청을 방지한다.
- [ ] 세션 삭제가 완료되면 확인 모달을 닫고, 세션 목록에서 해당 세션을 제거하며, 현재 열려 있던 세션이 삭제된 경우 대화창을 새 질문 상태로 초기화한다.
- [ ] 세션 삭제 API 호출 실패 시 확인 모달을 닫고 상단 에러 배너(`errorMsg`)에 `"세션 삭제에 실패했습니다."` 에러 메시지를 노출한다.

## 비요구사항 (Out of scope)
- 백엔드 세션 삭제 API(`DELETE /ai/rag/sessions/:sessionId`)나 게이트웨이 엔드포인트의 수정은 진행하지 않는다. (프론트엔드 UI/UX 단독 개선)
- 북마크(보관함) 항목의 개별 삭제 확인 모달은 이번 범위에 포함하지 않는다. (대화 세션 삭제에만 집중)
- 삭제된 세션의 휴지통 기능이나 복원(Undo) 기능은 구현하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 세션 삭제 확인 모달 상태(`deletingSession: { sessionId: string; title: string } | null`, `isDeletingSession: boolean`)를 추가한다.
- 사이드바 세션 삭제 버튼 클릭 시 `handleDeleteSessionClick(session, e)`을 통해 `deletingSession` 상태를 설정하여 확인 모달을 연다.
- 세션 삭제 확인 모달 UI(`session-delete-modal` 또는 기존 `delete-confirm-modal` 스타일 재사용)를 렌더링하고, 취소(`handleCancelDeleteSession`) 및 확인(`handleConfirmDeleteSession`) 핸들러를 연결한다.
- `Escape` 키 입력 시 열려 있는 세션 삭제 모달이 닫히도록 키보드 이벤트를 처리한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 대화 세션 목록을 조회한 상태에서
  When 특정 세션의 삭제(`✕`) 버튼을 클릭하면
  Then `deleteSessionById` API가 즉시 호출되지 않고 세션 삭제 확인 모달이 표시되며 세션 제목과 경고 문구가 노출된다.
- Given 세션 삭제 확인 모달이 열려 있는 상태에서
  When 사용자가 "취소" 버튼을 누르거나 모달 바깥 배경을 클릭하거나 `Escape` 키를 누르면
  Then 삭제 API가 호출되지 않고 모달이 닫히며 기존 세션 목록이 그대로 유지된다.
- Given 세션 삭제 확인 모달이 열려 있는 상태에서
  When 사용자가 "삭제" 버튼을 클릭하면
  Then 버튼이 "삭제 중..."으로 바뀌고 비활성화되며, `deleteSessionById`가 호출되어 성공 시 모달이 닫히고 목록에서 해당 세션이 제거된다.
- Given 세션 제목이 빈 문자열(`""`)이거나 없는 세션에 대해
  When 삭제 버튼을 클릭하면
  Then 확인 모달에 `"선택한 대화를 삭제하시겠습니까?"`라는 기본 문구가 오류 없이 안전하게 노출된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:345-354` (`handleDeleteSession`)
- 세션 목록 렌더링 및 삭제 버튼: `public-front/src/components/AiService.tsx:1020-1026` (`AiService`)
- 기존 문서 삭제 모달 참고 구현: `public-front/src/components/AiService.tsx:1550-1596` (`AiService`)
- 세션 삭제 API 호출부: `public-front/src/api/ai.ts:364-366` (`deleteSessionById`)
