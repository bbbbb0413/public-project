---
id: SPEC-025
title: 문서 업로드 및 인제스트 — 용량 제한 정정, 단계별 진행 이벤트, 재시도 진입점
status: draft
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 출처
명세서 "2. 지식베이스 문서 관리" 중 **2.1 문서 업로드 및 인제스트**(2.1.1 문서 업로드 접수, 2.1.2 인제스트 상태 조회).

## 현재 구현 상태 (조사 결과)

| 항목 | 상태 | 근거 |
|---|---|---|
| 업로드 API + 형식/용량 검증 | 🟡 부분구현(용량 불일치) | `apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:31` `MAX_FILE_SIZE_BYTES = 10MB`. 허용 MIME은 TXT/PDF/MD(`ALLOWED_MIME_TYPES`, 같은 파일 32행). 프론트도 확장자 이중 검증(`public-front/src/components/AiService.tsx:242-247`) |
| 업로드 즉시 job 반환 | ✅ 구현됨 | `createIngestJob`(`knowledge-job.controller.ts:39-81`) — Redis에 job 생성, Kafka `ai.knowledge.ingest.requested` 발행, `{jobId}` 202 즉시 반환 |
| 추출 → 청크분할 → 임베딩 → 색인 파이프라인 | ✅ 구현됨(단, 중간 이벤트 없음) | `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py`의 `execute()` — parent/child 2단계 청크 분할까지 포함해 명세서보다 정교함. 다만 전체가 **한 번의 동기 호출**로 처리되고 중간 단계별 이벤트를 전혀 발행하지 않음 |
| 진행률/완료/실패 조회 | 🟡 부분구현(퍼센트·단계 없음) | `ingest_requested_consumer.py:77-91` — 실제 발행 이벤트는 성공 `publish_done`(documentId, chunkCount) / 실패 `publish_error` **두 가지뿐**. `JobStoreService` 상태 enum도 `queued\|processing\|done\|error\|cancelled` 5가지뿐(`apps/gateway/src/ai/job/job-store.service.ts:6-15`), 퍼센트 필드 없음. SSE 전달 경로는 [[022-rag-streaming-qa]]와 동일한 `/ai/jobs/:jobId/stream`를 재사용하며 Last-Event-ID 재생도 서버 쪽엔 이미 있음 |
| 실패 시 재시도 진입점 | ❌ 미구현 | `AiService.tsx:284-286` — `onError` 시 에러 메시지만 표시, 전용 재시도 버튼/API 없음. 처음부터 파일을 다시 선택해야 함 |
| 프론트 업로드 UI | ✅ 구현됨(타입 갭 있음) | 업로드 input, 업로드 중 상태, 15초 타임아웃 후 목록 재조회 폴백, 문서별 상태 배지(pending/processed/failed, `AiService.tsx:523`). 단 `getDocuments()`(`public-front/src/api/ai.ts:5-8`)가 응답 타입을 정의하지 않아 목록 스키마가 코드상 암묵적 `any` |

## 명세서와 현재 구현 간 상충 / 차이
1. **업로드 용량 제한 불일치**: 코드 10MB vs 명세서 50MB. 명세서대로 올릴지, 지금 10MB가 의도된 제약(LLM 컨텍스트/처리 시간 고려)인지 착수 전에 확인 필요 — 단순히 상수만 바꾸면 되는 문제가 아니라 청크 분할·임베딩 처리 시간에도 영향을 줄 수 있다.
2. **단계별 진행률 표시는 프론트만 고쳐서 안 됨**: 명세서가 요구하는 "추출/청크분할/임베딩/색인" 세분화 진행 상태와 퍼센트는 데이터 자체가 파이썬 파이프라인에 없다. `ingest_document_use_case.py`가 각 단계 사이에 진행 이벤트를 발행하도록 바꾸는 백엔드 작업이 선행돼야, 그 다음에 gateway SSE·프론트 표시를 붙일 수 있다. 이 문서의 항목 중 가장 손이 많이 가는 부분이다.
3. **재시도는 "새 업로드"로 우회 가능하지만 명세서가 말하는 "재시도 진입점"은 아님**: 실패한 job을 이어서 재시도(같은 문서로 재인제스트)하는 API가 없어서, 사용자는 실패한 파일을 처음부터 다시 올려야 한다. 별도 재시도 API를 만들지, 그냥 "다시 업로드하세요" UX로 충분한지 결정 필요.

## 요구사항 (이번 작업 범위)
1. 업로드 용량 제한을 명세서 기준(50MB)으로 올릴지 결정하고 반영한다(gateway `MAX_FILE_SIZE_BYTES` + 프론트 안내 문구).
2. `ingest_document_use_case.py`에 단계별(추출/청크분할/임베딩/색인) 진행 이벤트 발행을 추가하고, `JobStoreService` 상태 모델에 퍼센트 또는 현재 단계 필드를 추가한다.
3. 프론트 문서 목록/업로드 상태 UI를 단계별 진행 표시로 갱신한다.
4. 실패한 인제스트 작업에 대한 재시도 API(또는 "다시 업로드" 유도 UX 확정)를 추가한다.
5. `src/api/ai.ts`의 `getDocuments()` 응답에 명시적 타입을 붙인다.

## 비요구사항 (Out of scope)
- 문서 원문 편집/공동 편집 — 명세서가 명시적으로 제외.
- OCR 등 이미지 기반 문서 지원 확장 — 현재 TXT/PDF/MD만 지원, 이번 범위에서 형식 확장은 다루지 않는다.

## 구현 기록
(구현 완료 후 작성)
