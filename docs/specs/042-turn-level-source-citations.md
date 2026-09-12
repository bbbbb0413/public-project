---
id: SPEC-042
title: AI 답변 메시지별 근거 출처(참고 문서) 표시 및 다중 턴 출처 유지
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 RAG 대화 화면(`public-front/src/components/AiService.tsx`)에서 답변의 근거가 되는 참고 문서(`SourceRef`)는 전체 채팅창 맨 하단에 단일 전역 상태(`currentSources`)로만 렌더링되고 있다. 이로 인해 다중 턴 대화 및 세션 복원 흐름에서 아래와 같은 결함이 발생한다.
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 인터페이스에 `sources` 필드가 누락되어 있어, 스트리밍 중 수신된 출처 목록이 개별 메시지 상태(`chatLog`)에 저장되지 않는다.
- `public-front/src/components/AiService.tsx:672-682` (`handleSendQuestion`의 `onDone`)에서 스트리밍 완료 시 `chatLog`에 새 AI 메시지를 추가할 때 해당 턴의 출처 목록을 메시지에 담지 않고 전역 `currentSources`에만 의존한다. 결과적으로 사용자가 2번째 이상의 후속 질문을 하면 이전 AI 답변의 근거 문서와 청크 정보가 화면에서 완전히 사라진다.
- `public-front/src/components/AiService.tsx:323-336` (`handleLoadSession`)에서 저장된 세션을 불러올 때 `detail.turns`의 개별 턴에 저장된 `sources` 메타데이터(SPEC-011로 백엔드에 구현됨)를 `chatLog`에 복원하지 않고, 오직 가장 마지막 어시스턴트 턴의 `sources` 하나만 전역 `currentSources`에 할당한다. 이로 인해 이전 턴들의 근거 정보가 화면에 복원되지 않는다.
- `public-front/src/components/AiService.tsx:1197-1260` (`AiService` 렌더링)에서 참고 문서 영역(`source-refs`)이 각 AI 메시지 버블 내부가 아닌 채팅 영역 하단에 분리 렌더링되어 있어, 대화가 길어질수록 어떤 답변에 대한 출처인지 사용자가 직관적으로 연결하여 확인하기 어렵다.
- 이는 "사용자가 답변의 근거를 스스로 확인하고 신뢰할 수 있어야 한다"는 이번 분기 핵심 제품 방향과 직결되며, 이미 배송된 SPEC-006(답변 근거 미리보기) 및 SPEC-011(세션 출처 복원)의 가치를 살려 다중 턴에서도 일관되게 근거를 검증할 수 있도록 바로잡아야 한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `sources?: SourceRef[]` 필드를 추가한다.
- [ ] `handleSendQuestion`의 스트리밍 완료 콜백(`onDone`) 실행 시, 현재 질문 스트리밍 중 수신된 `currentSources`를 새로 추가되는 AI 메시지 객체(`{ sender: 'ai', text: accumulated, confidence, missing, sources }`)에 포함하여 `chatLog`에 영속화한다.
- [ ] `handleLoadSession`에서 세션 복원 시 `detail.turns`의 각 턴(`t.sources`)을 `chatLog` 메시지의 `sources` 필드로 1:1 매핑하여 복원한다.
- [ ] 각 AI 메시지 버블(`chat-message ai`) 내부에 해당 답변의 참고 문서 목록(`source-refs`)을 렌더링하여, 개별 답변 버블마다 독립적으로 근거 문서 접기/펼치기 및 원문 보기(`handleViewSourceFile`)가 동작하도록 한다.
- [ ] 스트리밍 중인 최신 답변의 경우, 스트리밍 진행 중 수신된 실시간 출처 목록이 스트리밍 버블 하단에 자연스럽게 표시되도록 한다.
- [ ] 과거 레거시 세션 데이터처럼 `sources`가 없거나 빈 배열(`[]`)인 AI 메시지 버블에서는 참고 문서 영역이 렌더링되지 않고 기존과 동일하게 텍스트/신뢰도 배지만 정상 노출된다.
- [ ] 프론트엔드 단위 테스트(`public-front/src/components/AiService.test.tsx`)에 다중 턴 대화 시 각 AI 메시지별 독립된 참고 문서 렌더링 및 세션 복원 시 모든 턴의 출처 복원 검증 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 백엔드(NestJS Gateway, Python AI Service)의 이벤트 스키마나 DB 엔티티 수정 (이미 백엔드는 세션 턴별 `sources` 저장을 지원하고 있음).
- 원문 문서 다운로드 및 뷰어 팝업 로직의 변경 (`handleViewSourceFile` 기존 함수 재사용).
- 문서 청킹 또는 하이브리드 검색 알고리즘 및 점수 산정 방식 변경.

