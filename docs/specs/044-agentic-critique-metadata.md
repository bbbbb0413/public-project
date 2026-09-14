---
id: SPEC-044
title: RAG 비평 메타데이터 및 다중 턴 추론 요약 전달 연동
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 Agentic RAG 질의응답 루프(`AgenticAskUseCase`)는 반복 회차마다 비평(`Critique`)을 수행하여 답변의 완성도를 자체 평가하지만, 비평 결과 메타데이터가 프론트엔드로 온전히 전달되지 않거나 덮어쓰여 다중 턴 환경에서 추론 검증 정보를 유실하고 있다.

- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:120-138` (`AgenticAskUseCase.execute`)에서 매 반복 회차마다 `_critique_generator.generate`를 호출해 `confidence`, `missing`, `next_query`를 평가하고 `_emit_progress`로 전달하지만, 스트리밍 generator 자체는 최종 텍스트 토큰(`yield`)만 방출한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:118-124` (`on_progress`) 및 `:196-203` (`publish_done`)에서 최종 완료 시 마지막 회차의 단일 `last_confidence`와 `last_missing`만 `done` 이벤트 페이로드로 발행하여, 이전 반복 회차들의 개선 과정 요약(총 반복 횟수, 단계별 개선 내역)이 프론트엔드로 전달되지 않는다.
- `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`) 및 `public-front/src/components/AiService.tsx:672-680` (`handleSendQuestion`)에서 스트리밍 완료 시 수신한 `finalMeta`의 `confidence`와 `missing` 정보만 AI 메시지 객체에 반영하고 있어, 다중 턴 대화 시 각 답변이 어떤 추론 반복 과정을 거쳐 도출되었는지 사용자가 검증할 수 없다.

이는 "답을 그대로 믿으라고 요구하지 않는다 — 사용자가 스스로 확인할 수 있어야 한다"는 이번 분기 핵심 제품 방향에 부합하지 않으며, 신뢰도와 누락 정보 외에도 에이전트의 다단계 비평 검증 결과를 투명하게 노출하여 사용자의 답변 신뢰성을 높여야 한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`에서 Agentic RAG 완료(`done`) 이벤트 페이로드에 총 반복 횟수(`iterations`) 및 최종 신뢰도/누락 정보를 포함하여 발행한다.
- [ ] 단발성 비에이전틱 질의(`complexity != "complex"`)의 경우 기본 반복 횟수(`iterations: 1`) 또는 기존 `done` 페이로드 규격을 안전하게 유지한다.
- [ ] `public-front/src/api/ai.ts`의 `AskDoneMeta` 인터페이스에 `iterations?: number` 필드를 추가하고 `askQuestionStream`의 `onDone` 콜백에 전달한다.
- [ ] `public-front/src/components/AiService.tsx`의 `ChatMessage` 인터페이스에 `iterations?: number` 필드를 추가하고, 스트리밍 완료(`onDone`) 시 AI 메시지 객체에 영속화한다.
- [ ] `public-front/src/components/AiService.tsx`의 AI 메시지 버블에서 신뢰도 배지 옆에 총 추론 반복 회차(예: `[2회 반복 검증]`) 배지를 렌더링한다. (단, 단발성 1회 질의인 경우 생략하거나 기본 표시)
- [ ] 변경 사항에 대해 백엔드 컨슈머 완료 이벤트 페이로드 구성 및 프론트엔드 메시지 버블 렌더링 검증 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 비평 프롬프트 자체를 수정하거나 새로운 LLM 비평 모델을 도입하는 작업.
- 중간 반복 단계의 미완성 원시 텍스트나 실패한 임시 답변 초안 전체를 사용자에게 노출하는 작업.
- Gateway(`public-server`)의 Redis Streams 중계 로직 수정 (기존 범용 릴레이 메커니즘 유지).
- 대화 세션 MongoDB 스키마의 마이그레이션이나 필수 필드 강제화.

## 백엔드
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `on_progress` 핸들러에서 최종 실행 회차(`last_iteration`)를 추적하도록 보강.
  - `publish_done` 호출 시 `done_data` 페이로드에 `iterations: last_iteration`을 포함하여 발행.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `AskDoneMeta` 타입에 `iterations?: number` 추가.
- `public-front/src/components/AiService.tsx`:
  - `ChatMessage` 인터페이스에 `iterations?: number` 추가.
  - `handleSendQuestion`의 `onDone` 콜백에서 `finalMeta?.iterations`를 수신하여 `chatLog`의 AI 메시지 객체에 저장.
  - 메시지 버블 신뢰도 배지 영역에 반복 검증 회차 시각화 배지 추가.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG가 2회 이상 비평 루프를 거쳐 답변을 생성했을 때
  When 스트리밍이 완료되고 `done` 이벤트가 수신되면
  Then AI 메시지 버블에 최종 신뢰도와 함께 `[N회 반복 검증]` 배지가 렌더링된다.
- Given 비에이전틱 질의이거나 반복 정보가 없는 레거시 응답을 수신했을 때
  When 메시지가 렌더링되면
  Then 화면 에러 없이 기존과 동일하게 신뢰도 배지만 정상 렌더링된다.
- Given 신뢰도 및 누락 정보(`missing`)가 존재하는 경우
  When 답변이 화면에 표시되면
  Then 반복 회차 배지, 신뢰도 배지, 확인하지 못한 항목 목록이 순서대로 깨짐 없이 정렬되어 표시된다.

## 참고
- 비평 생성 및 루프 처리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:120-138` (`AgenticAskUseCase.execute`)
- 이벤트 수신 및 완료 통지: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:118-124`, `:196-203` (`on_progress`, `publish_done`)
- 프론트엔드 메시지 상태 관리: `public-front/src/components/AiService.tsx:50-55`, `:672-680` (`ChatMessage`, `handleSendQuestion`)
