---
id: SPEC-030
title: 대화 세션 조회·삭제의 소유권 검증 누락 수정 (IDOR)
status: done
targets: [python-server, server]
stages: [backend, qa]
priority: high
---

## 출처
SPEC-024 구현 중 발견해 그 문서의 "함께 발견한 것" 에 적어 둔 결함이다. 명세서 항목이 아니라 코드에서 나온 것이다.

## 배경 / 문제
`GET /rag/sessions/{session_id}` 와 `DELETE /rag/sessions/{session_id}` 가 요청한 사람이 세션의 주인인지 확인하지 않았다. 세션 id 하나만 알면 남의 대화 전문을 읽고 지울 수 있었다.

- 파이썬 라우터가 `session_id` 만 받고 `userId` 를 받지 않았다 — 소유자를 가릴 정보 자체가 서비스에 닿지 않았다.
- 게이트웨이 프록시(`rag-session-proxy.controller.ts`)도 목록 조회에만 `userId` 를 채우고 상세·삭제에는 붙이지 않았다.
- `SessionService.get_session`/`delete_session` 은 받은 id 를 그대로 리포지토리에 넘겼다.

SPEC-021 이 결제 조회에서 고친 것과 같은 종류다. PRODUCT.md 의 판단 기준 3번("되돌릴 수 없는 동작에 안전장치가 없으면 그것을 먼저 고친다")에 걸린다 — 삭제는 되돌릴 수 없다.

## 요구사항
- [x] 세션 상세 조회는 소유자만 읽을 수 있다.
- [x] 세션 삭제는 소유자만 지울 수 있다.
- [x] 없는 세션과 남의 세션을 구분할 수 없어야 한다.
- [x] `userId` 는 클라이언트가 아니라 게이트웨이가 인증 세션에서 채운다.
- [x] 소유권 검증에 대한 테스트를 남긴다.

## 비요구사항 (Out of scope)
- 세션 공유·위임 기능. 지금은 소유자 단독 접근만 있으면 된다.
- 목록 조회(`GET /rag/sessions`) — 이미 `userId` 로 걸러지고 있었다.
- 인증 토큰 발급·검증 자체(`libs/auth`).

## 수용 기준 (Acceptance Criteria)
- Given 사용자 A 의 세션이 있고 사용자 B 가 그 세션 id 를 알 때
  When B 가 `GET /rag/sessions/{id}` 를 호출하면
  Then 404 가 돌아오고 대화 내용이 노출되지 않는다.
- Given 사용자 A 의 세션이 있고 사용자 B 가 그 세션 id 를 알 때
  When B 가 `DELETE /rag/sessions/{id}` 를 호출하면
  Then 404 가 돌아오고 세션은 그대로 남는다.
- Given 존재하지 않는 세션 id 와 남의 세션 id 두 가지로 조회할 때 (경계 케이스)
  When 각각 호출하면
  Then 상태 코드와 응답 본문이 같아 어느 쪽이 실재하는지 구분할 수 없다.
- Given 소유자가 자기 세션을 열 때
  When `GET /rag/sessions/{id}` 를 호출하면
  Then 200 과 함께 턴 목록이 그대로 돌아온다.

## 구현 기록

### 소유권 조건을 어디에 둘 것인가
서비스에서 세션을 읽어 온 뒤 `user_id` 를 비교하는 방법과, 조건을 Mongo 쿼리에 함께 거는 방법이 있었다. **후자를 택했다.** 전자는 세션을 읽는 호출부가 늘 때마다 비교를 빠뜨릴 수 있고, 실제로 이번 결함이 그렇게 생겼다. 조건이 쿼리에 있으면 빠뜨릴 자리가 없다.

### 백엔드 (python-server)
- `ConversationSessionRepository` 에 `find_by_id_for_user` / `delete_by_id_for_user` 신설. 둘 다 `{"sessionId": ..., "userId": ...}` 로 건다. 삭제는 `deleted_count > 0` 을 돌려줘 호출부가 "지운 것이 없음" 을 알 수 있게 했다.
- 기존 `find_by_id` / `delete_by_id` 는 남겼다. Kafka 소비자가 자기가 만든 잡의 세션을 이어 쓸 때처럼 요청자가 없는 자리에서 필요하다. **HTTP 경로에서 쓰지 않는다**는 것을 각 메서드 독스트링에 못 박았다.
- `SessionService.get_session`/`delete_session` 이 `user_id` 를 받고, 대상이 없으면 `SessionNotFoundError` 를 던진다.
- 라우터가 `userId` 를 필수 쿼리로 받고 위 오류를 404 로 옮긴다. **없는 세션과 남의 세션에 같은 본문의 404** 를 준다 — 나누면 세션 id 로 존재 여부를 알아낼 수 있다.
- `GET` 응답 계약이 바뀌었다. 전에는 없는 세션에 `200 null` 이었고 지금은 `404` 다. 프론트는 이미 `try/catch` 로 감싸 "세션을 불러오는 데 실패했습니다" 를 띄우므로 화면 동작은 그대로다.
- 삭제는 지운 것이 없으면 404 다. `204` 로 성공한 척하면 호출부가 지워졌다고 믿는다.
- `FeedbackService` 도 새 리포지토리 메서드를 쓰도록 바꿨다. 어제 SPEC-024 에서 서비스 계층 비교로 짰던 것을 같은 규칙으로 모았다.
- `GetSessionUseCase` / `DeleteSessionUseCase` 는 composition 에 배선만 돼 있고 호출되지 않는 죽은 코드지만, 소유권 조건 없이 두면 다음에 배선하는 사람이 그대로 구멍을 연다. 같은 시그니처로 맞췄다.
- 테스트 11건 추가(리포지토리 쿼리 조건 4, 라우트 E2E 7). 130개 통과, ruff·mypy clean.

### 게이트웨이 (NestJS)
- `RagSessionProxyController` 의 상세·삭제가 `req.session.uuid` 를 `userId` 로 실어 보낸다. 클라이언트가 쿼리로 `userId` 를 보내도 쓰지 않는다 — 테스트로 고정했다.
- 테스트 4건 추가. gateway 66개 통과, 빌드 통과.

### 프론트엔드
변경 없다. 게이트웨이가 `userId` 를 채우므로 호출 형태가 그대로다. 175개 통과.

### 배포 시 주의
`GET /rag/sessions/{id}` 가 `userId` 를 **필수**로 받는다. 게이트웨이를 먼저 올리고 파이썬을 올리면 그 사이 요청이 422 로 떨어진다. 파이썬 → 게이트웨이 순으로 배포한다.
