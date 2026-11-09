---
id: SPEC-097
title: AI 답변 참고 문서 원문 보기 팝업 차단 및 빈 탭 잔류 결함 수정과 에러 상태 격리
status: done
targets: [front, server]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스에 등록된 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 것이다. 현재 AI 답변 메시지 버블 하단에는 답변 생성에 활용된 문서 목록과 청크 요약을 확인할 수 있는 참고 문서(출처, `source-refs`) 영역(SPEC-006, SPEC-042)이 제공되며, 각 문서 그룹마다 원문 전체를 확인할 수 있는 [원문 보기] 버튼(SPEC-067)이 배치되어 있다. 그러나 사용자가 답변의 실제 근거 원문을 확인하려는 핵심 검증 흐름에서 심각한 결함과 상태 오염이 발생하고 있다.
첫째, `public-front/src/components/AiService.tsx:361-381` (`handleViewSourceFile`)을 보면 사용자가 [원문 보기] 버튼을 클릭했을 때 비동기 파일 다운로드 API(`getDocumentFile`) 호출에 앞서 `window.open('', '_blank')`로 빈 브라우저 탭을 먼저 생성한다. 최신 웹 브라우저(Chrome, Safari 등)의 팝업 차단 정책은 사용자 클릭 이벤트와 동기적으로 발생하지 않는 비동기 지연 후의 내비게이션이나 팝업 생성을 보안 위협으로 간주하여 즉시 차단한다. 이로 인해 `newTab`이 `null`로 반환되거나 팝업 차단 경고가 뜨며 원문 열람이 원천 차단된다.
둘째, 설령 브라우저에서 팝업이 허용되더라도 게이트웨이 `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:41-51` (`KnowledgeProxyController.getFile`) 및 백엔드 `public-python-server/src/ai_service/knowledge/router.py:36-47` (`get_document_file`)가 전달하는 응답 헤더와 브라우저 Blob 처리 방식으로 인해 새 탭 내에서 파일이 직접 렌더링되지 않고 브라우저 백그라운드 다운로드가 실행된다. 이 과정에서 앞서 동기적으로 열어둔 새 탭은 `about:blank`인 채 빈 흰 화면으로 닫히지 않고 브라우저에 영구 잔류하여 사용자 경험을 심각하게 훼손한다.
셋째, `public-front/src/components/AiService.tsx:166` (`AiService`)에서 원문 조회 실패 에러 상태(`fileViewError`)가 단일 문자열(`string | null`) 전역 상태로 관리되고 있다. 이로 인해 특정 메시지 버블의 한 출처 문서에서 파일 로딩 오류가 발생하면, `public-front/src/components/AiService.tsx:1690-1694` (`AiService`) 및 `public-front/src/components/AiService.tsx:1835-1839` (`AiService`)에 의해 현재 대화창에 존재하는 **모든** AI 메시지 버블과 스트리밍 중인 최신 답변 버블의 참고 문서 영역 하단에 "원본 파일을 불러오는 데 실패했습니다."라는 빨간 에러 메시지가 일괄 노출되는 치명적인 전역 상태 오염이 발생한다.
넷째, 로딩 진행 상태를 나타내는 `viewingFileId`(`public-front/src/components/AiService.tsx:165`) 역시 전역 단일 문자열로 관리되어, 다중 턴 대화에서 서로 다른 메시지에 인용된 동일 문서나 다른 문서들의 [원문 보기] 버튼 상태가 독립적으로 제어되지 못한다.
따라서 사전 빈 탭 생성 방식을 제거하고 브라우저 팝업 차단 없이 안전하게 원문 파일을 다운로드/열람할 수 있는 표준 메커니즘을 적용하며, 출처 문서 그룹별로 로딩 및 에러 상태를 완전히 격리해야 한다.

