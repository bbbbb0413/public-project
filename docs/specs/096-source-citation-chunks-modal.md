---
id: SPEC-096
title: AI 답변 참고 문서(출처)의 청크 번호 표기(1-based) 통일 및 문서 색인 청크 모달 연동
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
이 제품의 핵심 목표는 올려 둔 문서를 근거로 질문에 답하며, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 것이다. 현재 AI 답변 메시지 버블 하단에는 답변 생성에 활용된 지식 문서와 청크 목록을 확인할 수 있는 참고 문서(출처, `source-refs`) 영역(SPEC-006, SPEC-042)이 제공된다. 그러나 사용자가 답변의 실제 근거를 대조하고 검증하는 흐름에서 심각한 표기 불일치와 기능 단절이 존재한다.
첫째, `public-front/src/components/AiService.tsx:1620-1622` (`AiService`) 및 `public-front/src/components/AiService.tsx:1764-1766` (`AiService`)에서 참고 문서 각 청크 번호가 0부터 시작하는 단순 배열 인덱스(`청크 {chunk.chunkIndex}`)로 렌더링되고 있다. 반면 백엔드 RAG 프롬프트 조립부인 `public-python-server/src/ai_service/rag/application/ask_use_case.py:176-180` (`AskUseCase._build_rag_messages`) 및 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:284-288` (`AgenticAskUseCase._build_messages`)에서는 LLM에 주입하는 컨텍스트 레이블로 `[출처 N: 문서명 (섹션 M)]`과 같이 1-based 번호(`chunk_index + 1`)를 사용하고 있다. 또한 지식베이스 색인 청크 열람 모달(`public-front/src/components/AiService.tsx:1995`)에서도 `청크 #{chunk.chunkIndex + 1}`로 1-based 표기를 사용한다. 이로 인해 답변 버블에서는 '청크 0'으로 표시되지만 실제 섹션 번호나 문서 모달에서는 '청크 #1'로 나타나 사용자가 근거 청크를 대조할 때 큰 혼란을 겪는다.
둘째, `public-front/src/components/AiService.tsx:1629-1636` (`AiService`)에서 참고 문서 청크는 최대 300자로 축약된 `snippet`만 노출된다. 답변의 사실관계를 정확히 검증하려면 해당 청크의 전체 텍스트와 전후 문맥을 확인할 수 있어야 하지만, 현재 답변 버블에서 제공하는 유일한 열람 수단은 `public-front/src/components/AiService.tsx:1607-1614` (`AiService`)의 [원문 보기] 버튼(`handleViewSourceFile`)뿐이다. 이 버튼은 수십~수백 페이지에 달하는 전체 원본 파일(PDF/TXT/MD)을 새 탭에서 통째로 다운로드하거나 열기 때문에, 사용자는 방대한 원본 문서 속에서 해당 청크가 어디에 위치하는지 다시 일일이 찾아 헤매야 한다.
셋째, 이미 SPEC-083을 통해 `public-front/src/api/ai.ts:430-445` (`getDocumentChunks`) 및 `public-front/src/components/AiService.tsx:665-685` (`handleOpenChunks`, `handleCloseChunksModal`)를 활용한 청크 열람 모달(`chunks-modal`, `public-front/src/components/AiService.tsx:1963-2002`)이 완성되어 있다. 그러나 이 기능은 지식베이스 관리 테이블(`doc-table`)에서만 호출할 수 있도록 닫혀 있으며, 정작 사용자가 답변을 신뢰하기 위해 근거를 확인해야 하는 AI 대화 화면의 참고 문서 영역에서는 이 청크 모달을 열 수 있는 진입점이 전혀 없다.
따라서 AI 답변 참고 문서 영역의 청크 번호 표기를 1-based(`청크 #{chunk.chunkIndex + 1}`)로 통일하고, 출처 그룹별로 [청크 목록] 버튼을 제공하여 기존 청크 열람 모달과 직접 연계함으로써 사용자가 답변 버블을 벗어나지 않고도 근거 청크 전문과 전체 문서 문맥을 즉시 확인할 수 있도록 개선해야 한다.

