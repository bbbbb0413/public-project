---
id: SPEC-077
title: 지식베이스 완료 문서 상태값(processed) 불일치 수정 및 다운로드 버튼 노출 정상화
status: done
targets: [front]
stages: [frontend, qa]
priority: high
---
## 배경 / 문제
SPEC-067을 통해 지식베이스 문서 목록 테이블에서 인제스트가 완료된 문서의 원본 파일을 내려받을 수 있도록 `[다운로드]` 버튼과 진행 상태 피드백을 추가했다. 그러나 백엔드가 반환하는 문서 완료 상태값과 프론트엔드 다운로드 버튼의 노출 조건이 서로 어긋나 있어 실제 운영 환경에서는 완료된 문서의 `[다운로드]` 버튼이 화면에 전혀 나타나지 않는다.

- Python 백엔드 도메인 스키마인 `public-python-server/src/ai_service/knowledge/schemas.py:7`의 `DocumentStatus` 타입은 문서 상태를 `"pending"`, `"processed"`, `"failed"` 세 가지로 정의하며, 인제스트 완료 시 `public-python-server/src/ai_service/knowledge/schemas.py:59-69` (`Document.mark_processed`)가 `status="processed"`로 저장한다.
- 문서 목록 조회 HTTP 입구인 `public-python-server/src/ai_service/knowledge/router.py:12-15` (`list_documents`)와 응답 모델 `public-python-server/src/ai_service/knowledge/schemas.py:152-171` (`DocumentOut.from_domain`) 역시 색인이 끝난 문서의 `status`를 `"processed"`로 직렬화하여 반환한다.
- 프론트엔드 `public-front/src/components/AiService.tsx:682` (`AiService`)의 대시보드 완료 문서 수 집계(`completedDocsCount`)와 `public-front/src/components/AiService.css:358-362`의 완료 상태 배지 스타일(`.status-badge.processed`)은 모두 `'processed'`를 기준으로 작성되어 있다.
- 반면 정작 문서 목록 테이블(`doc-table`)의 작업 열에서 다운로드 버튼을 렌더링하는 `public-front/src/components/AiService.tsx:859-869` (`AiService`)는 `doc.status === 'completed'`일 때만 버튼을 노출하도록 작성되어 있고, 단위 테스트인 `public-front/src/components/AiService.test.tsx:4024-4033` (`AiService`) 또한 목 데이터의 `status`를 `'completed'`로만 고정해 검증하면서 실제 백엔드 계약(`'processed'`)과의 불일치가 걸러지지 않았다.

이번 분기 핵심 방향인 "답변을 믿을 수 있게 만든다"에 따라 사용자가 RAG 답변의 근거가 되는 지식베이스 원본 문서를 직접 내려받아 스스로 대조·검증할 수 있도록 배송한 SPEC-067이 상태값 불일치로 작동하지 않는 결함을 바로잡는다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 문서 목록 테이블 행 렌더링에서 `doc.status === 'processed'`인 문서에 `[다운로드]` 버튼(`btn-download`)이 노출되고 활성화되어야 한다.
- [ ] 기존 테스트 및 하위 호환성을 위해 `doc.status === 'completed'`인 경우에도 동일하게 완료 문서로 판정하여 `[다운로드]` 버튼이 노출되고 대시보드 완료 문서 수(`completedDocsCount`) 집계 및 완료 상태 배지 스타일이 일관되게 동작하도록 처리한다.
- [ ] `doc.status === 'processed'`인 문서 행에서 `[다운로드]` 버튼 클릭 시 `getDocumentFile(doc.id)`가 호출되어 해당 문서의 `fileName`으로 파일 다운로드가 정상 실행되어야 한다.
- [ ] 인제스트 진행 중(`pending`, `processing`, `queued`)이거나 실패(`failed`, `error`) 상태, 또는 빈 문자열(`""`) 등 미완료 상태인 문서 행에는 `[다운로드]` 버튼이 노출되지 않아야 한다.
- [ ] `public-front/src/components/AiService.test.tsx`에 백엔드 실제 완료 상태값인 `status: 'processed'` 문서에 대한 다운로드 버튼 렌더링, 클릭 시 다운로드 트리거, 다운로드 진행 중 비활성화(`다운로드 중...`) 및 실패 시 에러 배너 노출을 검증하는 단위 테스트를 추가·보강한다.

