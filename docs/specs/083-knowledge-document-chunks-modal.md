---
id: SPEC-083
title: 지식베이스 색인 청크 목록 조회 API 연동 및 청크 열람 모달
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 지식베이스 문서 목록 테이블 `public-front/src/components/AiService.tsx:913` (`AiService`)에서 각 문서의 청크 수(`doc.chunkCount`)가 단순 텍스트로만 렌더링되고 있다. 사용자는 자신이 등록한 문서가 RAG 엔진에 의해 실제로 어떤 단위와 내용으로 분할(chunking)되어 벡터 저장소에 색인되었는지 확인할 수 없다.

Python AI 서비스에는 이미 특정 문서의 청크 인덱스와 텍스트 본문을 조회하는 `public-python-server/src/ai_service/knowledge/router.py:26-29` (`list_chunks`) 엔드포인트가 `public-python-server/src/ai_service/knowledge/schemas.py:174-178` (`ChunkOut`) 스키마와 함께 구현되어 있다. 게이트웨이 역시 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:34-38` (`KnowledgeProxyController.listChunks`)를 통해 `GET /ai/knowledge/documents/:id/chunks` 엔드포인트 프록시를 완전히 제공하고 있다.

그러나 프론트엔드 API 클라이언트 `public-front/src/api/ai.ts:417-422` (`getDocumentFile`) 주변에는 해당 청크 목록 API를 호출하는 함수가 누락되어 있고, `public-front/src/components/AiService.tsx:916-928` (`AiService`)의 문서 작업(`doc-actions`) 버튼 영역에도 청크를 열람하는 진입점이 없다. 이로 인해 이미 완성된 백엔드 기능이 방치되어 있고, 사용자가 답변의 근거가 되는 지식 문서를 스스로 점검할 수 없어 "답변을 믿을 수 있게 만든다"는 제품 핵심 방향을 충족하지 못하고 있다.

## 요구사항
- [ ] `public-front/src/api/ai.ts`에 청크 데이터 규격을 정의하는 `DocumentChunk` 인터페이스(`chunkIndex: number; text: string;`)를 선언하고 내보낸다.
- [ ] `public-front/src/api/ai.ts`에 `getDocumentChunks(documentId: string): Promise<DocumentChunk[]>` API 클라이언트 함수를 구현한다 (`GET /ai/knowledge/documents/:documentId/chunks`).
- [ ] `public-front/src/components/AiService.tsx`의 지식베이스 문서 목록 테이블에서 완료 상태(`doc.status === 'processed' || doc.status === 'completed'`)인 문서 행의 청크 수(`doc.chunkCount`)를 클릭 가능한 버튼(`btn-chunk-count`)으로 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`의 완료 상태 문서 행 작업(`doc-actions`) 영역에 [청크 보기] 버튼(`btn-view-chunks`)을 추가한다.
- [ ] 청크 수 버튼 또는 [청크 보기] 버튼 클릭 시 대상 문서의 청크 목록을 비동기 조회하여 표시하는 청크 열람 모달(`chunks-modal`)을 연다.
- [ ] 청크 모달 헤더에 대상 문서의 파일명(예: `"{fileName}" 청크 목록`)과 전체 청크 수(예: `총 {count}개`)를 표시한다.
- [ ] 청크 API 호출 진행 중에는 모달 본문에 로딩 피드백("청크 목록을 불러오는 중...")을 표시한다.
- [ ] 청크 API 호출 실패 시 모달 내에 에러 메시지("청크 목록을 불러오지 못했습니다.")를 표시한다.
- [ ] 청크 데이터 로드 완료 시 스크롤 가능한 목록 영역에 각 청크의 인덱스(예: `청크 #{chunkIndex + 1}`)와 본문 텍스트(`text`)를 구분 카드 형태로 렌더링한다.
- [ ] 색인된 청크가 0건인 경우(`chunks.length === 0`), "색인된 청크가 없습니다." 안내 문구를 표시한다.
- [ ] 청크 모달 상단 닫기 버튼(`✕`), 하단 [닫기] 버튼, 모달 바깥 배경(오버레이) 클릭, 키보드 `Escape` 키 입력 시 모달을 닫고 상태를 초기화한다.
- [ ] 인제스트 진행 중(`pending`, `processing`)이거나 실패(`failed`) 상태인 문서 행에는 [청크 보기] 버튼을 노출하지 않으며 청크 수를 일반 텍스트로 렌더링한다.
- [ ] 청크 목록 조회 및 모달 컴포넌트 렌더링, 닫기 인터랙션, 로딩·에러·빈 목록 처리를 검증하는 프론트엔드 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 개별 청크의 텍스트 수정, 청크 단건 삭제, 또는 신규 청크 수동 추가 기능은 제공하지 않는다. 청크 데이터는 원본 문서 인제스트 파이프라인에 의해서만 생성·관리된다.
- 백엔드 벡터 스토어의 청킹 알고리즘(청크 크기, 오버랩 비율) 변경이나 재인제스트 파이프라인 수정은 포함하지 않는다.
- 실패 문서의 원본 파일 재인제스트 로직 개선은 별도 작업으로 분리하며 이번 범위에 포함하지 않는다.
- 청크별 임베딩 벡터 수치 배열(float[])이나 Qdrant 내부 포인트 ID 등 불필요한 메타데이터는 화면에 노출하지 않는다.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `DocumentChunk` 인터페이스 선언 (`chunkIndex: number`, `text: string`)
  - `getDocumentChunks` 함수 추가
