---
id: SPEC-111
title: AI 답변 출처 청크 열람 모달의 전체 청크 수 오표시 결함 수정 및 인용 청크 관련도 점수 시각화 연동
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 이를 위해 답변 근거 미리보기(SPEC-006), 출처 청크 번호 1-based 표기 및 문서 색인 청크 모달 연동(SPEC-096), 개별 청크 모달 위치 이동·강조 연동(SPEC-106), 출처 청크 관련도 수준별 시각화(SPEC-108) 등이 단계적으로 배송되었다.
그러나 "사용자가 답변의 근거를 스스로 확인할 수 있는가"라는 제품 핵심 가치와 관련하여, 청크 전문을 확인하는 청크 열람 모달(`chunks-modal`)에 다음과 같은 심각한 수치 왜곡 결함과 근거 정보 단절이 존재한다:
첫째, `public-front/src/components/AiService.tsx:2120-2134` (`AiService` 내 완료 메시지 참고 문서 [전문 보기] 버튼 호출부) 및 `public-front/src/components/AiService.tsx:2307-2321` (`AiService` 내 스트리밍 참고 문서 [전문 보기] 버튼 호출부)를 보면, 개별 청크의 [전문 보기] 버튼 클릭 시 `handleOpenChunks`로 `chunkCount: group.chunks.length`를 전달하고 있다. 이는 대상 문서의 전체 청크 수가 아니라 이번 질의응답에서 검색된 청크 수(예: 1~2개)에 불과하다. 이로 인해 `public-front/src/components/AiService.tsx:2574-2579` (`AiService` 내 청크 모달 헤더 카운트 렌더링부)에서 `총 {viewingChunksDoc.chunkCount !== undefined ? viewingChunksDoc.chunkCount : chunks.length}개`로 평가될 때, 백엔드로부터 문서의 전체 청크 50개를 정상 수신했음에도 전달받은 `chunkCount`(1 또는 2)가 우선되어 모달 헤더에 "총 1개" 또는 "총 2개"로 심각하게 왜곡된 전체 청크 수가 표기된다. 사용자는 문서 전체가 고작 1개 청크로만 구성된 불완전한 문서로 오인하게 된다.
둘째, 반대로 `public-front/src/components/AiService.tsx:2101-2107` (`AiService` 내 완료 메시지 참고 문서 [청크 목록] 버튼 호출부)에서는 `chunkCount: (group as { chunkCount?: number }).chunkCount`를 넘기는데, `groupSourcesByDocument`가 반환하는 출처 그룹 객체에는 `chunkCount` 필드가 없어 항상 `undefined`가 전달된다. 이 경우 모달 헤더는 `chunks.length`로 평가되어 "총 50개"가 정상 표시된다. 즉, 동일한 문서의 동일한 모달을 열었음에도 [청크 목록] 버튼을 눌렀을 때는 "총 50개", 바로 옆의 [전문 보기] 버튼을 눌렀을 때는 "총 1개"가 표시되는 황당한 계약 불일치가 발생하고 있다.
셋째, `public-front/src/components/AiService.tsx:2603-2617` (`AiService` 내 청크 모달 청크 목록 렌더링부)를 보면, 모달 내부에서는 단지 클릭된 단일 타깃 청크(`targetChunkIndex`)만 2초간 일시적으로 반짝인 뒤 사라질 뿐, 해당 문서에서 답변의 근거로 인용된 청크들(`citedChunks`)이 무엇인지 모달 안에서는 일체 구별되지 않는다. 또한 SPEC-108을 통해 답변 버블에는 관련도 점수(`source-ref-score`, 예: 85%)와 수준별 색상(`score-high`, `score-medium`)이 도입되었으나, 청크 열람 모달 내부의 청크 헤더(`chunk-item-header`)에는 관련도 점수나 인용 배지가 전무하여, 사용자가 전후 문맥을 확인하러 모달에 들어왔을 때 어떤 청크가 답변을 뒷받침하는 핵심 근거인지 직관적으로 확인할 수 없다.
따라서 청크 열람 모달의 전체 청크 수 평가 로직을 실제 비동기 로드된 전체 청크 수(`chunks.length`) 및 지식베이스 원본 문서 청크 수 기준으로 바로잡고, 답변 버블에서 모달로 진입할 때 인용 청크 정보와 관련도 점수 맵을 전달하여 모달 내에서도 인용 청크 배지(`cited-chunk-badge`)와 관련도 수준별 배지를 명확히 렌더링함으로써 사용자의 근거 검증 경험을 완성해야 한다.
이 개선은 "답변을 믿을 수 있게 만든다"는 이번 분기 집중 방향 아래, 사용자가 올려 둔 문서를 근거로 답변을 스스로 대조·확인하는 핵심 흐름의 결함을 해소하는 필수 작업이다.
## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `handleOpenChunks` 시그니처를 확장하여, 인용 청크 목록 또는 점수 맵(예: `citedChunks?: SourceRef[]` 또는 `citedScores?: Record<number, number>`)을 선택적 매개변수로 수신할 수 있도록 한다.
- [ ] `public-front/src/components/AiService.tsx`의 완료 메시지 및 스트리밍 메시지 참고 문서 목록에서 [전문 보기] 및 [청크 목록] 클릭 시, 해당 문서 그룹의 청크 목록(`group.chunks`)을 `handleOpenChunks`에 전달하여 모달에 인용 청크 상태를 바인딩한다.
- [ ] `public-front/src/components/AiService.tsx`의 [전문 보기] 버튼 클릭 시 왜곡된 `chunkCount: group.chunks.length` 전달을 제거하고, 지식베이스 원본 문서의 `chunkCount`가 존재하는 경우에만 전달하거나 `undefined`로 안전하게 전달한다.
- [ ] `public-front/src/components/AiService.tsx`의 청크 모달 헤더(`chunks-modal-header`)에서 청크 목록 조회가 완료되어 `chunks.length > 0`인 경우, 비동기 응답으로 확인된 실제 전체 청크 수(`chunks.length`)를 모달 헤더 카운트(예: `총 {chunks.length}개`)에 우선 반영하여 진입 경로와 무관하게 정확한 전체 청크 수가 노출되도록 수정한다.
- [ ] 청크 모달 내부(`chunk-item`)에서 해당 청크가 답변에 인용된 청크(`citedChunks`에 포함된 청크)인 경우:
  - 청크 컨테이너 엘리먼트에 `cited-chunk` CSS 클래스를 부여하여 시각적 구분을 제공한다.
  - 청크 헤더 영역에 `[답변 근거 인용]` 상태 배지(`cited-chunk-badge`, `data-testid="cited-chunk-badge"`)를 렌더링한다.
  - 해당 청크의 관련도 점수가 존재하는 경우, 관련도 점수 배지(`<span className={`source-ref-score ${getScoreLevelClass(score)}`}>관련도 {Math.round(score * 100)}%</span>`)를 함께 렌더링한다.
