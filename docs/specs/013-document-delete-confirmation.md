---
id: SPEC-013
title: 지식베이스 문서 삭제 확인 대화상자 및 진행 상태 피드백
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 `public-front/src/components/AiService.tsx:328-333` 에서 지식베이스 문서 테이블의 "삭제" 버튼을 클릭하면 확인 절차 없이 즉시 `handleDeleteDocument(doc.id)` 가 호출된다.
이 함수(`public-front/src/components/AiService.tsx:211-219`)는 `public-front/src/api/ai.ts:21-24` 의 `deleteDocument(id)` API를 직접 실행하여 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:38-43` 엔드포인트로 삭제 요청을 보낸다.
이로 인해 다음과 같은 문제점이 발생한다.
- 사용자가 마우스 클릭 실수로 삭제 버튼을 누르면 확인 절차 없이 RAG 지식베이스에서 해당 문서와 벡터 인덱스가 즉시 영구 삭제된다.
- 삭제 요청이 처리되는 동안 로딩 표시나 버튼 비활성화 상태가 없어 중복 클릭으로 인한 다중 요청이 발생할 수 있다.
- 삭제 실패 시 백엔드에서 전달하는 구체적인 에러 메시지(있는 경우)를 파싱하지 않고 고정된 에러 문구(`"문서 삭제에 실패했습니다."`)만을 노출한다.

## 요구사항
- [ ] 문서 목록 테이블에서 "삭제" 버튼 클릭 시 즉시 삭제 API를 호출하지 않고 삭제 확인 모달(대화상자)을 표시한다.
- [ ] 삭제 확인 모달에는 삭제하려는 문서의 파일명(예: `"{fileName}" 문서를 삭제하시겠습니까?`)과 복구할 수 없다는 안내 문구를 노출한다.
- [ ] 확인 모달에서 "취소" 버튼을 누르거나 모달 배경 영역 클릭, 또는 `Escape` 키 입력 시 삭제 작업이 취소되고 모달이 닫힌다.
- [ ] 확인 모달에서 "삭제" 버튼을 누르면 `deleteDocument(id)` API를 호출하여 실제 삭제를 진행한다.
- [ ] 삭제 요청이 진행되는 동안 모달 내 삭제 버튼 텍스트를 "삭제 중..." 으로 변경하고 확인/취소 버튼을 모두 비활성화(`disabled`)하여 중복 요청을 방지한다.
- [ ] 삭제 완료 시 확인 모달을 닫고 `fetchDocuments()` 를 호출하여 최신 문서 목록을 갱신한다.
- [ ] 삭제 요청 실패 시 모달을 닫고 에러 배너(`setErrorMsg`)에 서버 응답 에러 메시지(`err.response.data.message`) 또는 기본 메시지("문서 삭제에 실패했습니다.")를 표시한다.
- [ ] 문서 삭제 확인 모달 열기/취소/확인/로딩/에러 동작에 대한 단위 및 컴포넌트 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 게이트웨이 및 Python 서버의 문서 삭제 API 수정 (기존 `DELETE /ai/knowledge/documents/:id` 엔드포인트 유지)
- 삭제된 문서의 휴지통 보관 및 복구(Soft Delete) 기능 구현
- 다중 문서 일괄 선택 및 일괄 삭제 기능 구현
- 브라우저 네이티브 `window.confirm` 사용 (웹 표준 및 스타일 일관성을 위해 React 모달 UI 컴포넌트로 구현)

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 삭제 대상 문서 정보(ID 및 파일명)를 보관하는 상태(`deletingDoc: { id: string; fileName: string } | null`) 추가
- 삭제 진행 상태를 나타내는 불리언 상태(`isDeleting: boolean`) 추가
- 삭제 버튼 클릭 시 `setDeletingDoc` 으로 모달을 열고, 확인 시 `confirmDeleteDocument` 실행
- 삭제 확인 모달 마크업 및 ESC 키 이벤트 핸들러 추가

`public-front/src/components/AiService.css`
- 삭제 확인 모달 오버레이, 다이얼로그 박스, 버튼(확인/취소), 경고 텍스트 스타일 정의

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 문서 목록에서 특정 문서(예: `handbook.pdf`)의 삭제 버튼을 클릭했을 때
  When 삭제 버튼이 눌리면
  Then API 호출이 즉시 발생하지 않고 `handbook.pdf` 파일명이 명시된 삭제 확인 모달이 화면에 표시된다.
- Given 삭제 확인 모달이 열려 있는 상태에서
  When 사용자가 "취소" 버튼을 클릭하거나 `Escape` 키를 누르면
  Then 삭제 API가 호출되지 않고 확인 모달이 즉시 닫힌다.
- Given 삭제 확인 모달이 열려 있는 상태에서
  When 사용자가 모달의 "삭제" 버튼을 클릭하면
  Then 백엔드 삭제 API가 호출되고, 요청 중에는 모달 버튼이 "삭제 중..." 으로 표시되며 비활성화된다.
- Given 삭제 요청이 성공적으로 완료되었을 때
  When 백엔드가 성공 응답을 반환하면
  Then 확인 모달이 닫히고 `fetchDocuments()` 가 호출되어 삭제된 문서가 목록에서 제거된다.
- Given 백엔드 장애나 네트워크 오류로 삭제 요청이 실패했을 때
  When API 호출이 에러를 반환하면
  Then 확인 모달이 닫히고 상단 에러 배너에 서버 에러 메시지 또는 "문서 삭제에 실패했습니다." 가 표시된다.
- Given 파일명이 빈 문자열이거나 유효하지 않은 문서 항목의 삭제 버튼을 클릭했을 때 (경계 케이스)
  When 확인 모달이 열리면
  Then 화면이 깨지지 않고 기본 텍스트("선택한 문서를 삭제하시겠습니까?")로 안전하게 렌더링된다.

## 참고
- 삭제 핸들러 기존 구현: `public-front/src/components/AiService.tsx:211-219`
- 삭제 버튼 렌더링 위치: `public-front/src/components/AiService.tsx:328-333`
- 프론트엔드 삭제 API: `public-front/src/api/ai.ts:21-24`
- 백엔드 게이트웨이 삭제 프록시 엔드포인트: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:38-43`