- `public-front/src/components/AiService.tsx`:
  - 청크 모달 열림 상태(`viewingChunksDoc: DocumentInfo | null`), 청크 목록 상태(`chunks: DocumentChunk[]`), 로딩 상태(`isLoadingChunks: boolean`), 에러 상태(`chunksError: string | null`) 관리
  - 완료 문서 행 청크 수 셀에 `btn-chunk-count` 버튼 바인딩 및 작업 영역에 `btn-view-chunks` 버튼 추가
  - 청크 목록 모달 UI(`chunks-modal-overlay`, `chunks-modal`, `chunk-list`, `chunk-item`) 렌더링 및 `Escape` 키 핸들러 연동
- `public-front/src/components/AiService.css`:
  - 청크 열람 모달 스타일, 스크롤 가능한 청크 리스트, 청크 번호 배지 및 텍스트 본문 pre 스타일 정의

## 수용 기준 (Acceptance Criteria)
- Given 완료(`processed` 또는 `completed`) 상태이고 3개의 청크를 가진 문서가 지식베이스 목록에 있을 때, When 사용자가 해당 행의 [청크 보기] 버튼을 클릭하면, Then `getDocumentChunks`가 호출되고 모달에 대상 파일명과 함께 3건의 청크 인덱스(`청크 #1`, `청크 #2`, `청크 #3`) 및 본문 텍스트가 순서대로 렌더링된다.
- Given 청크 목록 모달이 열려 있을 때, When 사용자가 모달 닫기 버튼을 클릭하거나 배경 오버레이를 클릭하거나 `Escape` 키를 누르면, Then 모달이 닫히고 문서 목록 화면으로 정상 복귀한다.
- Given 완료 상태이지만 생성된 청크가 0건인 문서(`chunkCount === 0`)가 있을 때, When 사용자가 [청크 보기] 버튼을 클릭하면, Then 청크 목록 대신 "색인된 청크가 없습니다." 안내 문구가 모달 내에 표시된다. (경계 케이스)
- Given 인제스트 진행 중(`pending`, `processing`)이거나 실패(`failed`) 상태인 문서가 있을 때, When 문서 목록 테이블을 확인하면, Then 해당 행에는 [청크 보기] 버튼이 렌더링되지 않고 청크 수는 클릭 불가능한 일반 텍스트로 표시된다. (경계 케이스)
- Given 청크 목록 API 호출 중 네트워크 오류나 500 에러가 발생했을 때, When 사용자가 [청크 보기] 버튼을 클릭하면, Then 모달 내에 "청크 목록을 불러오지 못했습니다." 에러 메시지가 표시되고 앱이 정상 상태를 유지한다. (경계 케이스)

## 참고
- 게이트웨이 청크 프록시 엔드포인트: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:34-38` (`KnowledgeProxyController.listChunks`)
- 백엔드 청크 조회 라우터: `public-python-server/src/ai_service/knowledge/router.py:26-29` (`list_chunks`)
- 백엔드 청크 응답 스키마: `public-python-server/src/ai_service/knowledge/schemas.py:174-178` (`ChunkOut`)
- 프론트엔드 문서 목록 테이블: `public-front/src/components/AiService.tsx:913-928` (`AiService`)
- 프론트엔드 기존 문서 파일 다운로드 API: `public-front/src/api/ai.ts:417-422` (`getDocumentFile`)