## 요구사항
- [ ] AI 메시지 버블 내 참고 문서 목록(`source-refs`)의 개별 청크 번호 표기를 0-based(`청크 {chunk.chunkIndex}`)에서 1-based(`청크 #{chunk.chunkIndex + 1}`)로 수정한다.
- [ ] 스트리밍 중인 최신 답변 버블의 실시간 참고 문서 목록에서도 개별 청크 번호를 1-based(`청크 #{chunk.chunkIndex + 1}`)로 일관되게 표시한다.
- [ ] 참고 문서 각 문서 그룹 행(`source-ref-item-container`) 작업 영역에 [청크 목록] 버튼(`btn-view-doc-chunks`)을 추가한다.
- [ ] 사용자가 [청크 목록] 버튼을 클릭하면, 해당 문서의 식별자(`group.documentId`)와 파일명(`group.fileName`)을 바탕으로 청크 열람 모달(`chunks-modal`)을 열고 색인 청크 목록을 비동기 조회하여 표시한다.
- [ ] `handleOpenChunks` 핸들러가 지식베이스 `DocumentInfo` 객체뿐만 아니라 출처 그룹 객체(`{ id: string; fileName: string; chunkCount?: number }`)로도 호출될 수 있도록 파라미터 타입을 확장하거나 안전한 호출 헬퍼를 제공한다.
- [ ] 출처 정보에 전체 `chunkCount`가 포함되어 있지 않은 경우, 청크 목록 비동기 조회 완료 후 반환된 청크 배열의 길이(`chunks.length`)를 모달 헤더의 전체 청크 수(예: `총 {count}개`)에 정확히 반영한다.
- [ ] 청크 모달이 열린 상태에서 상단 닫기 버튼(`✕`), 하단 [닫기] 버튼, 모달 바깥 배경 오버레이 클릭, 키보드 `Escape` 키 입력 시 모달을 정상적으로 닫고 상태를 초기화한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 이전 대화의 AI 메시지 버블에서도 [청크 목록] 버튼 및 1-based 청크 번호 표기가 동일하게 정상 동작한다.
- [ ] 변경된 출처 청크 번호 표기(1-based), [청크 목록] 버튼 클릭 시 모달 열기 및 데이터 조회, 모달 닫기 인터랙션을 검증하는 프론트엔드 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 백엔드 벡터 스토어의 청킹 알고리즘(청크 크기, 오버랩 비율) 변경이나 재인제스트 파이프라인 수정은 포함하지 않는다.
- 기존 [원문 보기] 버튼(`handleViewSourceFile`)의 파일 다운로드/새 탭 열기 동작은 유지하며 변경하지 않는다.
- 청크 열람 모달 내에서 개별 청크를 수정하거나 삭제하는 기능은 제공하지 않는다.
- 지식베이스 문서 목록 테이블(`doc-table`)의 기존 [청크 보기] 버튼 동작을 변경하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 청크 모달 대상 타입 확장: `viewingChunksDoc`의 타입을 `DocumentInfo | { id: string; fileName: string; chunkCount?: number } | null`로 조정하거나 공통 인터페이스 적용
- `handleOpenChunks`: 파라미터가 `DocumentInfo` 또는 출처 그룹 규격(`{ id: string; fileName: string; chunkCount?: number }`)을 수신할 수 있도록 시그니처 확장
- 출처 청크 번호 렌더링 수정:
  - 완료된 AI 메시지 버블: `청크 {chunk.chunkIndex}` -> `청크 #{chunk.chunkIndex + 1}`로 변경
  - 스트리밍 중인 AI 메시지 버블: `청크 {chunk.chunkIndex}` -> `청크 #{chunk.chunkIndex + 1}`로 변경
- 출처 문서 그룹 액션 추가:
  - `source-ref-view-btn`("원문 보기") 옆에 `btn-view-doc-chunks`("[청크 목록]") 버튼 렌더링
  - 클릭 시 `handleOpenChunks({ id: group.documentId, fileName: group.fileName, chunkCount: group.chunks.length })` 호출
- 청크 모달 헤더(`chunks-modal-count`) 보완:
  - `viewingChunksDoc.chunkCount !== undefined ? viewingChunksDoc.chunkCount : chunks.length`로 계산하여 청크 수 표시

