---
id: SPEC-034
title: 지식베이스 문서 인제스트 파이프라인 단계별 진행 이벤트 발행 및 진행률 표시 연동
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제

문서 업로드 후 RAG 지식베이스에 색인되는 과정은 파일 크기에 따라 텍스트 추출, 청크 분할, 임베딩 생성(외부 LLM/임베딩 API 호출), 벡터 DB 색인까지 수 초에서 수십 초가 소요된다.

1. `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82-99` (`_process`)에서 인제스트 작업 수행 시 `ingest_use_case.execute`를 단일 동기 루틴으로 호출하고, 중간 과정 없이 완료 시 `publish_done` 또는 실패 시 `publish_error` 이벤트만 발행한다.
2. `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:86-122` (`execute`)에서 텍스트 추출(`_extract_text`), 청크 분할(`_split_by_paragraph_first`), 임베딩 생성(`_embedding_provider.embed`), 벡터 DB 색인(`_vector_store.upsert`) 단계가 순차적으로 실행되지만 진행 상황을 알리는 콜백이나 이벤트 발행 메커니즘이 없다.
3. 반면 프론트엔드 `public-front/src/components/AiService.tsx:52-57` (`STEP_LABELS`), `public-front/src/components/AiService.tsx:368-374` (`subscribeIngestJob.onProgress`), `public-front/src/api/ai.ts:143-171` (`subscribeIngestJob`) 및 `public-server/apps/gateway/src/ai/job/job-store.service.ts:9-19` (`AiJobStep`, `AiJobMeta`)에는 이미 `extract`, `chunk`, `embed`, `index` 4단계 진행률을 수신하여 `uploadProgressText`에 표시하는 구조가 준비되어 있으나, 백엔드에서 `progress` 이벤트를 발행하지 않아 사용자는 인제스트가 완료될 때까지 단순 "업로드 중..." 또는 멈춘 상태로 체감하게 된다.

이로 인해 대용량 문서 업로드 시 처리가 정상 진행 중인지, 시스템이 멈춘 것인지 사용자가 알 수 없어 중복 업로드를 시도하거나 이탈하는 문제가 발생한다.

## 요구사항

- [ ] `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`의 `execute` 메서드에 단계별 진행 콜백(예: `on_progress: Callable[[str, int], Awaitable[None]] | None`) 매개변수를 지원한다.
- [ ] 문서 인제스트 수행 중 4개 단계(`extract`, `chunk`, `embed`, `index`)에 진입하거나 완료할 때 `step`과 `progress`(0~100 정수 퍼센트) 정보를 콜백으로 전달한다.
  - `extract` (텍스트 추출 시작 / 진행: 10% ~ 25%)
  - `chunk` (문서 청킹 및 전처리: 30% ~ 50%)
  - `embed` (임베딩 벡터 생성: 55% ~ 85%)
  - `index` (벡터 DB 색인 및 저장: 90% ~ 99%)
- [ ] `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py`의 `_process`에서 `JobEventPublisher.publish_progress(job_id, {"step": step, "progress": progress})`를 호출하여 Redis Streams `ai:job:{jobId}:events`에 `type: "progress"` 이벤트를 실시간 발행한다.
- [ ] 인제스트 처리 중 오류 발생 시 기존과 동일하게 `publish_error`가 발행되고 문서 상태가 `failed`로 전이된다.
- [ ] 프론트엔드 `public-front/src/components/AiService.tsx`에서 SSE `progress` 이벤트를 수신하여 업로드 버튼 및 진행 텍스트에 "텍스트 추출 중 (10%)", "청크 분할 중 (30%)", "임베딩 생성 중 (60%)", "색인 저장 중 (90%)" 등 단계명과 퍼센트가 실시간으로 갱신되어 표시된다.
- [ ] 인제스트 완료 시 `done` 이벤트 수신 후 기존과 같이 문서 목록을 새로고침하고 진행 표시가 초기화된다.

## 비요구사항 (Out of scope)

- 문서 인제스트 실패 시 전용 재시도(Retry) REST API 엔드포인트 신설 (기존 업로드 폼 재시도 UX 유지).
- 지원 파일 형식 확장 (TXT, PDF, MD 유지).
- Gateway `JobStoreService`의 Redis 잡 상태 폴링 API 구조 변경 (기존 SSE 스트림 `GET /ai/jobs/:jobId/stream` 이벤트 릴레이 메커니즘 활용).

## 백엔드

- `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`:
  - `execute(command: IngestDocumentCommand, on_progress: Callable[[str, int], Awaitable[None]] | None = None)` 시그니처 확장.
  - `_extract_text` 전후: `await on_progress("extract", 20)`
  - `_build_vector_docs` 청킹 단계: `await on_progress("chunk", 45)`
  - 임베딩 생성 루프/호출: `await on_progress("embed", 70)`
  - `_vector_store.upsert` 색인 단계: `await on_progress("index", 90)`
- `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py`:
  - `_process` 내에서 진행 콜백을 정의하여 `await self._publisher.publish_progress(message.job_id, {"step": step, "progress": progress})` 호출.

## 프론트엔드

- `public-front/src/components/AiService.tsx`:
  - `handleFileUpload`의 `subscribeIngestJob` `onProgress` 핸들러에서 수신한 `step`과 `progress`를 바탕으로 `setUploadProgressText` 갱신 (`STEP_LABELS[data.step] (data.progress%)`).
  - 업로드 완료 또는 에러 시 `uploadProgressText` 및 `uploading` 상태 안전 초기화.

## 수용 기준 (Acceptance Criteria)

- Given 문서 파일(TXT, PDF, MD)을 업로드할 때
  When 백엔드 인제스트 작업이 시작되면
  Then SSE 스트림(`ai:job:{jobId}:events`)을 통해 `step`(`extract`, `chunk`, `embed`, `index`)과 `progress` 값이 포함된 `type: "progress"` 이벤트가 순차적으로 발행된다.
- Given 프론트엔드에서 문서 업로드를 진행 중일 때
  When `progress` 이벤트가 수신되면
  Then 업로드 버튼의 텍스트가 해당 단계 한글 라벨과 진행률(예: "임베딩 생성 중 (70%)")로 실시간 갱신된다.
- Given 지원되지 않거나 손상된 파일로 인해 인제스트 도중 예외가 발생할 때
  When 에러가 발생하면
  Then `type: "error"` 이벤트가 정상 발행되고 프론트엔드에 에러 배너가 표시되며 진행 표시 상태가 정리된다.
- Given 진행률 이벤트의 `progress` 필드가 0 또는 누락된 경우(경계 조건)
  When `progress` 이벤트를 수신하면
  Then 에러 없이 단계 라벨(예: "텍스트 추출 중")만 정상 표시된다.

## 참고

- 백엔드 이벤트 발행 누락 위치: `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82-99` (`_process`)
- 백엔드 인제스트 처리 로직: `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:86-122` (`execute`)
- 백엔드 이벤트 발행기: `public-python-server/src/ai_service/core/events.py:28-32` (`publish_progress`)
- 프론트엔드 단계 라벨 매핑: `public-front/src/components/AiService.tsx:52-57` (`STEP_LABELS`)
- 프론트엔드 진행 이벤트 핸들러: `public-front/src/components/AiService.tsx:368-374` (`subscribeIngestJob`)
- 프론트엔드 스트림 구독 클라이언트: `public-front/src/api/ai.ts:143-171` (`subscribeIngestJob`)
