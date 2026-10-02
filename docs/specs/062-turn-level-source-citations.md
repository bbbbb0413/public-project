---
id: SPEC-062
title: AI 답변 메시지별 근거 출처(참고 문서) 표시 및 다중 턴 출처 유지
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 프론트엔드 RAG 대화창(`public-front/src/components/AiService.tsx`)은 수신된 참고 문서(출처)를 대화창 최하단 단일 전역 영역(`currentSources`)에만 렌더링하고 있다.

이로 인해 다음과 같은 결함과 사용성 문제가 발생한다:
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 인터페이스에 `sources` 필드가 누락되어 있어, 각 AI 답변 메시지가 개별 근거 목록을 소유하지 못한다.
- `public-front/src/components/AiService.tsx:672-682` (`handleSendQuestion`)에서 스트리밍 완료 시 생성된 AI 메시지에 `sources`를 저장하지 않고 오직 컴포넌트 전역 상태인 `currentSources`에만 의존한다. 따라서 사용자가 다음 질문을 전송하면 `setCurrentSources([])`로 초기화되어 이전 질문의 답변 근거가 화면에서 완전히 사라진다.
- `public-front/src/components/AiService.tsx:323-336` (`handleLoadSession`)에서 기존 대화 세션을 복원할 때 백엔드 `public-front/src/api/ai.ts:300-305` (`SessionTurn`)로부터 턴별 `sources`를 정상 전달받음에도 불구하고, 마지막 AI 턴의 `sources` 1건만 `currentSources`로 복원하고 나머지 이전 턴들의 출처는 전부 무시한다.
- `public-front/src/components/AiService.tsx:1197-1272` (`AiService`)에서 참고 문서 목록을 전체 대화창 하단에 고정 렌더링하므로, 대화가 길어질수록 어떤 답변에 연결된 출처인지 식별하기 어렵고 SPEC-006(답변 근거 미리보기) 및 SPEC-011(대화 세션 복원 시 출처 복원)의 작업 성과를 화면에서 지우는 문제가 발생한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `sources?: SourceRef[]` 선택적 필드를 추가한다.
- [ ] 질문 스트리밍 완료 시점(`handleSendQuestion`의 `onDone`)에 수신된 출처 목록(`currentSources`)을 새로 추가되는 AI 메시지 객체의 `sources` 필드에 포함하여 `chatLog`에 보존한다.
- [ ] 세션 복원 시점(`handleLoadSession`)에 `SessionDetailOut.turns`의 각 턴에 포함된 `sources`를 `chatLog`의 각 AI 메시지 객체 `sources` 필드로 1:1 매핑하여 복원한다.
- [ ] 각 AI 메시지 버블(`chat-message ai`) 내부에 해당 메시지만의 참고 문서 목록(`source-refs`)을 렌더링하여, 다중 턴 대화에서도 과거 답변의 출처가 사라지지 않고 개별 버블 안에서 유지되도록 한다.
- [ ] 개별 AI 메시지 버블 내 참고 문서의 그룹별 접기/펼치기(`toggleGroup`) 및 원문 보기(`handleViewSourceFile`) 기능이 메시지 턴 단위로 독립적으로 정상 동작해야 한다.
- [ ] 스트리밍 중인 최신 답변의 경우, 스트리밍 진행 중 실시간으로 수신된 `currentSources`가 스트리밍 버블 내부 또는 하단에 자연스럽게 표시되도록 한다.
- [ ] 대화창 최하단에 단일 전역으로 고정 렌더링되던 기존 중복 출처 영역(`!isStreaming && currentSources.length > 0`)을 제거하고 메시지 버블 내부 렌더링 구조로 일원화한다.

## 비요구사항 (Out of scope)
- 백엔드(NestJS Gateway, Python AI Service)의 RAG 검색 알고리즘 및 출처 생성 로직 수정
- 출처 문서의 실시간 하이라이팅 또는 PDF 뷰어 내 앵커 스크롤 이동 기능 신설
- 세션/턴 저장소의 스키마 변경이나 DB 마이그레이션

## 프론트엔드
- 대상 컴포넌트: `public-front/src/components/AiService.tsx`
- 대상 스타일: `public-front/src/components/AiService.css`
- 주요 변경:
  - `ChatMessage` 타입 정의에 `sources?: SourceRef[]` 추가
  - `handleSendQuestion` 내 스트리밍 완료 시 `chatLog` 메시지 객체에 `sources` 할당
  - `handleLoadSession` 내 턴 데이터 매핑 시 `sources` 필드 보존
  - AI 메시지 버블 렌더링 영역 내부에 개별 `source-refs` 목록 및 접기/펼치기, 원문 보기 UI 배치

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스 기반 대화 세션에서 첫 번째 질문에 대한 답변과 2건의 참고 문서가 수신된 상태에서
  When 사용자가 두 번째 질문을 전송하고 두 번째 답변이 완료되었을 때
  Then 첫 번째 AI 답변 버블 내부에는 첫 번째 답변의 참고 문서 2건이 그대로 유지되어 표시되고, 두 번째 AI 답변 버블 내부에는 두 번째 답변의 참고 문서가 각각 독립적으로 렌더링되어야 한다.
- Given 과거 다중 턴 질의응답 이력과 각 턴별 `sources` 데이터가 저장된 세션이 있을 때
  When 사용자가 사이드바에서 해당 대화 세션을 클릭하여 복원(`handleLoadSession`)했을 때
  Then 각 AI 답변 메시지 버블마다 해당 턴에 속한 참고 문서 목록이 올바르게 복원되어 렌더링되어야 한다.
- Given 출처 정보가 없는 단발성 일반 대화이거나 `sources`가 빈 배열(`[]`) 또는 `undefined`인 레거시 AI 메시지일 때
  When 해당 메시지가 화면에 렌더링될 때
  Then 참고 문서 영역이 렌더링되지 않고 기존 텍스트 및 신뢰도 배지만 정상적으로 표시되어야 한다 (경계 케이스).
- Given 특정 AI 메시지 버블 내부의 참고 문서 항목이 접혀 있는 상태에서
  When 사용자가 참고 문서 제목 또는 토글 버튼을 클릭했을 때
  Then 해당 메시지 내의 해당 문서 청크 목록 및 스니펫만 펼쳐지고 다른 메시지 버블의 참고 문서 상태에는 영향을 주지 않아야 한다.

## 참고
- 고쳐야 할 인터페이스 및 핸들러: `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`), `public-front/src/components/AiService.tsx:323-336` (`handleLoadSession`), `public-front/src/components/AiService.tsx:672-682` (`handleSendQuestion`)
- 기존 전역 출처 렌더링 영역: `public-front/src/components/AiService.tsx:1197-1272` (`AiService`)
- 출처 정의 및 세션 턴 규약: `public-front/src/api/ai.ts:300-305` (`SessionTurn`), `public-front/src/api/ai.ts:26` (`SourceRef`)
