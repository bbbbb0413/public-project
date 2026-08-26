---
id: SPEC-014
title: AI 답변 클립보드 복사 및 답변 생성 중단 UI 연동
status: draft
targets: [front, python-server]
stages: [frontend, backend, qa]
priority: normal
---

> **폐기됨 (2026-08-26).** SPEC-015 가 대신한다.
>
> 이 문서는 세 저장소가 feature 브랜치에 남아 있던 상태에서 만들어졌다. 조사가
> `main` 이 아니라 뒤처진 작업트리를 읽었고, 그 결과 **1번 항목이 사실과 다르다**
> — 클립보드 복사는 SPEC-008 로 이미 머지되어 `AiService.tsx:543-551` 에 있다.
>
> 2번 항목(생성 중단 미연동)은 사실이며, SPEC-015 가 그 범위만 떼어 다룬다.
> 같은 일이 반복되지 않도록 실행 진입점에서 저장소를 기준 브랜치 최신으로
> 맞추도록 엔진을 고쳤다.

## 배경 / 문제
기배송된 기능 중 프론트엔드 UI 및 백엔드 취소 감지 흐름이 누락되어 사용자가 제공받아야 할 핵심 기능(클립보드 복사, 생성 중단)을 온전히 사용하지 못하고 있다.

1. **답변 복사 기능 미제공**: `public-front/src/components/AiService.tsx:490-520` 에서 AI가 답변한 메시지를 렌더링할 때 마크다운 텍스트와 함께 신뢰도 배지(`confidence-badge`), 미확인 항목(`missing-info`)을 함께 노출한다. 하지만 복사 버튼이 없어 사용자가 텍스트를 드래그하여 수동 복사해야 하며, 메타데이터 영역까지 원치 않게 복사되거나 서식이 깨지는 불편이 발생한다 (`SPEC-008` 배송 대상이었으나 UI 컴포넌트에 미반영).
2. **생성 중단 수단 부재 및 백엔드 취소 미감지**: `public-server/apps/gateway/src/ai/job/job.controller.ts:60-84` 및 `public-server/apps/gateway/src/ai/job/job-store.service.ts:51-57` 에 `DELETE /ai/jobs/:jobId` 엔드포인트와 Redis `job:{jobId}:cancelled` 키 등록 로직이 구현되어 있다. 그러나 `public-front/src/components/AiService.tsx:606-613` 에서는 스트리밍 중 전송 버튼이 '답변 중...'으로만 비활성화되고 중단 버튼이 없으며, `public-front/src/api/ai.ts:195-294` 의 `askQuestionStream` 에도 스트림 취소(Abort) 메커니즘이 누락되어 있다. 또한 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:143-157` 의 토큰 스트리밍 루프에서 Redis 취소 플래그를 확인하지 않고 끝까지 실행하여 불필요한 토큰 비용 및 대기 시간이 발생한다 (`SPEC-007` 배송 대상의 연동 누락).

이로 인해 이미 배송 완료로 기록된 작업의 성과가 지워진 상태이므로, 신뢰도 높은 RAG 답변 경험을 완성하기 위해 프론트엔드 복사 버튼과 스트리밍 중단 UI 및 취소 감지 흐름을 연동해야 한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx` 의 완료된 AI 메시지 버블(`msg.sender === 'ai'`)에 "복사" 버튼을 추가한다.
- [ ] 복사 버튼 클릭 시 `navigator.clipboard.writeText(msg.text)` 로 마크다운 텍스트를 클립보드에 복사하고, 2초간 버튼 텍스트를 "복사됨" 으로 변경한 후 복구한다.
- [ ] 복사 실패 시 `setErrorMsg('답변 복사에 실패했습니다.')` 로 에러 배너를 노출한다.
- [ ] 스트리밍 중인 임시 버블 및 사용자 메시지 버블에는 복사 버튼을 노출하지 않는다.
- [ ] `public-front/src/api/ai.ts` 의 `askQuestionStream` 에 `AbortSignal` 또는 취소 콜백을 지원하여 SSE 연결을 즉시 중단하고 `DELETE /ai/jobs/:jobId` 취소 API를 호출하도록 구현한다.
- [ ] `public-front/src/components/AiService.tsx` 의 스트리밍 진행 중(`isStreaming === true`) 입력 폼 영역에 "중단" 버튼을 노출하고, 클릭 시 스트리밍을 즉시 중단하고 입력창을 재활성화한다.
- [ ] 중단 시점까지 수신된 스트리밍 답변 텍스트를 채팅 로그에 보존하고 중단 안내(예: `(생성이 중단되었습니다)`)를 반영한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` 의 스트리밍 루프에서 Redis 취소 플래그(`job:{jobId}:cancelled`)를 주기적으로 확인하여 취소 시 안전하게 조기 종료하고 `publish_done` 을 발행한다.
- [ ] 프론트엔드 및 백엔드 변경 사항에 대한 단위/컴포넌트 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 답변 내부의 개별 코드 블록 단위 복사 버튼 구현
- HTML Rich Text 형식 복사 (순수 마크다운 텍스트 복사만 지원)
- 중단된 시점부터 이어서 생성하는 이어받기 기능
- 게이트웨이(`public-server`)의 신규 인증 가드 수정 (기존 DELETE 엔드포인트 활용)

## 백엔드
`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
- 스트림 토큰 루프(`async for chunk in stream:`) 내에서 Redis의 `job:{job_id}:cancelled` 키를 확인하는 로직을 추가한다 (`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:143-157`).
- 취소 감지 시 루프를 조기 탈출하고 `publish_done(message.job_id)` 를 호출하여 스트림을 정상 종료한다.

