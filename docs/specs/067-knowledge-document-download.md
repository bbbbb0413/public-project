---
id: SPEC-067
title: 지식베이스 문서 목록 내 원본 파일 다운로드 기능 연동 및 상태 피드백
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 지식베이스에 등록된 문서의 원본 파일 바이너리를 조회하는 백엔드 API와 프론트엔드 API 클라이언트 함수가 이미 구현되어 배송된 상태이다.
`public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:40-49` (`KnowledgeProxyController.getFile`)에 문서 원본 다운로드 게이트웨이 프록시 엔드포인트(`GET /ai/knowledge/documents/:id/file`)가 준비되어 있고, `public-front/src/api/ai.ts:408-413` (`getDocumentFile`)에 해당 엔드포인트로부터 파일 Blob 데이터를 조회하는 API 함수가 정의되어 있다.
하지만 `public-front/src/components/AiService.tsx:811-849` (`AiService`)의 지식베이스 문서 목록 테이블(`doc-table`) 액션 영역(`doc-actions`)에는 "재시도" 및 "삭제" 버튼만 존재하고 "다운로드" 버튼이 누락되어 있다.
이로 인해 사용자는 자신이 업로드한 지식베이스 문서 목록에서 원본 파일(.txt, .pdf, .md)을 직접 내려받아 검토하거나 백업할 수 없으며, 오직 RAG 답변 하단의 출처 스니펫을 통해서만 원문 팝업을 확인할 수 있는 불편이 존재한다.
또한 파일 다운로드 시 네트워크 지연이 발생할 경우 다운로드 진행 상태 피드백이 없어 사용자가 중복 클릭을 유발할 수 있다.

## 요구사항
- [ ] 지식베이스 문서 목록 테이블의 각 행 작업(`doc-actions`) 영역에 [다운로드] 버튼을 추가한다.
- [ ] 색인 완료(`status === 'completed'`) 상태인 문서에 대해서만 [다운로드] 버튼을 노출하거나 활성화한다.
- [ ] 사용자가 [다운로드] 버튼 클릭 시 `getDocumentFile(doc.id)`를 호출하여 브라우저에서 해당 파일명(`doc.fileName`)으로 다운로드가 실행되도록 처리한다.
- [ ] 다운로드가 진행되는 동안 해당 버튼 텍스트를 "다운로드 중..."으로 변경하고 버튼을 비활성화(`disabled`)하여 중복 요청을 방지한다.
- [ ] 다운로드 API 호출 실패 시 에러 배너에 "문서 다운로드에 실패했습니다." 에러 메시지를 노출한다.
- [ ] 여러 문서가 목록에 있을 때 각 문서 행의 다운로드 진행 상태는 서로 독립적으로 관리되어야 한다.

## 비요구사항 (Out of scope)
- 백엔드 파일 다운로드 API(`GET /ai/knowledge/documents/:id/file`)나 GridFS 스토리지 조회 로직 수정
- 문서 일괄 다운로드(ZIP 압축 다운로드) 기능 추가
- 문서 업로드 용량 제한 상향 또는 신규 파일 포맷 파서 추가
- 문서 뷰어(인라인 PDF 뷰어 등) 화면 신설

## 프론트엔드
`public-front/src/components/AiService.tsx`의 지식베이스 문서 목록 테이블(`doc-table`) 렌더링 영역 및 다운로드 핸들러:
- 개별 문서 다운로드 상태를 추적하기 위한 상태(예: `downloadingDocId: string | null`) 추가
- `handleDownloadDocument(doc: DocumentInfo)` 핸들러 구현:
  - `getDocumentFile(doc.id)` 호출 후 `URL.createObjectURL(blob)` 생성
  - 임시 `<a>` 엘리먼트 생성 및 `download` 속성에 `doc.fileName` 지정 후 클릭 트리거
  - `URL.revokeObjectURL`을 통한 메모리 해제
  - 실패 시 `setErrorMsg('문서 다운로드에 실패했습니다.')` 설정
- `public-front/src/components/AiService.css`에 다운로드 버튼 스타일(`btn-download`) 추가
- `public-front/src/components/AiService.test.tsx`에 다운로드 버튼 렌더링, 다운로드 성공 트리거, 다운로드 중 비활성화 상태, 에러 처리 단위 테스트 추가

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 색인 완료(`status: 'completed'`)된 "report.pdf" 문서가 목록에 표시되어 있을 때
  When 사용자가 해당 행의 [다운로드] 버튼을 클릭하면
  Then `getDocumentFile`이 해당 문서 ID로 호출되고 브라우저에서 "report.pdf" 파일 다운로드가 시작된다.
- Given 특정 문서의 파일 다운로드가 네트워크 통신 중일 때
  When 다운로드가 완료되기 전까지
  Then 해당 행의 다운로드 버튼 텍스트가 "다운로드 중..."으로 표시되고 클릭이 불가능한 `disabled` 상태가 된다.
- Given 다운로드 API 호출 시 네트워크 오류나 500 응답이 발생했을 때
  When 다운로드 핸들러가 예외를 포착하면
  Then 화면 상단 에러 배너에 "문서 다운로드에 실패했습니다." 메시지가 표시되고 다운로드 진행 상태가 해제된다.
- Given 인제스트 진행 중(`status: 'processing'`)이거나 실패(`status: 'failed'`) 상태인 문서의 경우
  When 문서 목록 테이블이 렌더링될 때
  Then 해당 문서 행에는 다운로드 버튼이 노출되지 않거나 비활성화되어 미완성된 파일 다운로드가 차단된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:839-846` (`AiService`)
- 기존 API 정의: `public-front/src/api/ai.ts:408-413` (`getDocumentFile`)
- 백엔드 게이트웨이 엔드포인트: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:40-49` (`KnowledgeProxyController.getFile`)
- 원문 보기 유사 구현 참고: `public-front/src/components/AiService.tsx:266-286` (`handleViewSourceFile`)
