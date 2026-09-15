---
id: SPEC-045
title: 시스템 프롬프트 설정 모달 컴포넌트 분리 및 상태 관리 모듈화
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 `public-front/src/components/AiService.tsx`는 1600줄이 넘는 단일 거대 컴포넌트로 구성되어 있으며, 문서 업로드/관리, RAG 실시간 대화 스트리밍, 대화 세션 및 북마크 사이드바와 함께 "개인 시스템 프롬프트 설정 모달" 관련 상태와 비즈니스 로직이 한곳에 강하게 결합되어 있다.

- `public-front/src/components/AiService.tsx:110-120` (`AiService`)에서 시스템 프롬프트 설정 모달 전용 상태(`isPromptSettingsOpen`, `myPrompt`, `promptList`, `promptDraft`, `promptLoading`, `promptSaving`, `promptError`, `promptSuccessMsg` 등 8개 이상의 상태)를 메인 컴포넌트 최상단에서 직접 선언하고 관리한다.
- `public-front/src/components/AiService.tsx:533-622` (`handleOpenPromptSettings`, `handleSavePrompt`, `handleActivateSlot`, `handleDeleteSlot`, `handleResetPrompt`)에서 프롬프트 조회, 슬롯 저장, 버전 활성화, 슬롯 삭제, 기본값 초기화 등 모달 내부에서만 사용되는 비즈니스 핸들러가 컴포넌트 본문에 나열되어 있다.
- `public-front/src/components/AiService.tsx:1318-1547` (`isPromptSettingsOpen` 렌더링 블록)에서 230줄에 달하는 모달 인라인 JSX가 메인 렌더 트리에 직접 포함되어 있어, 프롬프트 입력 및 슬롯 조작 시 `AiService` 전체가 불필요하게 리렌더링되고 코드 유지보수성이 저하된다.

이로 인해 RAG 답변 및 지식베이스 관련 핵심 로직을 수정하거나 테스트할 때 프롬프트 모달 상태와의 결합으로 인해 회귀 버그 발생 위험이 높다. 프롬프트 설정 모달을 독립 컴포넌트(`PromptSettingsModal`)로 분리하여 책임을 명확히 격리해야 한다.

## 요구사항
- [ ] 시스템 프롬프트 설정 모달을 전담하는 독립 컴포넌트 `public-front/src/components/ai/PromptSettingsModal.tsx`를 생성한다.
- [ ] `PromptSettingsModal`은 `isOpen`, `onClose` props를 수신하여 모달 열림/닫힘 상태를 제어하고, 모달 내부에서 프롬프트 조회/저장/활성화/삭제/초기화 상태 및 핸들러를 캡슐화한다.
- [ ] 모달 배경 클릭(`onClick={(e) => { if (e.target === e.currentTarget) onClose(); }}`) 및 닫기 버튼 클릭 시 모달이 닫혀야 한다.
- [ ] 기존 `AiService.tsx`에서 프롬프트 모달 전용 상태 및 중복 핸들러 코드를 제거하고, `PromptSettingsModal` 컴포넌트를 import하여 연결한다.
- [ ] 슬롯 목록 표시(최대 10개), 새 프롬프트 작성/저장, 슬롯별 활성화, 슬롯 삭제, 기본값 초기화 등 기존 UI 및 기능 동작이 완벽하게 동일하게 유지되어야 한다.
- [ ] 분리된 `PromptSettingsModal`에 대한 독립 단위 테스트(`PromptSettingsModal.spec.tsx` 또는 `PromptSettingsModal.test.tsx`)를 작성하여 모달 렌더링, 프롬프트 저장, 버전 활성화/삭제, 에러 메시지 노출 동작을 검증한다.

## 비요구사항 (Out of scope)
- 백엔드(`public-server`, `public-python-server`) 프롬프트 API 스키마나 라우터 변경.
- 문서 삭제 확인 모달이나 채팅 대화창 등 `AiService` 내 다른 하위 기능의 추가 분리.
- 개인 프롬프트 최대 슬롯 제한(10개) 정책이나 슬롯 버전 관리 비즈니스 규칙 변경.
- 새로운 스타일 테마나 UI 프레임워크 도입.

## 프론트엔드
- `public-front/src/components/ai/PromptSettingsModal.tsx`:
  - `PromptSettingsModalProps` 정의 (`isOpen: boolean; onClose: () => void; onPromptUpdated?: () => void;`).
  - 모달 내부에서 `getMyPrompt`, `getMyPromptList`, `saveMyPrompt`, `activateMyPrompt`, `deleteMyPromptVersion`, `resetMyPrompt` API 호출 및 로컬 상태 관리.
- `public-front/src/components/AiService.tsx`:
  - 불필요해진 모달 내부 상태(`promptDraft`, `promptLoading`, `promptSaving`, `promptError`, `promptSuccessMsg`, `myPrompt`, `promptList` 등) 및 핸들러(`handleSavePrompt`, `handleActivateSlot`, `handleDeleteSlot`, `handleResetPrompt`) 제거.
  - 모달 오픈 플래그(`isPromptSettingsOpen`)와 모달 트리거 핸들러만 유지하고 `<PromptSettingsModal isOpen={isPromptSettingsOpen} onClose={() => setIsPromptSettingsOpen(false)} />`로 교체.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 AI 서비스 화면에 진입하여 시스템 프롬프트 설정 버튼을 클릭했을 때
  When `isPromptSettingsOpen`이 `true`로 설정되면
  Then `PromptSettingsModal`이 열리고 현재 활성화된 프롬프트와 슬롯 목록(최대 10개)이 로드되어 렌더링된다.
- Given 사용자가 새 프롬프트를 입력하고 '새 슬롯으로 저장 및 적용' 버튼을 눌렀을 때
  When API 호출이 성공하면
  Then 새로운 슬롯이 추가되고 성공 메시지("새 프롬프트 슬롯이 저장되고 활성화되었습니다.")가 표시된다.
- Given 프롬프트 내용이 비어있거나 슬롯 제한(10개)에 도달했을 때
  When 저장 버튼을 누르면
  Then API 호출을 방지하고 유효성 에러 메시지가 화면에 노출된다.
- Given 모달이 열려 있는 상태에서
  When 모달 배경 오버레이 영역을 클릭하거나 '닫기' 버튼을 누르면
  Then `onClose` 콜백이 호출되어 모달이 정상적으로 닫힌다.

## 참고
- 메인 컴포넌트 프롬프트 상태 정의: `public-front/src/components/AiService.tsx:110-121` (`AiService`)
- 메인 컴포넌트 프롬프트 API 핸들러: `public-front/src/components/AiService.tsx:533-622` (`handleOpenPromptSettings`)
- 메인 컴포넌트 프롬프트 모달 JSX: `public-front/src/components/AiService.tsx:1318-1547` (`isPromptSettingsOpen`)
- 프롬프트 API 클라이언트 정의: `public-front/src/api/ai.ts:220-272` (`getMyPrompt`, `saveMyPrompt`, `activateMyPrompt`, `deleteMyPromptVersion`, `resetMyPrompt`)