## 비요구사항 (Out of scope)
- Python 백엔드(`public-python-server`)의 `DocumentStatus` 값(`"processed"`)이나 MongoDB에 저장된 기존 문서 데이터의 `status` 필드를 변경하는 마이그레이션은 수행하지 않는다.
- NestJS 게이트웨이(`public-server`)의 문서 목록·파일 다운로드 프록시 엔드포인트(`KnowledgeProxyController`)는 수정하지 않는다.
- 실패 문서 재시도 로직 변경이나 문서 일괄 다운로드, 인라인 미리보기 뷰어 등 새로운 기능 추가는 포함하지 않는다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `AiService`(`AiService.tsx:682`)의 `completedDocsCount` 계산 및 문서 테이블(`AiService.tsx:833-873`) 행 렌더링에서 완료 상태 판정 조건을 `doc.status === 'processed' || doc.status === 'completed'`(예: `isCompleted`)로 통일한다.
  - `AiService.tsx:859`의 다운로드 버튼 조건부 렌더링을 `isCompleted`로 변경하여 백엔드가 반환하는 `status: 'processed'` 문서에서 `[다운로드]` 버튼이 정상 노출되도록 수정한다.
- `public-front/src/components/AiService.css`:
  - `.status-badge.processed`(`AiService.css:358-362`)에 `.status-badge.completed` 선택자를 함께 지정하여 두 상태값 모두 동일한 완료 배지 스타일을 갖도록 정리한다.
- `public-front/src/components/AiService.test.tsx`:
  - `status: 'processed'`인 문서에서 `[다운로드]` 버튼이 정상 렌더링되고 `getDocumentFile` 호출 및 다운로드 상태 피드백이 동작하는지 검증하는 테스트 케이스를 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 백엔드 문서 목록 API(`GET /ai/knowledge/documents`)가 인제스트 완료 상태인 `status: 'processed'` 문서(`id: 'doc-1'`, `fileName: 'guide.pdf'`)를 반환했을 때, When `AiService` 화면의 문서 목록 테이블이 렌더링되면, Then 해당 문서 행의 작업 열에 활성화된 `[다운로드]` 버튼이 표시되고 클릭 시 `getDocumentFile('doc-1')`이 호출되어 `guide.pdf` 다운로드가 실행된다.
- Given 문서 목록에 `status: 'processed'` 문서 2건이 표시되어 있을 때, When 첫 번째 문서의 `[다운로드]` 버튼을 클릭하여 요청이 진행 중이면, Then 첫 번째 문서 버튼만 `"다운로드 중..."` 텍스트와 함께 `disabled` 상태가 되고 두 번째 문서의 `[다운로드]` 버튼은 활성 상태로 유지된다.
- Given 문서의 `status`가 `'pending'`, `'processing'`, `'queued'`, `'failed'`, `'error'` 또는 빈 문자열(`''`)인 경우(경계 케이스), When 문서 목록 테이블이 렌더링되면, Then 해당 문서 행에는 `[다운로드]` 버튼이 렌더링되지 않는다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:682` (`AiService`), `public-front/src/components/AiService.tsx:859-869` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.test.tsx:4024-4033` (`AiService`)
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/knowledge/schemas.py:7` (`DocumentStatus`), `public-python-server/src/ai_service/knowledge/schemas.py:59-69` (`Document.mark_processed`), `public-python-server/src/ai_service/knowledge/router.py:12-15` (`list_documents`), `public-front/src/components/AiService.css:358-362`
