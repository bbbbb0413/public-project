---
id: SPEC-106
title: AI 답변 참고 문서(출처)의 개별 청크 모달 위치 이동·강조 연동 및 출처 청크 번호 오름차순 정렬
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 올려 둔 문서를 근거로 질문에 답하며, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 것이다. 현재 시스템은 AI 답변 버블 하단에 답변 생성에 활용된 지식 문서와 청크 목록을 보여주는 참고 문서(출처, `source-refs`) 영역(SPEC-006, SPEC-042)을 제공하며, SPEC-096을 통해 출처 그룹별 [청크 목록] 버튼(`btn-view-doc-chunks`)을 눌러 색인 청크 열람 모달(`chunks-modal`)을 조회할 수 있도록 연계하였다.
그러나 사용자가 답변의 근거를 직접 대조하고 확인하는 실제 사용 흐름에서 여전히 다음과 같은 심각한 불편과 탐색 단절이 존재한다:
첫째, `public-front/src/components/AiService.tsx:382-409` (`groupSourcesByDocument`)에서 각 문서 그룹 내 청크들을 점수 내림차순(`(b.score ?? -Infinity) - (a.score ?? -Infinity)`)으로 정렬하고 있다. 이로 인해 문서 내 청크 번호가 `청크 #5`, `청크 #1`, `청크 #3`과 같이 뒤죽박죽 섞인 순서로 나열되어, 사용자가 문서의 흐름과 맥락에 맞춰 근거를 순차적으로 파악하기 어렵다. 문서 그룹 자체는 최고 관련도(`maxScore`) 내림차순으로 배치하되, 문서 그룹 내부의 청크들은 청크 번호(`chunkIndex`) 오름차순으로 정렬되어야 자연스러운 문서 독해가 가능하다.
둘째, `public-front/src/components/AiService.tsx:1960-1983` (`AiService` 내 `source-group-chunks`) 및 `public-front/src/components/AiService.tsx:2132-2155` (`AiService` 내 스트리밍 참고 문서 렌더링)에서 각 청크는 최대 300자로 축약된 `snippet`만 제공한다. 축약된 스니펫만으로는 구체적인 사실관계나 수치, 전후 문맥을 명확히 확인하기 부족하여 청크 전문 열람이 필수적이지만, 개별 청크 아이템에는 모달로 연결되는 직통 액션 버튼이 전혀 없다.
셋째, `public-front/src/components/AiService.tsx:816-831` (`handleOpenChunks`) 및 `public-front/src/components/AiService.tsx:2419-2428` (`AiService` 내 `chunks-modal` 바디)에서 [청크 목록] 버튼(`btn-view-doc-chunks`)을 누르면 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:35-39` (`KnowledgeProxyController.listChunks`)와 `public-python-server/src/ai_service/knowledge/router.py:30-34` (`list_chunks`) 엔드포인트를 호출하여 대상 문서의 전체 청크를 조회한 뒤 1번 청크부터 단순 나열할 뿐, 특정 청크로의 스크롤 이동이나 시각적 강조 기능이 전무하다. 문서에 수십~수백 개의 청크가 존재하는 경우, 사용자는 모달을 열고 자신이 답변 버블에서 보았던 청크(예: `청크 #8`)를 찾기 위해 스크롤을 한참 내려가며 일일이 눈으로 찾아 헤매야 한다.
넷째, `public-front/src/api/ai.ts:439-442` (`getDocumentChunks`) 및 `public-front/src/api/ai.ts:263-270` (`SourceRef`)에서 개별 청크 번호(`chunkIndex`)와 전문 데이터를 다루는 타입 계약이 이미 완비되어 있으므로, 클라이언트 측에서 특정 청크를 타깃으로 스크롤 및 강조를 수행할 기술적 기반이 충분히 마련되어 있다.
따라서 참고 문서 그룹 내 청크 정렬을 청크 번호 오름차순으로 바로잡고, 개별 청크 아이템에 [전문 보기] 버튼을 제공하며, 청크 열람 모달 오픈 시 대상 청크 위치로 부드럽게 자동 스크롤(`scrollIntoView`) 및 2초간 일시적인 시각적 강조(`highlighted-chunk`)를 적용하여 사용자가 답변의 실제 근거를 즉시 확인하고 대조할 수 있도록 개선해야 한다.
## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `groupSourcesByDocument` 함수에서 각 문서 그룹 내 `chunks` 배열을 청크 인덱스(`chunkIndex`) 오름차순으로 정렬하여 반환한다. (문서 그룹 목록 자체의 `maxScore` 내림차순 정렬은 유지한다.)
- [ ] 동일 문서 내 동일 `chunkIndex`를 가진 청크가 중복 검색된 경우, `score`가 더 높은 청크 하나만 남기고 중복을 제거한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleOpenChunks` 핸들러에 선택적 매개변수 `targetChunkIndex?: number | null`을 추가하여 특정 청크 번호를 타깃으로 모달을 열 수 있도록 지원한다.
- [ ] 청크 모달 상태에 타깃 청크 인덱스 ref(`targetScrollChunkIndexRef`) 및 강조 상태(`highlightedChunkIndex`)를 관리하는 로직을 추가한다.
- [ ] 청크 열람 모달(`chunks-modal`)의 청크 목록 렌더링 영역(`chunk-item`)에 `data-chunk-index={chunk.chunkIndex}` 속성을 부여한다.
- [ ] 청크 모달의 청크 목록 비동기 조회가 완료(`data` 수신)되고 `targetChunkIndex`가 유효한 숫자로 전달된 경우, 해당 청크 엘리먼트(`[data-chunk-index="${targetChunkIndex}"]`) 위치로 부드럽게 자동 스크롤(`scrollIntoView({ behavior: 'smooth', block: 'center' })`)한다.
- [ ] 자동 스크롤된 대상 청크 엘리먼트에 일시적인 시각적 강조 테두리 및 배경 스타일(`highlighted-chunk`)을 적용하고, 2초 후 자동으로 강조를 해제한다. 모달 닫기나 재호출 시 타이머를 안전하게 정리(`clearTimeout`)한다.
- [ ] 완료된 AI 메시지 버블 내 참고 문서 목록의 각 청크 아이템(`source-chunk-item`) 메타 영역에 [전문 보기] 버튼(`btn-view-chunk-detail`)을 추가한다.
- [ ] [전문 보기] 버튼 클릭 시, 해당 문서의 `handleOpenChunks`를 호출하면서 문서 정보와 함께 해당 청크 인덱스(`chunk.chunkIndex`)를 전달하여 청크 모달 오픈 및 대상 청크 자동 스크롤·강조를 트리거한다.
- [ ] 스트리밍 진행 중인 최신 AI 메시지의 참고 문서 목록(`currentSources`)에서도 청크 번호 오름차순 정렬 및 각 청크의 [전문 보기] 버튼과 모달 스크롤 연동이 동일하게 동작한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 이전 대화 세션의 AI 메시지 버블에서도 출처 청크 번호 오름차순 정렬 및 [전문 보기] 모달 연동이 일관되게 동작한다.
- [ ] `public-front/src/components/AiService.css`에 개별 청크 [전문 보기] 버튼(`.btn-view-chunk-detail`) 스타일 및 청크 모달 내 강조 스타일(`.chunk-item.highlighted-chunk`)을 추가한다.
- [ ] 문서 그룹 내 청크 인덱스 오름차순 정렬, 개별 청크 [전문 보기] 버튼 클릭 시 모달 열기 및 타깃 청크 스크롤/강조 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 백엔드 벡터 검색 및 하이브리드 리랭킹(`HybridSearchUseCase`)의 유사도 점수 계산 로직이나 검색 파이프라인을 변경하지 않는다.
- 지식베이스 관리 테이블(`doc-table`)의 문서 단위 [청크 보기] 버튼 동작을 변경하지 않는다. (타깃 청크 없이 1번부터 열람하는 기존 동작 유지)
- 청크 열람 모달 내에서 개별 청크 내용을 수정하거나 삭제하는 기능을 추가하지 않는다.
- 원본 파일 다운로드(`handleViewSourceFile`)의 다운로드 트리거 동작을 변경하지 않는다.
## 프론트엔드
`public-front/src/components/AiService.tsx`
- `groupSourcesByDocument`:
  - `groups.values()` 순회 시 각 `group.chunks`를 `(a.chunkIndex - b.chunkIndex)` 오름차순으로 정렬한다.
  - 동일 `chunkIndex` 중복 제거: Map 키를 `chunkIndex`로 두고 `score`가 더 높은 것을 보존한다.
  - 문서 그룹 자체는 기존대로 `(b.maxScore ?? -Infinity) - (a.maxScore ?? -Infinity)` 내림차순을 유지한다.
- 상태 및 타이머 추가:
  - `targetScrollChunkIndexRef`: 모달 오픈 시 이동할 대상 청크 인덱스 ref (`useRef<number | null>(null)`).
  - `highlightedChunkIndex`: 현재 강조 표시 중인 청크 인덱스 상태 (`useState<number | null>(null)`).
  - `chunkHighlightTimerRef`: 2초 후 강조 해제용 타이머 ref (`useRef<NodeJS.Timeout | null>(null)`).
- `handleOpenChunks` 시그니처 및 로직 확장:
  - `handleOpenChunks(doc: DocumentInfo | { id: string; fileName: string; chunkCount?: number }, targetChunkIndex?: number | null)`로 시그니처를 확장한다.
  - `targetScrollChunkIndexRef.current = targetChunkIndex ?? null`로 타깃 인덱스를 저장한다.
  - 청크 데이터 로드 완료 후 `setTimeout` 또는 DOM 렌더링 타이밍에 `document.querySelector(`[data-chunk-index="${targetChunkIndex}"]`)?.scrollIntoView({ behavior: 'smooth', block: 'center' })`를 호출하고, `setHighlightedChunkIndex(targetChunkIndex)` 설정 후 2초 타이머를 등록한다.
- 모달 닫기 핸들러(`handleCloseChunksModal`):
  - `setHighlightedChunkIndex(null)`, `targetScrollChunkIndexRef.current = null` 초기화 및 활성 타이머를 정리한다.
- 참고 문서 청크 렌더링 (`source-chunk-meta`):
  - `청크 #{chunk.chunkIndex + 1}` 옆에 `<button type="button" className="btn-view-chunk-detail" onClick={() => handleOpenChunks({ id: group.documentId, fileName: group.fileName, chunkCount: group.chunks.length }, chunk.chunkIndex)}>전문 보기</button>`를 추가한다.
  - 완료된 메시지 버블 및 스트리밍 메시지 버블 모두에 동일하게 적용한다.
