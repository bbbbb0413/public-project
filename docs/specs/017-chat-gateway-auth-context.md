---
id: SPEC-017
title: 채팅 게이트웨이 웹소켓 연결 시 인증 정보 주입 및 오류 처리 개선
status: done
targets: [server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:61-69` 의 `handleConnection` 에서는 클라이언트가 전달한 토큰(`client.handshake?.auth?.token || client.handshake?.query?.token`)의 단순 존재 여부만 확인하고 로깅 후 연결을 허용한다.
하지만 토큰을 검증하여 소켓 세션 데이터(`client.data.user`)에 주입하는 처리가 누락되어 있다.

이로 인해 `public-front/src/hooks/useChatSocket.ts:29-37` 에서 정상적으로 JWT 토큰을 담아 소켓 연결을 맺고 방에 입장하더라도, 사용자가 메시지를 전송(`handleSendMessage`)할 때 `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:214-217` 에서 `client.data.user` 가 `undefined` 로 판정되어 `{ success: false, error: 'Authentication failed' }` 에러가 반환되고 메시지 저장이 거부된다.

또한 `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:61-69` 에서 토큰이 누락된 비인증 연결 시도시 소켓을 즉시 끊지만 클라이언트 측에 에러 이유를 명시하지 않고, `handleSendMessage`(`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:190-245`)에서 gRPC 호출(`public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13-40`) 실패 시 발생하는 오류를 구분하여 처리할 수 없다.

## 요구사항
- [ ] `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts` 의 `handleConnection` 에서 전달받은 토큰을 `JwtService` 또는 인증 유틸을 통해 검증하고, 유효한 사용자 정보를 `client.data.user` 에 저장한다.
- [ ] 토큰이 유효하지 않거나 만료된 경우 소켓 연결을 거부하고 연결을 종료한다.
- [ ] `handleSendMessage` 호출 시 `client.data.user` 에 저장된 사용자 정보(`uuid`, `nickName`, `id`)를 기반으로 세션 메타데이터를 올바르게 구성하여 gRPC `saveMessage` 를 정상 호출한다.
- [ ] 인증 실패 및 유효하지 않은 페이로드에 대해 명확한 에러 응답을 반환한다.
- [ ] 게이트웨이 웹소켓 인증 및 메시지 전송 처리에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 토큰 발급, 갱신 및 서명 비밀키 생성/변경 (`libs/auth` 내부 핵심 로직 변경 금지)
- gRPC `ChatService` 컨트롤러 및 Redis 메시지 브로드캐스트 로직 변경
- 프론트엔드 채팅 UI/컴포넌트 구조 변경 및 신규 기능 추가

## 백엔드
`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts`
- `JwtService` 또는 인증 서비스 의존성을 주입받아 `handleConnection` 시점에 토큰을 파싱 및 검증한다.
- 토큰 검증 성공 시 `client.data.user` 에 **`handleSendMessage:219-224` 가 실제로 읽는 필드**(`uuid`, `nickName`, `id`)를 주입한다. HTTP 쪽 `GatewayAuthGuard:67-72` 가 쓰는 정규화와 같은 모양이어야 한다 — `{ uuid: payload.uuid ?? String(payload.id), nickName: payload.nickName ?? payload.name, id: payload.id ?? 0 }`.
- `{ id, name }` 형태로 주입하면 안 된다. `Session.create` 가 `user.uuid` / `user.nickName` / `user.id` 를 읽으므로 값이 `undefined` 로 채워진다.
- `handleSendMessage` 에서 정상적으로 주입된 `client.data.user` 를 활용해 `Session.create` 및 gRPC 메시지 저장을 수행한다.

## 수용 기준 (Acceptance Criteria)
- Given 유효한 사용자 JWT 토큰을 가진 클라이언트가 웹소켓(`/chat/ws`)에 연결할 때
  When 연결이 수립되면
  Then `client.data.user` 에 `uuid`, `nickName`, `id` 가 정상 주입되고, 그 값으로 만든 `Session` 의 `uuid`·`nickName` 이 토큰의 것과 일치한다.
- Given `client.data.user` 가 정상 주입된 소켓 클라이언트가 특정 방(`roomId: 'lobby'`)에 입장 후 메시지 전송 이벤트를 발행할 때
  When `send_message` 핸들러가 실행되면
  Then `Authentication failed` 오류 없이 gRPC `saveMessage` 가 호출되고 `{ success: true }` 응답을 반환한다.
- Given 토큰이 누락되었거나 만료/위조된 토큰으로 웹소켓 연결을 시도할 때 (경계 케이스)
  When 연결 요청이 들어오면
  Then `client.disconnect(true)` 가 호출되어 연결이 즉시 차단되고 `client.data.user` 가 설정되지 않는다.

## 참고
- 고쳐야 할 자리: `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:61-69` (`handleConnection`)
- 고쳐야 할 자리: `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:214-224` (`handleSendMessage` 의 `client.data.user` 검사와 `Session.create`)
- 그대로 따라야 할 본보기: `public-server/apps/gateway/src/auth/gateway-auth.guard.ts:64-80` (HTTP 쪽이 JWT 를 `Session` 으로 정규화하는 방식)
- 관련 정의 및 호출부: `public-front/src/hooks/useChatSocket.ts:29-37`
- 관련 gRPC 저장 구현: `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13-40`
