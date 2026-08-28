---
id: SPEC-023
title: 답변 근거 확인 — 원문 위치 이동, 관련도 순위
status: done
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 출처
명세서 "1. AI 질의응답 경험" 중 **1.2 답변 근거 확인**. 스트리밍 자체는 [[022-rag-streaming-qa]], 평가/피드백은 [[024-answer-feedback]] 참고.

## 배경
근거 문서 목록 표시 자체는 이미 완료된 [[006-source-snippet|SPEC-006]]의 산출물로 존재한다. 이번 명세서는 그보다 한 걸음 더 나아가 "원문 위치로 이동"과 "관련도 순위"를 요구하는데, 이 둘은 SPEC-006 작성 당시 명시적으로 비요구사항으로 뺐던 항목이라 정면으로 상충한다.

## 현재 구현 상태 (조사 결과)

| 항목 | 상태 | 근거 |
|---|---|---|
| 근거 목록(문서명 + 인용 구간) 조회 | ✅ 구현됨 | `public-python-server/src/ai_service/rag/application/ask_use_case.py:97-104` — `__SOURCES:` SSE 이벤트로 `fileName`/`chunkIndex`/`documentId`/`snippet`(300자, 마스킹 적용) 전달. 프론트 `public-front/src/api/ai.ts:150-154`(`SourceRef` 타입), `AiService.tsx`의 펼치기/접기 UI가 소비 |
| 근거 항목 선택 → 원문 위치로 이동 | ❌ 미구현 | [[006-source-snippet|SPEC-006]]이 "원본 문서 전문을 여는 뷰어나 다운로드 기능"을 명시적으로 비요구사항 처리함 |
| 관련도 순위 표시 | ❌ 미구현 | 데이터 모델(`SourceRef`)에 관련도 점수 필드 자체가 없음. 벡터 검색 단계에서 유사도 점수는 내부적으로 계산되지만 응답 페이로드에 실려나가지 않는 것으로 보임(원문 위치 이동 기능과 함께 재확인 필요) |

## 명세서와 현재 구현 간 상충 / 차이
1. **[[006-source-snippet|SPEC-006]]과 정면 상충**: SPEC-006은 "원문 전체를 열람하는 뷰어는 만들지 않는다"고 명시했는데, 이번 명세서 1.2는 정확히 그 기능("근거 항목을 선택하면 해당 문서의 인용 위치를 연다")을 요구한다. 두 문서 중 어느 쪽 결정이 최신인지 착수 전에 확인해야 한다 — 이 문서는 이번 명세서가 우선한다고 가정하고 작성했다.
2. **원문 뷰어가 없으면 "인용 위치로 이동"이 불가능**: 지금은 청크 스니펫만 저장/전달하고 원문 파일 자체를 다시 열람하는 경로(파일 저장소 접근, 뷰어 컴포넌트)가 없다. 이 기능을 하려면 최소한 "원문 파일을 다운로드/새 탭에서 열기 + 해당 청크의 대략적 위치(페이지 번호 또는 문자 오프셋) 안내" 수준까지는 백엔드에서 원문 접근 경로를 새로 열어줘야 한다.
3. **관련도 점수는 파이프라인 내부엔 있을 가능성이 높음**: Qdrant 벡터 검색은 통상 유사도 점수를 반환하므로, 이미 계산되는 값을 API 응답에 노출만 하면 될 수도 있다 — 착수 시 `ask_use_case.py`의 검색 호출부에서 점수가 버려지고 있는지 먼저 확인.

## 요구사항 (이번 작업 범위)
1. SPEC-006과의 상충을 사용자와 먼저 확정한다(원문 열람 기능을 지금 만들 것인지).
2. (만들기로 하면) 문서 원문에 접근할 수 있는 최소 경로를 백엔드에 추가한다 — 업로드된 원본 파일을 소유자만 다운로드/열람할 수 있는 API.
3. 근거 응답(`SourceRef`)에 관련도 점수 필드를 추가하고, 검색 단계에서 이미 계산되는 유사도 점수를 그대로 실어보낸다.
4. 프론트 근거 목록 UI에 관련도 순 정렬 및 순위 표시, "원문 보기" 진입점을 추가한다.

## 비요구사항 (Out of scope)
- 원문 파일의 인라인 하이라이트/정밀 위치 스크롤(페이지 번호 수준의 안내로 충분, 문자 단위 정밀 이동은 다루지 않는다).
- 답변 평가/피드백 — [[024-answer-feedback]].