## 프론트엔드
- 대상 컴포넌트: `public-front/src/components/AiService.tsx`, `public-front/src/components/AiService.css`
- `ChatMessage` 타입 정의 변경:
  - `sources?: SourceRef[]` 필드 추가
- 상태 관리 및 핸들러 수정:
  - `handleSendQuestion`: `onDone` 시점에 스트리밍 중 축적된 `sources` 배열을 메시지 객체에 포함하여 `chatLog`에 추가
  - `handleLoadSession`: `detail.turns.map(t => ({ ..., sources: t.sources }))`로 턴별 출처 매핑
  - `expandedGroups`: 단일 documentId 기반 키에서 메시지 인덱스와 문서 ID를 조합한 고유 키(`${msgIdx}-${documentId}`)로 관리하여 턴 간 펼침 상태 간섭 방지
- UI 렌더링 구조 개선:
  - `chatLog.map((msg, idx) => ...)` 루프 내부의 AI 메시지 버블 하단(`msg.sender === 'ai' && msg.sources && msg.sources.length > 0`)에 `source-refs` 렌더링 로직 배치
  - 최하단 전역 `currentSources` 단독 렌더링 블록 제거 (또는 스트리밍 진행 중에만 임시 노출되도록 정리)

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 세션 내에서 1번째 질문을 완료하여 답변과 참고 문서 A가 표시된 상황
  When 사용자가 이어서 2번째 질문을 전송하고 답변과 참고 문서 B를 수신할 때
  Then 1번째 AI 답변 버블 아래에는 참고 문서 A가 유지되고, 2번째 AI 답변 버블 아래에는 참고 문서 B가 독립적으로 렌더링된다.
- Given 여러 턴의 질문-답변 및 출처가 저장된 대화 세션 ID가 존재하는 상황
  When 사이드바에서 해당 세션을 클릭하여 `handleLoadSession`이 호출될 때
  Then `chatLog`의 모든 AI 메시지 버블에 각 턴에 해당하는 참고 문서 목록이 올바르게 복원되어 표시된다.
- Given 출처 정보가 없는 단발성 질의이거나 구버전 세션 데이터인 상황
  When 메시지가 렌더링될 때
  Then 화면 에러 없이 참고 문서 영역만 생략되고 답변 본문과 신뢰도 정보가 정상 렌더링된다.
- Given 특정 AI 메시지 버블의 참고 문서 그룹 펼치기 버튼을 클릭했을 때
  When 해당 문서의 청크 목록이 펼쳐질 때
  Then 다른 AI 메시지 버블의 참고 문서 펼침 상태에는 영향을 주지 않는다.

## 참고
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage` 인터페이스 정의)
- `public-front/src/components/AiService.tsx:237-264` (`groupSourcesByDocument` 그룹화 함수)
- `public-front/src/components/AiService.tsx:323-336` (`handleLoadSession` 세션 복원 함수)
- `public-front/src/components/AiService.tsx:672-682` (`handleSendQuestion` 스트리밍 완료 핸들러)
- `public-front/src/components/AiService.tsx:1197-1260` (`AiService` 참고 문서 렌더링 JSX)
- `public-front/src/api/ai.ts:298-305` (`SessionTurn` 타입 정의)