`public-front/src/components/AiService.css`
- `.btn-view-doc-chunks`: 참고 문서 출처 그룹 내 청크 목록 버튼 스타일 (기존 `.source-ref-view-btn`과 조화를 이루는 패딩, 테두리, 폰트 크기 및 hover 스타일)

`public-front/src/components/AiService.test.tsx`
- AI 답변 버블의 참고 문서 내 청크 번호가 1-based(`청크 #1`)로 렌더링되는지 검증하는 단위 테스트
- 참고 문서 그룹의 [청크 목록] 버튼 클릭 시 `getDocumentChunks`가 올바른 `documentId`로 호출되고 청크 모달이 열리는지 검증하는 단위 테스트
- 청크 모달이 열린 상태에서 닫기 버튼 및 `Escape` 키 입력 시 닫히는지 검증하는 단위 테스트
- `chunkCount`가 없는 출처 객체로 모달이 열렸을 때 로드된 청크 수로 모달 헤더 카운트가 정상 표시되는지 검증하는 단위 테스트 (경계 케이스)

## 수용 기준 (Acceptance Criteria)
- Given AI 답변 버블에 참고 문서(출처) 목록이 렌더링되어 있을 때, When 사용자가 해당 출처를 펼쳐 청크 메타 영역을 확인하면, Then 각 청크 번호가 0-based(`청크 0`)가 아닌 1-based(`청크 #1`, `청크 #2`) 형식으로 표시된다.
- Given AI 답변 버블의 참고 문서 목록에 특정 문서 출처가 표시되어 있을 때, When 사용자가 해당 문서 행의 [청크 목록] 버튼(`btn-view-doc-chunks`)을 클릭하면, Then `getDocumentChunks` API가 해당 `documentId`로 호출되고 대상 문서 파일명이 표시된 청크 열람 모달(`chunks-modal`)이 화면에 렌더링된다.
- Given 답변 참고 문서를 통해 청크 열람 모달이 열린 상태에서 When 사용자가 모달 닫기 버튼(`✕`)을 클릭하거나 배경 오버레이를 클릭하거나 `Escape` 키를 누르면 Then 청크 모달이 닫히고 대화창 화면으로 정상 복귀한다.
- Given 출처 정보에 전체 `chunkCount` 메타데이터가 없는 상태(경계 케이스)에서 When 사용자가 [청크 목록] 버튼을 클릭해 2건의 청크가 로드되면 Then 모달 헤더에 `총 2개`로 조회된 청크 개수가 정상 표시된다.
- Given 세션 복원(`handleLoadSession`)으로 과거 대화 내역을 불러온 상황에서 When AI 메시지의 출처 목록을 확인하면 Then 동일하게 1-based 청크 번호(`청크 #1`)와 [청크 목록] 버튼이 정상적으로 노출되고 클릭 시 청크 모달이 정상 동작한다.
- Given 답변 스트리밍이 진행 중인 상황에서 When 실시간 수신된 출처 목록(`currentSources`)을 확인하면 Then 스트리밍 버블 하단에서도 1-based 청크 번호(`청크 #1`)와 [청크 목록] 버튼이 일관되게 렌더링된다.

## 참고
- 고쳐야 할 자리 (완료 답변 출처 렌더링부): `public-front/src/components/AiService.tsx:1607-1640` (`AiService`)
- 고쳐야 할 자리 (스트리밍 답변 출처 렌더링부): `public-front/src/components/AiService.tsx:1751-1780` (`AiService`)
- 고쳐야 할 자리 (청크 모달 핸들러): `public-front/src/components/AiService.tsx:665-685` (`handleOpenChunks`, `handleCloseChunksModal`)
- 기존 청크 모달 렌더링부: `public-front/src/components/AiService.tsx:1963-2002` (`AiService` 내 `chunks-modal` 렌더링부)
- 백엔드 1-based 컨텍스트 포맷: `public-python-server/src/ai_service/rag/application/ask_use_case.py:176-180` (`AskUseCase._build_rag_messages`)
- 백엔드 1-based 컨텍스트 포맷: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:284-288` (`AgenticAskUseCase._build_messages`)
- 청크 목록 API 클라이언트: `public-front/src/api/ai.ts:430-445` (`getDocumentChunks`)