- 청크 모달 리스트 (`chunk-item`):
  - `<div key={chunk.chunkIndex} className={`chunk-item${chunk.chunkIndex === highlightedChunkIndex ? ' highlighted-chunk' : ''}`} data-chunk-index={chunk.chunkIndex}>` 형태로 렌더링한다.

`public-front/src/components/AiService.css`
- `.btn-view-chunk-detail`: 청크 메타 영역 내 전문 보기 버튼 스타일 (간결한 인라인 텍스트 버튼 형태, hover 시 밑줄 또는 배경 전환).
- `.chunk-item.highlighted-chunk`: 대상 청크 강조 스타일 (시각적으로 눈에 띄는 파란색/청록색 테두리 및 옅은 하이라이트 배경색, 부드러운 전환 효과).

`public-front/src/components/AiService.test.tsx`
- 출처 청크 목록이 `chunkIndex` 오름차순으로 정렬되어 렌더링되는지 검증하는 단위 테스트.
- 개별 청크의 [전문 보기] 버튼 클릭 시 `handleOpenChunks`가 해당 청크 인덱스와 함께 호출되고 모달이 열리는지 검증하는 단위 테스트.
- 청크 모달 로드 후 타깃 청크 엘리먼트에 `highlighted-chunk` 클래스가 적용되고 `scrollIntoView`가 호출되는지 검증하는 단위 테스트.
- 타깃 청크 인덱스 없이 문서 헤더의 [청크 목록] 버튼으로 모달을 열었을 때(경계 케이스) 강조 효과나 스크롤 이동 없이 전체 목록이 정상 렌더링되는지 검증하는 단위 테스트.
## 수용 기준 (Acceptance Criteria)
- Given 동일 문서에서 여러 청크(예: 청크 인덱스 4, 1, 3)가 검색되어 AI 답변 참고 문서로 전달되었을 때, When 사용자가 해당 문서 출처를 펼쳐 청크 목록을 확인하면, Then 청크들이 `청크 #2`, `청크 #4`, `청크 #5`와 같이 청크 번호 오름차순으로 정렬되어 화면에 표시된다.
- Given AI 답변 버블의 참고 문서 내 특정 청크(`청크 #3`) 아이템에 [전문 보기] 버튼이 표시되어 있을 때, When 사용자가 해당 [전문 보기] 버튼(`btn-view-chunk-detail`)을 클릭하면, Then 대상 문서의 청크 열람 모달(`chunks-modal`)이 열리고 청크 목록 조회가 완료된 후 `청크 #3` 엘리먼트(`[data-chunk-index="2"]`) 위치로 자동 스크롤되며 2초간 `highlighted-chunk` 강조 스타일이 적용된다.
- Given 스트리밍 진행 중 실시간 출처(`currentSources`)가 수신되었을 때, When 실시간 참고 문서 영역이 펼쳐지면, Then 완료된 메시지와 동일하게 각 청크에 [전문 보기] 버튼이 제공되고 클릭 시 모달 열기 및 해당 청크 스크롤 연동이 정상 동작한다.
- Given 타깃 청크 지정 없이 문서 행의 [청크 목록] 버튼(`btn-view-doc-chunks`)을 통해 모달을 연 경우(경계 케이스), When 청크 목록이 로드되면, Then 스크롤 강제 이동이나 특정 청크 강조 클래스(`highlighted-chunk`) 부여 없이 전체 청크 목록이 기본 상태로 안정적으로 렌더링된다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:382-409` (`groupSourcesByDocument`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:816-831` (`handleOpenChunks`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1960-1983` (`AiService`의 `source-group-chunks`)
- 관련 정의 및 모달 렌더링: `public-front/src/components/AiService.tsx:2419-2428` (`AiService`의 `chunks-modal` 내부 청크 렌더링)
- 클라이언트 API 입구: `public-front/src/api/ai.ts:439-442` (`getDocumentChunks`)
- 백엔드 게이트웨이 입구: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:35-39` (`KnowledgeProxyController.listChunks`)
- 백엔드 서비스 입구: `public-python-server/src/ai_service/knowledge/router.py:30-34` (`list_chunks`)
- 참고 패턴(턴 스크롤 및 하이라이트): `public-front/src/components/AiService.tsx:533-547` (`handleLoadSession` 내 `targetTurnIndex` 스크롤 및 2초 하이라이트 타이머)
