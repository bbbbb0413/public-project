---
id: SPEC-113
title: 지식베이스 문서 재색인 실패 시 기존 벡터 청크 유실 방지 및 안전한 롤백 처리
status: done
targets: [python-server]
stages: [backend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스에 등록된 문서를 근거로 질문에 신뢰할 수 있는 답변을 제공하는 것이다. 이를 위해 문서 업로드 유효성 검증(SPEC-004), 단계별 인제스트 진행 이벤트(SPEC-034), 실패 문서 원본 기반 재인제스트 API(SPEC-087), 색인 청크 열람 모달(SPEC-083, SPEC-096, SPEC-111) 등을 지속해서 발전시켜 왔다.
그러나 현재 지식베이스 문서 재색인(인제스트 재시도) 처리 파이프라인에는 일시적인 네트워크 순단이나 Qdrant/임베딩 서비스 장애 발생 시 기존에 성공적으로 색인되어 있던 정상 벡터 청크까지 영구 유실(Data Loss)되는 치명적인 결함이 존재한다:
첫째, `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:121-133` (`IngestDocumentUseCase.execute`)에서는 신규 청크 색인을 수행하기 직전인 123행에서 `await self._vector_store.delete_by_document_id(document_id)`를 무조건 먼저 호출하여 Qdrant 벡터 컬렉션에 존재하는 대상 문서의 모든 청크를 일괄 삭제한다.
둘째, `public-python-server/src/ai_service/knowledge/repository.py:287-296` (`QdrantVectorAdapter.delete_by_document_id`)을 보면, 전달받은 `documentId` 필터 조건에 매칭되는 모든 포인트를 즉시 물리 삭제한다. 이후 124행의 `await self._vector_store.upsert(vector_docs)` 단계에서 Qdrant 일시 타임아웃, 커넥션 풀 고갈, 프로세스 강제 종료, 임베딩 장애 등 예외가 발생하면, 신규 청크는 적재되지 못한 채 기존 청크는 이미 삭제되어 벡터 DB상에서 해당 문서의 지식 청크가 완전히 증발한다.
셋째, `public-python-server/src/ai_service/knowledge/schemas.py:77-88` (`Document.mark_failed`)에서는 예외 발생 시 `status = "failed"`, `chunk_count = 0`으로 상태를 덮어씌운다. 이로 인해 `public-python-server/src/ai_service/knowledge/router.py:49-72` (`retry_document_ingest`) 엔드포인트를 통해 이미 정상 색인되어 있던 문서(`processed`)에 대해 재색인을 시도하다가 일시 장애가 발생했을 때, 기존에 잘 동작하던 RAG 검색에서 해당 문서가 완전히 배제되고 프론트엔드(`public-front/src/components/AiService.tsx:845-859`, `handleRetryDocument`)에서도 문서가 실패 상태로 바뀌어 버린다.
이는 시스템 동작에 되돌릴 수 없는 데이터 유실 안전장치가 부재한 결함이며, SPEC-087(재인제스트 지원)의 안정성을 파괴하고 사용자가 믿을 수 있는 RAG 지식 기반을 위협한다.
따라서 선삭제 후 적재 방식을 선적재 후 잉여 청크 정리(Upsert-then-Prune) 또는 안전한 원자적 교체 방식으로 전환하고, 재색인 실패 시 기존 정상 청크와 문서 상태를 롤백·보존하는 안전장치를 구현해야 한다.
## 요구사항
- [ ] `public-python-server/src/ai_service/knowledge/repository.py`의 `QdrantVectorAdapter`에 특정 문서의 기존 포인트 ID 목록을 조회하거나 신규 청크에 포함되지 않은 잉여 청크 포인트를 안전하게 삭제하는 메서드(예: `delete_points_by_ids` 또는 `prune_stale_chunks`)를 구현한다.
- [ ] `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`의 `IngestDocumentUseCase.execute`에서 신규 벡터 청크 적재 전 기존 청크를 전량 선삭제(`delete_by_document_id`)하던 취약한 처리 순서를 제거한다.
- [ ] `IngestDocumentUseCase.execute`에서 신규 청크(`vector_docs`)를 Qdrant에 먼저 성공적으로 `upsert`한 후, 기존에 존재하던 포인트 중 신규 청크에 포함되지 않은 구버전 잔여 포인트만 선별적으로 정리하도록 개선한다.
- [ ] 재색인 수행 전 기존 문서가 완료 상태(`document.status == "processed"`)였던 경우, 신규 텍스트 추출·임베딩·Qdrant 적재 과정에서 예외가 발생하더라도 기존 벡터 청크가 유실되지 않도록 보존한다.
- [ ] 기존 완료 문서(`status == "processed"`)의 재색인이 실패한 경우, 문서 전체를 `failed` 및 `chunkCount: 0`으로 파괴하지 않고, 기존 완료 상태(`status: "processed"`)와 이전 `chunkCount`를 안전하게 롤백 유지하면서 `errorMessage`에만 재시도 실패 사유를 기록한다.
- [ ] 최초 신규 문서 등록 시 인제스트가 실패한 경우에는 기존과 동일하게 `status = "failed"` 및 `chunk_count = 0`으로 정상 실패 처리된다.
- [ ] 재인제스트 시 신규 청크 수가 기존 청크 수보다 적은 경우(예: 10개 -> 4개), 신규 청크 4개가 갱신되고 이전의 6개 잔여 청크 포인트가 깨끗하게 정리되어 검색 중복이나 왜곡이 발생하지 않음을 보장한다.
- [ ] 재색인 도중 `upsert` 실패 시 기존 벡터 청크 보존, 잉여 청크 선별 삭제, 기존 완료 문서 상태 롤백을 검증하는 백엔드 단위/통합 테스트를 작성한다.
## 비요구사항 (Out of scope)
- Qdrant 컬렉션 자체의 스키마 변경, 인덱스 차원 변경, 또는 거리 함수(Cosine) 설정을 변경하지 않는다.
- 프론트엔드(`AiService.tsx`)의 문서 목록 테이블 UI나 재시도 버튼 이벤트 핸들러를 수정하지 않는다.
- 원본 파일 저장소(`DocumentRepository.save_original_file`)의 파일 암호화나 저장 경로를 변경하지 않는다.
- RAG 질의 처리 파이프라인(`AskUseCase`, `AgenticAskUseCase`)의 검색 가중치나 유사도 임계값을 변경하지 않는다.
## 백엔드
`public-python-server/src/ai_service/knowledge/repository.py`
- `QdrantVectorAdapter` 확장:
  - `find_point_ids_by_document_id(self, document_id: str) -> list[str]`: 특정 문서에 속한 모든 포인트의 Qdrant ID 목록을 스크롤 조회한다.
  - `delete_points_by_ids(self, point_ids: list[str]) -> None`: 지정된 포인트 ID 목록을 컬렉션에서 안전하게 일괄 삭제한다.
  - 또는 `prune_stale_chunks(self, document_id: str, active_point_ids: set[str]) -> None`: 해당 문서의 기존 포인트 중 `active_point_ids`에 포함되지 않은 잔여 포인트들만 선별 삭제한다.

`public-python-server/src/ai_service/knowledge/schemas.py`
- `Document` 도메인 모델에 재색인 실패 시 안전한 롤백을 지원하는 헬퍼 메서드 추가(예: `mark_reindex_failed(self, error_message: str)`).
  - 기존 `status == "processed"`인 경우 `status`와 `chunk_count`를 유지하고 `error_message`만 갱신.
  - 기존 상태가 `processed`가 아니었던 경우 기존 `mark_failed`와 동일하게 `status = "failed"`, `chunk_count = 0` 처리.

`public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`
- `IngestDocumentUseCase.execute` 흐름 개선:
  - 인제스트 시작 전 기존 문서의 상태(`was_processed = (document.status == "processed")`, `prev_chunk_count = document.chunk_count`)를 기록.
  - 123행의 사전 `delete_by_document_id(document_id)` 제거.
  - 기존 문서에 속해 있던 기존 포인트 ID 목록을 사전에 확보: `existing_point_ids = await self._vector_store.find_point_ids_by_document_id(document_id)`.
  - 신규 `vector_docs`를 `await self._vector_store.upsert(vector_docs)`로 먼저 안전하게 저장.
  - 저장 성공 후 신규 포인트 ID 세트(`new_point_ids = {_to_point_id(d.id) for d in vector_docs}`)를 생성하고, 기존 포인트 중 신규에 없는 잉여 포인트 ID(`stale_ids = [pid for pid in existing_point_ids if pid not in new_point_ids]`)를 `await self._vector_store.delete_points_by_ids(stale_ids)`로 정리.
  - `except Exception as exc` 발생 시:
    - `was_processed`가 참이면 기존 청크가 보존되어 있으므로 `document.mark_reindex_failed(error_message)`를 호출하여 기존 문서의 `processed` 상태와 이전 `chunk_count`를 보존.
    - 최초 생성이거나 미완료 문서였던 경우 기존대로 `document.mark_failed(error_message)` 호출.
## 수용 기준 (Acceptance Criteria)
- Given 이미 정상 색인되어 청크 5개가 Qdrant에 저장된 완료 문서(`status == "processed"`, `chunkCount == 5`)가 존재할 때
  When 사용자가 해당 문서에 대해 재인제스트를 요청하였으나 Qdrant `upsert` 도중 네트워크 타임아웃 예외가 발생하면
  Then Qdrant에서 기존 5개 청크가 삭제되지 않고 온전히 보존되며, 문서의 `status`는 `processed`, `chunkCount`는 5로 유지되고 RAG 질의에서 해당 문서 검색이 계속 정상 동작한다.
- Given 완료 문서(`chunkCount == 8`)에 대해 텍스트가 줄어들어 신규 청크 3개로 재색인 요청이 성공적으로 완료되었을 때
  When 인제스트 실행이 종료되면
  Then 신규 3개 청크가 정상 적재되고 구버전 잉여 청크 5개는 Qdrant에서 완전히 삭제되어, 총 3개의 청크만 검색 대상이 된다.
- Given 기존에 등록된 적 없는 신규 문서에 대해 최초 인제스트를 수행하던 중 임베딩 생성 오류가 발생했을 때 (경계 케이스)
  When 인제스트가 실패하면
  Then 문서는 정상적으로 `status = "failed"`, `chunk_count = 0`으로 저장되고 발생한 상세 에러 메시지가 `errorMessage`에 기록된다.
- Given 기존 완료 문서의 재색인 도중 텍스트 추출(`_extract_text`)이나 입력 검증(`inspect_input`) 단계에서 실패한 경우 (경계 케이스)
  When 인제스트가 중단되면
  Then Qdrant 포인트에 대한 변경이나 삭제 없이 기존 청크 5개와 `processed` 상태가 안전하게 유지된다.
## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:121-133` (`IngestDocumentUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/knowledge/repository.py:287-296` (`QdrantVectorAdapter.delete_by_document_id`)
- 관련 스키마 정의: `public-python-server/src/ai_service/knowledge/schemas.py:77-88` (`Document.mark_failed`)
- 재시도 입구 라우터: `public-python-server/src/ai_service/knowledge/router.py:49-72` (`retry_document_ingest`)
- 프론트 재시도 호출부: `public-front/src/components/AiService.tsx:845-859` (`handleRetryDocument`)