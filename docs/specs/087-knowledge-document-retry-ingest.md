---
id: SPEC-087
title: 지식베이스 실패 문서 원본 기반 재인제스트 API 연동 및 재시도 UI 개선
status: ready
targets: [front, python-server, server]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 지식베이스 문서 인제스트 실패 시 사용자가 원본 파일을 재업로드하지 않고도 즉시 재처리할 수 있는 백엔드 엔드포인트와 프론트엔드 연동이 누락되어 있다.
`public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:93-107` (`IngestDocumentUseCase.execute`)에서는 나중에 재시도할 때 사용할 수 있도록 업로드 시 원본 파일을 `save_original_file`로 GridFS/스토리지에 안전하게 보존하고, `command.document_id`가 주어지면 기존 문서를 갱신하도록 구현되어 있다.
그러나 `public-python-server/src/ai_service/knowledge/router.py:32-51` (`get_document_file`, `delete_document`)에는 원본 파일 다운로드와 삭제 엔드포인트만 있을 뿐, 보존된 원본 파일을 꺼내어 재인제스트를 실행하는 `POST /{document_id}/retry` 엔드포인트가 정의되어 있지 않다.
또한 게이트웨이 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:34-56` (`KnowledgeProxyController`)에도 청크 조회, 원본 파일 다운로드, 문서 삭제 프록시는 존재하지만 재시도 프록시 라우트가 부재하다.
이로 인해 프론트엔드 `public-front/src/components/AiService.tsx:975-977` (`AiService`)에서 실패 상태(`isFailed`)인 문서 행에 렌더링된 [재시도] 버튼을 클릭해도, `public-front/src/components/AiService.tsx:520-522` (`handleRetryUpload`)는 대상 문서 식별자 없이 단순히 OS 파일 선택 다이얼로그(`fileInputRef.current?.click()`)만 다시 띄운다.
결과적으로 사용자는 이미 서버에 올려둔 실패 문서를 즉시 재처리하지 못하고 PC에서 원본 파일을 다시 찾아 새로 업로드해야 하며, 이 과정에서 새 문서 레코드가 중복 생성되어 기존 실패 문서는 데이터베이스에 영구히 방치되는 문제가 발생한다.
RAG 질의응답의 신뢰할 수 있는 답변 생성은 온전한 지식베이스 색인에서 출발하므로, 실패한 문서의 보존된 원본을 바탕으로 동일 식별자에 대한 재인제스트를 수행하고 화면에서 직관적인 진행 상태 피드백을 제공하도록 개선해야 한다.

## 요구사항
- [ ] `public-python-server`의 `router.py`에 보존된 원본 파일을 기반으로 재인제스트를 수행하는 `POST /knowledge/documents/{document_id}/retry` 엔드포인트를 구현한다.
- [ ] 재인제스트 요청 시 대상 문서(`document_id`)가 존재하지 않거나 보존된 원본 파일이 누락된 경우 HTTP 404 Not Found 에러를 반환한다.
- [ ] 재인제스트 성공 시 `DocumentOut` 스키마 형태로 갱신된 문서 정보(`status: "processed"`, `chunkCount`)를 반환한다.
- [ ] 게이트웨이 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts`에 `POST /ai/knowledge/documents/:id/retry` 엔드포인트를 추가하고 ai-service-py로 요청을 프록시한다.
- [ ] 프론트엔드 `public-front/src/api/ai.ts`에 `retryDocument(documentId: string): Promise<DocumentItem>` API 함수를 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 실패 문서 행 [재시도] 버튼 클릭 시 파일 선택창을 열지 않고 `retryDocument(doc.id)`를 호출하도록 수정한다.
- [ ] 재시도 요청이 진행되는 동안 해당 문서 행의 [재시도] 버튼 텍스트를 "재시도 중..."으로 변경하고 버튼을 비활성화(`disabled`)하여 중복 요청을 방지한다.
- [ ] 여러 문서가 목록에 존재할 때 재시도 진행 상태(`retryingDocId`)는 개별 문서 단위로 독립적으로 동작해야 한다.
- [ ] 재시도가 성공하면 최신 문서 목록(`fetchDocuments`)을 갱신하여 문서 상태가 완료 배지로 변경되고 정상화되어야 한다.
- [ ] 재시도 실패 시 에러 배너에 오류 메시지를 노출하고 버튼 상태를 원래대로 복귀한다.
- [ ] 백엔드 라우터, 게이트웨이 프록시 컨트롤러 및 프론트엔드 컴포넌트에 대한 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 문서 업로드 시 사용되는 Kafka 비동기 잡 및 Redis Streams SSE 단계별 진행률 스트리밍 구조는 재시도에 강제하지 않고, 관리 프록시 단건 요청(동기 처리)으로 단순화하여 구현한다.
- 지식베이스 문서 목록 화면의 테이블 레이아웃이나 페이징 구조 변경은 포함하지 않는다.
- 원본 파일 다운로드(`GET /file`)나 청크 조회(`GET /chunks`) 로직은 수정하지 않는다.
- 파일 내용 자체를 수정하여 재시도하는 에디터 기능은 제공하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/main.py`:
  - `knowledge_composition` 빌드 시 생성된 `ingest_use_case`를 `app.state.ingest_use_case`에 바인딩한다.
- `public-python-server/src/ai_service/knowledge/dependencies.py`:
  - `request.app.state.ingest_use_case`를 주입하는 `get_ingest_use_case` 및 `IngestUseCaseDep` 어노테이션을 정의한다.
- `public-python-server/src/ai_service/knowledge/router.py`:
  - `POST /{document_id}/retry` 엔드포인트(`retry_document_ingest`)를 선언한다.
  - `repo.find_by_id(document_id)`로 문서를 확인하고, `repo.get_original_file(document_id)`로 보존된 바이너리 원본 및 파일 메타데이터를 조회한다.
  - `IngestDocumentCommand(document_id=document.id, file_name=file_name, mime_type=mime_type, content=content)`를 생성하여 `ingest_use_case.execute`를 실행한다.
  - 처리 완료된 도메인 객체를 `DocumentOut.from_domain(processed)`로 변환하여 반환한다.
- `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts`:
  - `@Post(':id/retry')` 라우트를 선언하고 `this.aiServicePy.post({ method: \`knowledge/documents/\${id}/retry\` })`를 호출하여 프록시한다.
- `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.spec.ts`:
  - `POST /ai/knowledge/documents/:id/retry` 요청 시 aiServicePy로 올바른 엔드포인트가 위임되는지 검증하는 단위 테스트를 추가한다.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `export const retryDocument = async (documentId: string): Promise<DocumentItem>` 함수를 추가하여 `POST /ai/knowledge/documents/${documentId}/retry`를 호출한다.
- `public-front/src/components/AiService.tsx`:
  - `retryingDocId` 상태(`string | null`)를 추가하여 현재 재처리 중인 문서 ID를 추적한다.
  - `handleRetryUpload` 대신 `handleRetryDocument(doc: DocumentInfo)` 핸들러를 구현한다.
  - 문서 목록 렌더링에서 `isFailed` 조건의 [재시도] 버튼 클릭 시 `handleRetryDocument(doc)`를 트리거한다.
  - 재시도 진행 중(`retryingDocId === doc.id`)일 때 버튼 텍스트를 "재시도 중..."으로 표시하고 `disabled` 속성을 부여한다.
  - 재시도 완료 시 `fetchDocuments()`를 호출하고 기존 에러 상태를 초기화한다.
  - 실패 시 catch 블록에서 에러 메시지를 `setErrorMsg`에 반영한다.

## 수용 기준 (Acceptance Criteria)
- Scenario: 실패 문서 원본 기반 재인제스트 성공
  - Given 지식베이스에 "doc-failed-1" 문서가 실패(`status: "failed"`) 상태로 존재하고 원본 파일이 보존되어 있을 때
  - When 사용자가 해당 행의 [재시도] 버튼을 클릭하면
  - Then 버튼 텍스트가 "재시도 중..."으로 변경되고 버튼이 비활성화되며, 백엔드로 `POST /ai/knowledge/documents/doc-failed-1/retry` 요청이 전송된다.
  - And 재처리가 성공하면 문서 상태가 완료(`processed` 또는 `completed`)로 갱신되고 청크 수가 정상적으로 표시된다.

- Scenario: 재시도 실패 시 에러 메시지 노출 및 상태 복구
  - Given 재인제스트 도중 임베딩 모델 호출 실패 등 백엔드 에러가 발생한 경우
  - When 재시도 요청이 실패(HTTP 500 등)하면
  - Then 화면 상단 에러 배너에 "문서 재처리에 실패했습니다."(또는 백엔드 반환 메시지)가 표시되고, 해당 버튼은 다시 활성화된 [재시도] 상태로 복귀한다.

- Scenario: 경계 케이스 — 존재하지 않거나 원본 파일이 손실된 문서 재시도
  - Given 데이터베이스에 존재하지 않는 ID("doc-not-found") 또는 원본 파일이 누락된 문서에 대해
  - When `POST /knowledge/documents/doc-not-found/retry` 요청을 전송하면
  - Then 시스템은 HTTP 404 Not Found 에러를 반환하고 데이터 손상 없이 요청을 거절한다.

## 참고
- 원본 보존 및 재인제스트 지원 유스케이스: `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:93-107` (`IngestDocumentUseCase.execute`)
- 실패 문서 [재시도] 버튼 클릭 핸들러 결함: `public-front/src/components/AiService.tsx:520-522` (`handleRetryUpload`)
- 실패 문서 액션 렌더링 위치: `public-front/src/components/AiService.tsx:975-977` (`AiService`)
- 게이트웨이 지식문서 프록시 컨트롤러: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:34-56` (`KnowledgeProxyController`)
- 파이썬 백엔드 지식문서 라우터 정의: `public-python-server/src/ai_service/knowledge/router.py:32-51` (`get_document_file`, `delete_document`)
- 프론트엔드 지식문서 API 클라이언트: `public-front/src/api/ai.ts:123-126` (`getDocuments`)
