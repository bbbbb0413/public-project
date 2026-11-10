---
id: SPEC-101
title: 지식베이스 문서 인제스트 실패 원인 저장 및 에러 상세 노출
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
지식베이스 문서 인제스트 파이프라인 수행 중 에러가 발생했을 때 실패 원인이 저장되지 않고 유실되어 사용자가 원인을 파악할 수 없는 결함이 존재한다.
`public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:128-132` (`IngestDocumentUseCase.execute`)에서는 텍스트 추출, 프롬프트 인젝션 차단 검사(`_rag_validator.inspect_input`), 임베딩 벡터 생성 및 Qdrant 색인 과정에서 예외가 발생하면 에러 로그만 남기고 `document.mark_failed()`를 호출한다.
그러나 `public-python-server/src/ai_service/knowledge/schemas.py:71-81` (`Document.mark_failed`)의 도메인 모델 구현은 상태를 단순히 `"failed"`, 청크 수를 `0`으로 설정할 뿐 에러 원인이나 상세 메시지를 전달받거나 보관하는 필드가 존재하지 않는다.
또한 `public-python-server/src/ai_service/knowledge/repository.py:99-105` (`DocumentRepository.update`) 역시 `status`와 `chunkCount`만 `$set`으로 업데이트하므로 MongoDB 저장소에 실패 원인이 영속화되지 않는다.
이로 인해 `public-front/src/api/ai.ts:115-121` (`DocumentItem`) 및 `public-front/src/components/AiService.tsx:1120-1164` (`AiService`)의 지식베이스 문서 목록 화면에서는 실패한 문서 행에 단순한 `failed` 상태 배지만 표시되며, 사용자는 해당 문서가 왜 색인되지 못했는지(예: 프롬프트 인젝션 차단, 손상된 파일, 빈 텍스트 추출 등)를 알 수 없어 수정하거나 대처할 방법이 없다.
이번 분기 집중 과제인 "답변을 믿을 수 있게 만든다"의 핵심은 근거 문서에 기반한 신뢰할 수 있는 RAG 답변을 제공하는 것이며, 답변의 근거가 될 지식베이스 문서가 왜 수집되지 못했는지 사용자에게 투명하게 안내하여 올바른 근거 문서를 구축할 수 있도록 지원해야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/knowledge/schemas.py`의 `DocumentProps`, `Document`, `DocumentOut`에 선택적 문자열 필드 `error_message`(직렬화 alias: `errorMessage`)를 추가한다.
- [ ] `Document.mark_failed` 메서드에 `error_message: str | None = None` 매개변수를 지원하여 실패 원인을 도메인 모델에 반영하고, `Document.mark_processed` 시에는 `error_message`가 제거(클리어)되도록 처리한다.
- [ ] `public-python-server/src/ai_service/knowledge/repository.py`의 `DocumentRepository.persist`, `update`, `_to_domain`에서 `errorMessage` 필드를 MongoDB에 안전하게 저장하고 역직렬화하여 복원한다. 기존 필드가 없는 레거시 데이터는 `None`으로 안전하게 폴백한다.
- [ ] `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`의 `IngestDocumentUseCase.execute`에서 예외 발생 시 예외 메시지(`str(exc)`)를 추출하여 `document.mark_failed(error_message=...)`에 전달하고 영속화한다.
- [ ] `public-front/src/api/ai.ts`의 `DocumentItem` 인터페이스 및 `public-front/src/components/AiService.tsx`의 `DocumentInfo` 인터페이스에 선택적 필드 `errorMessage?: string`을 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 지식베이스 문서 목록 테이블에서 실패 상태(`isFailed`)인 문서의 상태 배지에 마우스 호버 시 툴팁(`title={doc.errorMessage || '문서 인제스트 처리에 실패했습니다.'}`)으로 구체적인 실패 사유를 노출한다.
- [ ] `public-front/src/components/AiService.tsx`의 실패 문서 행에 실패 사유를 즉시 인지할 수 있는 시각적 안내(예: 에러 안내 문구 표시 또는 상세 확인 아이콘)를 제공한다.
- [ ] 백엔드 도메인/리포지토리/유스케이스 및 프론트엔드 문서 목록 렌더링에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 문서 파싱 라이브러리 교체 또는 지원 파일 포맷(PDF, TXT, MD 외)의 신규 파서 추가
- 프론트엔드 업로드 폼 사전 유효성 검증(SPEC-004) 및 단계별 진행 이벤트 규격(SPEC-034)의 변경
- 인제스트 실패 문서에 대한 백그라운드 자동 주기적 재시도 스케줄러 구현 (사용자 수동 [재시도] 버튼 연동 범위 유지)

## 백엔드
- `public-python-server/src/ai_service/knowledge/schemas.py`:
  - `DocumentProps` 데이터클래스에 `error_message: str | None = None` 필드 추가.
  - `Document.__init__` 및 `Document.restore`에서 `error_message` 속성을 바인딩.
  - `Document.mark_failed(self, error_message: str | None = None) -> "Document"` 메서드로 확장하여 실패 메시지를 유지.
  - `Document.mark_processed(self, chunk_count: int) -> "Document"` 실행 시 `error_message=None`으로 전달하여 이전 실패 사유 초기화.
  - `DocumentOut` Pydantic 모델에 `error_message: str | None = Field(default=None, alias="errorMessage")` 필드를 추가하고 `from_domain`에서 `doc.error_message` 매핑.
- `public-python-server/src/ai_service/knowledge/repository.py`:
  - `DocumentRepository.persist`에서 `record["errorMessage"] = document.error_message` 저장.
  - `DocumentRepository.update`에서 `$set` 필드에 `"errorMessage": document.error_message`를 포함하여 상태 갱신.
  - `DocumentRepository._to_domain`에서 `record.get("errorMessage")`를 `DocumentProps.error_message`로 복원.
- `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`:
  - `IngestDocumentUseCase.execute`의 `except Exception as exc:` 블록에서 `error_message = str(exc).strip() or "문서 인제스트 처리 중 오류가 발생했습니다."`를 구성하여 `failed = document.mark_failed(error_message)`로 갱신 후 리포지토리에 저장.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `DocumentItem` 인터페이스에 `errorMessage?: string;` 선택적 필드 추가.
- `public-front/src/components/AiService.tsx`:
  - `DocumentInfo` 인터페이스에 `errorMessage?: string;` 선택적 필드 추가.
  - 문서 목록 테이블 행 렌더링 시 `isFailed` 조건일 때 `status-badge failed` 요소에 `title={doc.errorMessage || '문서 인제스트 처리에 실패했습니다.'}` 속성을 부여하여 툴팁으로 실패 원인을 제공.
  - 실패 문서의 상태 셀 내부에 에러 원인을 명확히 식별할 수 있는 안내 텍스트나 인라인 배너/툴팁 힌트(`doc-error-hint`)를 렌더링.
  - [재시도] 중인 문서(`retryingDocId === doc.id`) 상태 및 재시도 성공 시 `errorMessage`가 정상적으로 제거되고 최신 상태가 반영되도록 처리.

## 수용 기준 (Acceptance Criteria)
- Given 프롬프트 인젝션 검증 실패("인제스트 차단: ...") 또는 파싱 에러로 인제스트가 실패한 문서가 존재할 때, When 문서 목록 조회 API(`GET /knowledge/documents`)를 호출하면, Then 응답의 `DocumentOut` 객체에 `status: "failed"` 및 구체적인 에러 메시지(`errorMessage`)가 포함되어 반환된다.
- Given `errorMessage`가 포함된 실패 문서가 화면에 표시될 때, When 사용자가 해당 행의 실패 상태 배지 또는 안내 영역을 확인할 때, Then 백엔드로부터 전달받은 실패 원인 상세 텍스트가 명확하게 노출된다.
- Given 실패했던 문서에 대해 [재시도](`handleRetryDocument`)를 수행하여 처리가 완료되었을 때, When 최신 문서 목록(`fetchDocuments`)을 갱신하면, Then 문서의 `status`는 `"processed"`로 변경되고 `errorMessage`는 존재하지 않거나 비어있는 상태로 정상 갱신된다.
- Given (경계 케이스) MongoDB에 `errorMessage` 필드가 없는 기존 레거시 실패 문서가 조회될 때, When 백엔드와 프론트엔드가 이를 역직렬화하고 화면에 렌더링할 때, Then 오류 없이 기본 폴백 메시지("문서 인제스트 처리에 실패했습니다.")가 화면에 안전하게 표시된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:128-132` (`IngestDocumentUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/knowledge/schemas.py:71-81` (`Document.mark_failed`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/knowledge/repository.py:99-105` (`DocumentRepository.update`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1120-1164` (`AiService`)
- 관련 정의: `public-front/src/api/ai.ts:115-121` (`DocumentItem`)