- [ ] 지식베이스 문서 관리 테이블의 [청크 목록] 및 청크 수 버튼을 통해 모달을 연 경우(답변 인용 맥락이 아닌 일반 열람 컨텍스트), 기존과 동일하게 전체 청크 수가 정상 노출되고 `cited-chunk` 배지는 렌더링되지 않는다.
- [ ] `public-front/src/components/AiService.css`에 `cited-chunk` 및 `cited-chunk-badge` 스타일을 추가한다.
- [ ] 청크 모달 헤더의 전체 청크 수 정상 표시, 답변 인용 청크 배지 렌더링, 관련도 수준별 배지 적용을 검증하는 프론트엔드 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 백엔드 청크 조회 API(`GET /ai/knowledge/documents/:documentId/chunks`)의 응답 규격이나 DB 스키마를 수정하지 않는다.
- 청크 모달 내 검색(필터링)이나 페이지네이션 기능을 추가하지 않는다.
- 답변 버블의 아코디언 접힘/펼침 동작이나 원본 파일 다운로드(`handleViewSourceFile`) 로직을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 화면(`Payment.tsx`) 컴포넌트를 수정하지 않는다.
## 프론트엔드
`public-front/src/components/AiService.tsx`
- 청크 모달 상태 관리 보강:
  - `const [modalCitedChunks, setModalCitedChunks] = useState<Record<number, number>>({});` (청크 인덱스별 관련도 점수 매핑)
- `handleOpenChunks` 매개변수 확장:
  - `const handleOpenChunks = useCallback(async (doc: DocumentInfo | { id: string; fileName: string; chunkCount?: number }, targetChunkIndex?: number | null, citedChunks?: SourceRef[]) => { ... });`
  - `citedChunks`가 전달되면 `const map: Record<number, number> = {}; citedChunks.forEach(c => { if (typeof c.score === 'number') map[c.chunkIndex] = c.score; else map[c.chunkIndex] = -1; }); setModalCitedChunks(map);` 처리.
  - 전달되지 않으면 `setModalCitedChunks({});` 처리.