## 프론트엔드
`public-front/src/api/ai.ts`
- `askQuestionStream` 함수에서 작업 취소 함수(`cancelStream: () => Promise<void>`)를 반환하거나 `AbortController` 를 통한 취소 요청(`DELETE /ai/jobs/:jobId`) 기능을 추가한다 (`public-front/src/api/ai.ts:195-235`).

`public-front/src/components/AiService.tsx`
- AI 메시지 버블 내부에 복사 버튼 렌더링 및 복사 상태(`copiedIndex: number | null`) 관리 로직을 추가한다 (`public-front/src/components/AiService.tsx:490-520`).
- 스트리밍 중단 핸들러(`handleCancelStreaming`)를 구현하고, 스트리밍 상태일 때 전송 버튼 대신 "중단" 버튼을 렌더링한다 (`public-front/src/components/AiService.tsx:596-613`).

`public-front/src/components/AiService.css`
- 메시지 복사 버튼 스타일 및 스트리밍 중단 버튼 스타일을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 완료된 AI 답변 메시지가 채팅창에 표시된 경우
  When 사용자가 "복사" 버튼을 클릭하면
  Then 클립보드에 해당 답변의 마크다운 텍스트가 복사되고 2초간 버튼이 "복사됨" 으로 변경된 후 다시 "복사" 로 복귀한다.
- Given `navigator.clipboard.writeText` 호출이 실패한 경우
  When 사용자가 복사 버튼을 클릭하면
  Then 버튼 상태는 유지되고 에러 배너에 "답변 복사에 실패했습니다." 가 표시된다.
- Given AI 답변이 스트리밍되고 있는 상태에서
  When 사용자가 "중단" 버튼을 클릭하면
  Then 즉시 스트림 구독이 종료되고 입력창이 활성화되며, 현재까지 수신된 답변 텍스트가 채팅창에 보존된다.
- Given 백엔드에서 AI 질의 작업이 실행 중인 상태에서 취소 요청이 접수된 경우
  When 컨슈머가 Redis 취소 플래그를 감지하면
  Then 추가 토큰 생성을 중단하고 즉시 완료 이벤트를 발행하여 종료한다.
- Given 빈 문자열이거나 유효하지 않은 메시지 항목(경계 케이스)
  When 복사 버튼을 클릭하거나 중단 요청을 시도하면
  Then 애플리케이션 오류 없이 안전하게 예외 처리된다.

## 참고
- 복사 버튼 추가 위치: `public-front/src/components/AiService.tsx:490-520`
- 중단 버튼 추가 위치: `public-front/src/components/AiService.tsx:596-613`
- 스트림 클라이언트 함수: `public-front/src/api/ai.ts:195-235`
- 백엔드 취소 엔드포인트: `public-server/apps/gateway/src/ai/job/job.controller.ts:60-84`
- 백엔드 Redis 취소 처리: `public-server/apps/gateway/src/ai/job/job-store.service.ts:51-57`
- Python 컨슈머 토큰 루프: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:143-157`