## 요구사항
- [ ] `handleViewSourceFile` 실행 시 비동기 호출 전에 빈 탭(`window.open('', '_blank')`)을 사전에 생성하는 방식을 제거하여 브라우저 팝업 차단 및 `about:blank` 빈 탭 잔류 문제를 원천 차단한다.
- [ ] `getDocumentFile(documentId)` 비동기 호출을 통해 Blob 데이터를 정상 수신한 후, 안전한 객체 URL(`URL.createObjectURL`)과 임시 앵커 엘리먼트(`<a download>`) 트리거를 활용하여 파일명(`group.fileName`)을 유지한 채 안전하게 다운로드되도록 처리한다.
- [ ] 원문 파일 다운로드/열람 실패 시 발생하는 에러 상태(`fileViewError`)를 단일 전역 문자열 대신 문서 식별자 또는 그룹 키 단위(`Record<string, string | null>`)로 관리하여, 특정 출처의 에러가 다른 메시지 버블이나 스트리밍 영역으로 전파되지 않도록 완벽히 격리한다.
- [ ] 원문 파일 요청 진행 상태(`viewingFileId`)를 특정 출처 항목 단위(`viewingDocIds: Set<string>` 또는 `Record<string, boolean>`)로 관리하여, 여러 답변 버블의 버튼들이 서로 간섭받지 않고 독립적으로 "불러오는 중…" 상태와 비활성화(`disabled`) 처리를 유지하도록 한다.
- [ ] 게이트웨이 `KnowledgeProxyController.getFile` (`GET /ai/knowledge/documents/:id/file`)에서 원본 파일 다운로드 시 브라우저가 파일명을 온전히 인식할 수 있도록 `Content-Disposition` 헤더 및 적절한 `Content-Type` 헤더가 안정적으로 전달되는지 확인하고 검증한다.
- [ ] 스트리밍 진행 중인 최신 AI 메시지의 출처 목록(`currentSources`)에서도 동일하게 격리된 로딩 상태와 에러 표시가 적용되어야 한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 이전 대화의 AI 메시지 버블 내 출처 목록에서도 동일한 안전한 원문 다운로드 및 에러 격리가 정상 동작해야 한다.
- [ ] 변경된 원문 파일 다운로드 흐름, 팝업 차단 없는 다운로드 트리거, 문서별 에러 상태 격리 동작을 검증하는 프론트엔드 및 게이트웨이 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 지식베이스 색인 청크 열람 모달(`chunks-modal`)의 내부 동작이나 디자인을 변경하는 작업은 포함하지 않는다 (SPEC-083, SPEC-096 영역).
- 브라우저 내부에 별도의 전용 PDF 뷰어 컴포넌트나 텍스트 에디터를 신규 구현하여 임베딩하는 것은 포함하지 않는다. (브라우저 표준 파일 다운로드/열람 동작에 집중한다.)
- 지식베이스 문서 목록 테이블(`doc-table`)의 기존 파일 다운로드(`handleDownloadDocument`) 로직은 이미 `<a download>` 방식으로 안전하게 구현되어 있으므로 변경하지 않는다.
- 백엔드 MongoDB 파일 저장소의 저장 구조나 청킹 알고리즘 변경은 포함하지 않는다.

## 백엔드
`public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts`
- `getFile` 메서드(`GET /ai/knowledge/documents/:id/file`):
- `aiServicePy.getBinary`를 통해 수신된 바이너리 데이터의 `Content-Type`과 `Content-Disposition` 헤더를 응답에 정확히 매핑하여 전달한다.
- `Content-Disposition` 헤더가 누락되었거나 안전하지 않은 경우 기본 파일명(`filename="{id}"`)을 보완하여 클라이언트가 올바른 파일 형식으로 수신할 수 있도록 보장한다.

`public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.spec.ts`
- 원본 파일 다운로드 요청 시 올바른 헤더 전달 및 에러 전파 동작을 검증하는 단위 테스트를 유지·보강한다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 에러 상태 구조 변경:
  - 기존: `const [fileViewError, setFileViewError] = useState<string | null>(null);`
  - 변경: `const [fileViewErrors, setFileViewErrors] = useState<Record<string, string>>({});`
- 로딩 상태 구조 변경:
  - 기존: `const [viewingFileId, setViewingFileId] = useState<string | null>(null);`
  - 변경: `const [viewingFileKeys, setViewingFileKeys] = useState<Record<string, boolean>>({});`
- `handleViewSourceFile` 핸들러 개선:
  - 인자로 `documentId: string`, `fileName: string`, `groupKey: string`을 전달받도록 수정한다.
  - 비동기 호출 전 `window.open('', '_blank')` 사전 빈 탭 생성 코드를 완전히 제거한다.
  - 해당 `groupKey`에 대한 로딩 상태(`viewingFileKeys[groupKey] = true`)를 활성화하고 기존 에러를 초기화한다.
  - `const blob = await getDocumentFile(documentId)` 호출로 Blob 수신 후, `<a download={fileName}>` 엘리먼트를 동적으로 생성·클릭하여 브라우저 표준 파일 다운로드를 실행한다.
  - 다운로드 완료 후 `URL.revokeObjectURL`을 안전하게 호출하여 메모리 누수를 방지한다.
  - 예외 발생 시 `fileViewErrors[groupKey] = '원본 파일을 불러오는 데 실패했습니다.'`로 해당 출처 그룹에만 국소적으로 에러를 기록한다.