- `handleCloseChunksModal` 시 `setModalCitedChunks({});` 초기화.
- [전문 보기] 및 [청크 목록] 호출부 정정:
  - 완료 메시지 [전문 보기]: `handleOpenChunks({ id: group.documentId, fileName: group.fileName }, chunk.chunkIndex, group.chunks)`
  - 완료 메시지 [청크 목록]: `handleOpenChunks({ id: group.documentId, fileName: group.fileName }, null, group.chunks)`
  - 스트리밍 메시지 [전문 보기] 및 [청크 목록]에서도 동일하게 `group.chunks` 전달.
- 청크 모달 헤더 카운트 수정:
  - `const totalChunkCount = chunks.length > 0 ? chunks.length : (viewingChunksDoc.chunkCount ?? 0);`
  - `<span className="chunks-modal-count">총 {totalChunkCount}개</span>`
- 청크 모달 본문 청크 헤더 렌더링 확장:
  - `const isCited = modalCitedChunks[chunk.chunkIndex] !== undefined;`
  - `const citedScore = isCited && modalCitedChunks[chunk.chunkIndex] >= 0 ? modalCitedChunks[chunk.chunkIndex] : undefined;`
  - `className={`chunk-item${chunk.chunkIndex === highlightedChunkIndex ? ' highlighted-chunk' : ''}${isCited ? ' cited-chunk' : ''}`}`
  - 헤더 영역:
    ```tsx
    <div className="chunk-item-header">
      <span className="chunk-badge">청크 #{chunk.chunkIndex + 1}</span>
      {isCited && (
        <span className="cited-chunk-badge" data-testid="cited-chunk-badge">
          답변 근거 인용
        </span>
      )}
      {typeof citedScore === 'number' && (
        <span className={`source-ref-score ${getScoreLevelClass(citedScore)}`.trim()}>
          관련도 {Math.round(citedScore * 100)}%
        </span>
      )}
    </div>
    ```
`public-front/src/components/AiService.css`
- `.chunk-item.cited-chunk`: 답변 인용 청크에 대한 강조 테두리 및 은은한 배경 스타일 (`border: 1px solid rgba(165, 180, 252, 0.4); background: rgba(99, 102, 241, 0.05);`)
- `.cited-chunk-badge`: 인용 청크 배지 스타일 (연보라 계열 배경 `rgba(165, 180, 252, 0.18)`, 텍스트 `#c7d2fe`, 11px, 둥근 모서리)
## 수용 기준 (Acceptance Criteria)
- Given 전체 청크가 20개인 문서에서 2개 청크(청크 #1, 청크 #5)가 답변 출처로 인용되었을 때 When 사용자가 참고 문서 목록의 청크 #1 [전문 보기] 버튼을 클릭하면 Then 청크 모달 헤더에 왜곡된 "총 2개"가 아닌 "총 20개"가 정확히 표시되고, 청크 #1 위치로 자동 스크롤된다.
- Given 답변 참고 문서 그룹에서 [전문 보기] 또는 [청크 목록]을 클릭하여 청크 모달이 열렸을 때 When 인용된 청크(청크 #1 관련도 88%, 청크 #5 관련도 65%) 항목을 확인하면 Then 해당 청크 아이템에 `cited-chunk` 클래스가 적용되고, 헤더에 `[답변 근거 인용]` 배지와 함께 각각 `score-high`(관련도 88%), `score-medium`(관련도 65%) 점수 배지가 표시된다.
- Given 동일 문서 내에서 답변에 인용되지 않은 다른 청크(예: 청크 #2, #3)를 모달에서 확인했을 때 When 렌더링 결과를 확인하면 Then `cited-chunk` 클래스와 `[답변 근거 인용]` 배지가 표시되지 않는다.
- Given (경계 케이스) 지식베이스 문서 관리 테이블에서 문서 행의 청크 수 버튼이나 [청크 보기] 버튼을 클릭하여 모달을 열었을 때 When 청크 목록을 확인하면 Then "총 {N}개"가 정상 표시되고 모든 청크에 `cited-chunk` 배지가 표시되지 않는다.
- Given (경계 케이스) 인용 청크 데이터에 `score` 정보가 누락되어 있거나 유효하지 않은 경우 When 인용 청크 모달을 열면 Then `[답변 근거 인용]` 배지는 정상 표시되되 관련도 점수 배지는 오류 없이 생략된다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2101-2134` (`AiService` 내 완료 메시지 참고 문서 버튼부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2287-2321` (`AiService` 내 스트리밍 참고 문서 버튼부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2570-2580` (`AiService` 내 청크 모달 헤더 카운트 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2603-2617` (`AiService` 내 청크 모달 청크 목록 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:878-916` (`handleOpenChunks` 콜백 함수 정의부)
- 관련 기존 구현: `public-front/src/components/AiService.css:1229-1250` (`.source-ref-score`, `.score-high` 등 스타일 정의부)
