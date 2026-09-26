---
id: SPEC-056
title: AI 답변 메시지별 근거 출처(참고 문서) 표시 및 다중 턴 출처 유지
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 지식베이스 기반 질의응답(RAG) 기능은 검색된 문서 청크를 기반으로 답변을 생성하고 근거 문서 목록을 제공한다. 백엔드는 다회차 검색 출처 누적(SPEC-047)과 세션 턴별 출처 저장(SPEC-011)을 지원하고 있으며, 세션 상세 조회 API(`getSessionDetail`) 응답의 각 턴(`turn.sources`)에도 개별 질의 시점의 참고 문서 정보가 정상적으로 포함되어 있다.

그러나 프론트엔드 UI 컴포넌트(`public-front/src/components/AiService.tsx`)에서는 대화 로그의 각 메시지 객체에 출처 목록을 보존하지 않고, 전체 대화창 하단에 단일 전역 상태(`currentSources`)로만 출처를 렌더링하고 있어 다음과 같은 심각한 사용성 및 신뢰도 결함이 발생한다.

1. `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 인터페이스에 `sources` 필드가 누락되어 있어, 스트리밍 완료 시점(`public-front/src/components/AiService.tsx:672-680`, `handleSendQuestion`)에 수신된 출처 배열이 메시지 로그(`chatLog`)에 저장되지 않고 버려진다.
2. 사용자가 후속 질문을 입력하여 전송하면 `public-front/src/components/AiService.tsx:658` (`handleSendQuestion`)에서 `setCurrentSources([])`가 실행되어 직전 답변의 참고 문서 영역이 대화창에서 완전히 사라진다.
3. 세션 목록에서 이전 대화를 불러올 때(`public-front/src/components/AiService.tsx:323-336`, `handleLoadSession`), 백엔드가 각 턴별로 보존한 `sources` 데이터를 `chatLog` 메시지에 매핑하지 않고 마지막 어시스턴트 턴의 출처 1개만 `currentSources`에 덮어쓴다.
4. `public-front/src/components/AiService.tsx:1094-1165` (`chatLog.map`) 루프의 개별 AI 메시지 버블 내부에는 출처 UI가 전혀 렌더링되지 않으며, `public-front/src/components/AiService.tsx:1197-1272`와 같이 대화창 최하단에 단일 영역으로 분리되어 있어 어떤 답변이 어떤 문서를 근거로 작성되었는지 시각적으로 매핑되지 않는다.

이로 인해 다중 턴 대화 환경에서 사용자가 과거 답변의 근거를 재확인할 수 없고, 이전에 배송된 답변 근거 미리보기(SPEC-006) 및 세션 복원 시 근거 복원(SPEC-011) 작업의 성과가 다중 턴 대화 시 무력화되고 있다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `sources?: SourceRef[]` 선택적 필드를 추가한다.
- [ ] `handleSendQuestion`에서 스트리밍 완료 콜백(`onDone`) 실행 시, 현재 질문에 대해 수신된 출처 목록(`currentSources`)을 신규 추가되는 AI 메시지 객체(`sources`)에 포함하여 `chatLog`에 영속화한다.
- [ ] `handleLoadSession`에서 세션 복원 시 `detail.turns`의 각 턴(`t.sources`)을 `chatLog`의 해당 AI 메시지 객체 `sources` 필드로 1:1 매핑하여 복원한다.
- [ ] 각 AI 메시지 버블(`chat-message ai`) 내부에 해당 답변의 `sources` 목록을 렌더링하여, 개별 답변 버블마다 독립적으로 참고 문서 접기/펼치기 및 원문 보기(`handleViewSourceFile`)가 정상 동작하도록 구성한다.
- [ ] 스트리밍 중인 최신 답변의 경우, 스트리밍 진행 중 실시간으로 수신된 `currentSources`가 스트리밍 버블 하단에 자연스럽게 표시되도록 처리한다.
- [ ] 대화창 최하단에 단일 전역으로 고정 렌더링되던 기존 중복 출처 영역을 제거하고, 메시지 버블 내부 렌더링 구조로 일원화한다.
- [ ] 과거 세션 데이터나 일반 대화처럼 `sources`가 없거나 빈 배열(`[]`)인 AI 메시지 버블에서는 참고 문서 영역이 렌더링되지 않고 기존 텍스트/신뢰도 배지만 안전하게 표시된다.

## 비요구사항 (Out of scope)
- 백엔드 Python 서버(`public-python-server`) 및 NestJS 게이트웨이(`public-server`)의 API 스키마나 DTO 구조 변경은 진행하지 않는다. (이미 백엔드 계약 및 영속화는 SPEC-011, SPEC-047을 통해 완성되어 있음)
- 출처 문서 청크에 대한 하이라이팅 위치 이동이나 문서 뷰어 컴포넌트의 신규 개발은 포함하지 않는다. (기존 모달 기반 원문 보기 로직을 그대로 재사용)
- 다중 턴 간 출처 문서 필터링이나 검색 알고리즘 변경은 다루지 않는다.

## 프론트엔드
- `public-front/src/components/AiService.tsx`
  - `ChatMessage` 인터페이스에 `sources?: SourceRef[]` 필드 정의
  - `handleSendQuestion` 스트리밍 완료 핸들러에서 `chatLog` 업데이트 시 `sources` 포함
  - `handleLoadSession` 세션 복원 로직에서 `detail.turns` 매핑 시 `sources` 바인딩
  - 개별 메시지 버블(`chat-message ai`) 내부에 문서 그룹화(`groupSourcesByDocument`), 청크 목록, 스니펫, 관련도 점수, 원문 보기 버튼 렌더링
  - 각 메시지 또는 문서 그룹별 접힘/펼침 상태 독립 제어
- `public-front/src/components/AiService.css` (필요 시)
  - 메시지 버블 내부의 참고 문서 영역 레이아웃 및 간격 스타일 정돈

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스 문서를 업로드하고 첫 번째 질문을 완료했을 때
  When 답변 생성이 완료되면
  Then 해당 AI 메시지 버블 하단에 참고 문서 목록 및 근거 청크 수가 올바르게 렌더링된다.
- Given 첫 번째 질문에 대한 답변과 출처가 표시된 상태에서
  When 사용자가 두 번째 후속 질문을 전송하고 새 답변을 받았을 때
  Then 첫 번째 AI 메시지 버블의 참고 문서 목록이 그대로 유지되어 사라지지 않고, 두 번째 AI 메시지 버블에도 새 출처가 독립적으로 표시된다.
- Given 다중 턴 대화 이력이 저장된 세션이 존재할 때
  When 세션 목록에서 해당 세션을 클릭하여 복원(`handleLoadSession`)하면
  Then 대화창의 모든 과거 AI 답변 메시지에 각 턴에 해당하는 참고 문서 목록이 개별적으로 복원되어 렌더링된다.
- Given 검색된 지식베이스 문서가 없거나 단발성 일반 대화인 경우 (`sources`가 `undefined`이거나 빈 배열 `[]`)
  When 답변 메시지가 렌더링될 때
  Then 참고 문서 영역이 화면에 나타나지 않고 텍스트 및 기존 배지만 깨짐 없이 정상 렌더링된다.
- Given AI 메시지 버블 내부에 참고 문서 목록이 렌더링되었을 때
  When 사용자가 특정 참고 문서의 "원문 보기" 버튼을 클릭하면
  Then 해당 문서 ID로 `handleViewSourceFile`이 정상 호출되어 원문 조회 모달이 열린다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:50-55` (`ChatMessage` 인터페이스)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:323-328` (`handleLoadSession`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:658` (`handleSendQuestion` 진입 시 초기화)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:672-680` (`handleSendQuestion` 스트리밍 완료 콜백)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1094-1165` (`chatLog.map` 메시지 렌더링 루프)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1197-1272` (기존 하단 단일 출처 렌더링 영역)
- 관련 백엔드 세션 DTO 정의: `public-front/src/api/ai.ts:80-92` (`SessionTurn`, `SessionDetailOut`)