## 구현 기록

**SPEC-006과의 상충은 이번 명세서 우선으로 해결**: 원문 열람 기능을 새로 만들었다. 원본 파일 저장소는 사용자가 MongoDB GridFS를 지정.

### 백엔드 (python-server)
- `DocumentRepository`에 GridFS 기반 원본 파일 저장소 추가(`AsyncIOMotorGridFSBucket`, 버킷명 `knowledge_files`, `document_id`를 GridFS `filename`으로 사용). `save_original_file` / `get_original_file` / `_delete_original_file` 신설, `remove()`가 문서 삭제 시 원본도 함께 지움.
- `IngestDocumentUseCase.execute()`가 텍스트 추출/청킹 이전에 업로드 원본 바이트를 무조건 GridFS에 먼저 저장(추출/처리가 이후 실패해도 원본은 남도록 `try` 블록 밖에 배치). 재인제스트 시 기존 원본을 지우고 새로 저장.
- 신규 엔드포인트 `GET /knowledge/documents/{document_id}/file` — 원본이 없으면 404, 있으면 `Content-Disposition: inline`으로 바이너리 그대로 반환.
- `ask_use_case.py`(단순 경로)와 `agentic_ask_use_case.py`(에이전틱 경로) 양쪽의 sources 페이로드에 `score` 필드 추가 — `SimilaritySearchResult.score`는 이미 RRF 융합/리랭커를 거치며 정렬된 채로 존재했으므로 그대로 실어보내기만 하면 됐다(파이프라인 재작업 불필요).
- 테스트: `tests/unit/knowledge/test_document_repository.py` 신규 6건(저장/재저장/조회/메타데이터 fallback/삭제). Motor의 `AsyncIOMotorGridFSBucket`이 mock DB를 거부하는 문제는 `DocumentRepository.__new__`로 `__init__`을 우회하고 `_collection`/`_gridfs`를 직접 mock 주입해 해결. 전체 80/80 통과, ruff/mypy clean.

### 게이트웨이 (NestJS)
- `AiServicePyHttpService`에 `getBinary()` 추가(`responseType: 'arraybuffer'`로 프록시, `content-type`/`content-disposition` 헤더 그대로 전달) 및 바이너리 응답 전용 예외 변환기(`toBinaryHttpException` — 에러 바디도 ArrayBuffer로 오므로 JSON 디코드 시도 후 실패 시 메시지로 폴백).
- `KnowledgeProxyController`에 `GET :id/file` 라우트 추가, 원본 바이트를 그대로 스트리밍.
- 테스트: `ai-service-py-http.service.spec.ts`, `knowledge-proxy.controller.spec.ts` 신규(총 45/45 gateway 유닛 테스트 통과). 작성 중 `Buffer.from('hello world').buffer`가 Node의 문자열 풀링 할당 때문에 실제 문자열 바이트보다 훨씬 큰 공유 `ArrayBuffer`를 반환한다는 걸 발견 — 정확한 크기의 버퍼가 필요한 mock에는 `new TextEncoder().encode(str).buffer`를 써야 함(다른 테스트는 이미 이 패턴을 쓰고 있었음).

### 프론트엔드
- `SourceRef`에 `score?: number` 추가, `getDocumentFile(documentId)` API(blob 응답) 신설.
- `AiService.tsx`: 근거 목록을 `score` 내림차순으로 정렬해 렌더링하고 각 항목에 "관련도 NN%" 배지를 표시. 각 근거 항목에 "원문 보기" 버튼을 추가 — 클릭 시 `getDocumentFile`로 blob을 받아 `URL.createObjectURL` + `window.open`으로 새 탭에 연다(블롭 URL은 60초 후 `revokeObjectURL`로 정리).
- 테스트 2건 추가(관련도 순 정렬/배지 표시, "원문 보기" 클릭 시 API 호출 및 새 탭 오픈 검증). 프론트 전체 156/156 통과.

### 배포
- `ai-service-py`, `gateway`, `frontend` 이미지 재빌드 후 재기동, 부팅 로그로 `/ai/knowledge/documents/:id/file` 라우트 등록 확인.

### 비요구사항 처리
- 페이지 번호/문자 오프셋 수준의 정밀 위치 안내는 이번 범위에서 제외했다 — "원문 보기"는 파일 전체를 새 탭에서 여는 수준까지만 구현. 필요해지면 후속 SPEC에서 다룬다.