- 렌더링부 수정:
  - 완료된 AI 메시지 버블(`AiService.tsx:1655`): `handleViewSourceFile(group.documentId, group.fileName, groupKey)` 호출로 변경하고, 에러 표시부(`AiService.tsx:1690`)를 `fileViewErrors[groupKey]`가 존재할 때만 해당 그룹 내부에 국소 렌더링하도록 수정한다.
  - 스트리밍 중인 최신 AI 메시지 버블(`AiService.tsx:1800, 1835`): 동일하게 `groupKey` 기반 로딩/에러 바인딩을 적용한다.

`public-front/src/components/AiService.test.tsx`
- [원문 보기] 버튼 클릭 시 `window.open` 호출 없이 `getDocumentFile`이 호출되고 앵커 다운로드가 트리거되는지 검증하는 단위 테스트
- 특정 문서 원문 다운로드 실패 시 해당 출처 영역에만 에러 메시지가 노출되고, 다른 AI 메시지 버블의 출처 영역에는 에러가 노출되지 않음을 검증하는 상태 격리 단위 테스트
- 다운로드 진행 중 해당 버튼만 "불러오는 중…"으로 변경되고 다른 문서 그룹의 버튼은 비활성화되지 않음을 검증하는 단위 테스트
- 세션 복원 대화 및 스트리밍 대화에서 원문 다운로드 동작이 일관되게 동작함을 검증하는 단위 테스트

## 수용 기준 (Acceptance Criteria)
- Given AI 답변 버블의 참고 문서 목록에 특정 문서 출처가 표시되어 있을 때 When 사용자가 해당 문서 행의 [원문 보기] 버튼을 클릭하면 Then 사전 빈 탭(`about:blank`)이나 팝업 차단 경고창이 뜨지 않고, `getDocumentFile` API가 해당 `documentId`로 호출된 후 브라우저 표준 다운로드(`a.download = fileName`)가 정상적으로 트리거된다.
- Given 대화창에 2개 이상의 AI 답변 메시지가 렌더링되어 있고 각각 참고 문서가 표시된 상태에서 When 첫 번째 답변의 특정 출처 문서 원문 조회가 네트워크 오류 등으로 실패하면 Then 첫 번째 답변의 해당 출처 그룹 하단에만 "원본 파일을 불러오는 데 실패했습니다." 에러가 노출되고, 두 번째 답변 버블의 출처 목록에는 에러 문구가 전혀 노출되지 않는다.
- Given 특정 문서 원문 다운로드가 진행 중인 상황에서 When 출처 영역을 확인하면 Then 다운로드가 요청된 해당 버튼만 "불러오는 중…" 텍스트로 변경되고 비활성화(`disabled`)되며, 동일 대화창 내 다른 문서 출처의 [원문 보기] 버튼은 영향을 받지 않는다.
- Given 세션 복원(`handleLoadSession`)으로 과거 대화 내역을 불러온 상황에서 When 특정 AI 메시지의 출처 문서에서 [원문 보기]를 클릭하면 Then 동일하게 빈 탭 잔류 없이 해당 문서 파일명으로 다운로드가 정상 실행된다.
- Given 파일 다운로드가 실패하여 에러 메시지가 표시된 출처 그룹에서 When 사용자가 [원문 보기]를 다시 클릭하면 Then 이전에 표시되었던 에러 메시지가 즉시 초기화되고 새로운 다운로드 요청이 실행된다.

## 참고
- 고쳐야 할 자리 (원문 조회 핸들러): `public-front/src/components/AiService.tsx:361-381` (`handleViewSourceFile`)
- 고쳐야 할 자리 (완료 답변 출처 렌더링 및 에러 노출부): `public-front/src/components/AiService.tsx:1655-1659, 1690-1694` (`AiService`)
- 고쳐야 할 자리 (스트리밍 답변 출처 렌더링 및 에러 노출부): `public-front/src/components/AiService.tsx:1800-1804, 1835-1839` (`AiService`)
- 원문 파일 프록시 컨트롤러 입구: `public-server/apps/gateway/src/ai/proxy/knowledge-proxy.controller.ts:41-51` (`KnowledgeProxyController.getFile`)
- 원본 파일 스트리밍 API 클라이언트: `public-front/src/api/ai.ts:425-430` (`getDocumentFile`)
- 백엔드 원본 파일 라우터: `public-python-server/src/ai_service/knowledge/router.py:36-47` (`get_document_file`)
