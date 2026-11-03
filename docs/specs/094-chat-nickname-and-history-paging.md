---
id: SPEC-094
title: 실시간 채팅 발신자 닉네임 표시 및 이전 대화 조회 커서 연동
status: ready
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 실시간 채팅 서비스는 게이트웨이와 백엔드 gRPC 통신, 소켓 및 프론트엔드 렌더링 간 계약 및 매핑 결함으로 인해 사용자가 원활하게 소통할 수 없는 두 가지 명백한 기능 결함을 안고 있다.
첫째, `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:247-255` (`handleSendMessage`)를 보면 웹소켓 인증 세션(`user.nickName`)이 존재함에도 불구하고 gRPC `saveMessage` 호출 시 `senderUuid: user.uuid`만 전달하고 발신자 닉네임을 누락하고 있다. gRPC 계약 정의인 `public-server/libs/rpc/proto/chat.proto:9-13` (`SaveMessageRequest`) 및 `public-server/libs/rpc/proto/chat.proto:25-30` (`ChatMessage`)에도 닉네임 필드가 정의되어 있지 않다.
둘째, `public-server/apps/chat-service/src/message/rpc/chat.grpc-mapper.ts:13-22` (`ChatGrpcMapper.toMessageBuffer`)에서 FlatBuffers `buildChatMessage` 호출 시 메타데이터 필드를 빈 문자열(`''`)로 고정하여 직렬화하고, `public-front/src/components/ChatRoom.tsx:84-86` (`ChatRoom`)에서는 발신자 정보로 `msg.senderId`를 직접 렌더링한다. 이로 인해 채팅방 화면에 사용자의 실제 닉네임 대신 36자리 UUID 문자열(예: `e4f1a234-5678-4321-abcd-ef0123456789`)이 그대로 노출되어 대화 상대를 식별할 수 없다.
셋째, `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:47-51` (`ChatGrpcController.getMessages`)에서는 이전 대화 목록 조회 시 요청된 커서(`request.before`)를 참조하지 않고 `Date.now() * 1000`으로 현재 시각을 하드코딩하여 조회하고 있다. `public-server/libs/rpc/proto/chat.proto:20-23` (`GetMessagesRequest`)에도 페이지네이션 커서 필드가 없어, 클라이언트(`public-front/src/hooks/useChatSocket.ts:51-62`)나 게이트웨이가 과거 대화를 더 불러오고 싶어도 항상 최신 메시지만 반환되어 이전 대화 조회가 원천적으로 불가능하다.
제품 가이드상 채팅은 유지 대상이나 명백한 결함은 반드시 바로잡아야 하므로, 발신자 닉네임의 gRPC·FlatBuffers 전달 및 화면 렌더링을 보장하고, gRPC 대화 조회 시 `before` 커서 지원을 연동하여 과거 대화 페이징 결함을 해결해야 한다.

## 요구사항
- [ ] `public-server/libs/rpc/proto/chat.proto`의 `SaveMessageRequest` 및 `ChatMessage` 메시지에 `sender_nickname` (tag 4, 5) 필드를 추가하고 proto 정의를 컴파일한다.
- [ ] `public-server/libs/rpc/proto/chat.proto`의 `GetMessagesRequest`에 `before` (int64, tag 3) 선택적 타임스탬프 필드를 추가하고 proto 정의를 컴파일한다.
- [ ] 게이트웨이 `handleSendMessage`에서 gRPC `saveMessage` 호출 시 인증된 세션의 사용자 닉네임(`user.nickName`)을 `senderNickname`으로 함께 전달한다.
- [ ] chat-service의 `ChatGrpcMapper.toMessageBuffer`에서 전달받은 `senderNickname`을 FlatBuffers `metadata` 필드(또는 전용 필드)에 직렬화하여 저장한다.
- [ ] chat-service의 `ChatGrpcMapper.toGetMessagesResponse`에서 FlatBuffers의 닉네임 메타데이터를 gRPC `ChatMessage.senderNickname`으로 복원하여 반환한다.
- [ ] chat-service의 `ChatGrpcController.getMessages`에서 `request.before`가 주어지면 해당 타임스탬프를 사용하고, 주어지지 않거나 0 이하인 경우에만 `Date.now() * 1000`을 기본값으로 사용하도록 개선한다.
- [ ] 게이트웨이 `handleJoinRoom`의 `HistoryMessage` 인터페이스 및 응답에 `senderNickname` 필드를 포함하고 소켓 클라이언트로 전달한다.
- [ ] 프론트엔드 `useChatSocket` 훅에서 실시간 수신 메시지(`new_messages`) 및 초기 이력(`join_room` 응답) 매핑 시 닉네임 정보를 보존한다.
- [ ] 프론트엔드 `ChatRoom.tsx` 메시지 행에서 닉네임이 존재하는 경우 UUID 대신 실제 닉네임을 우선 표시하고, 닉네임이 없는 레거시 메시지의 경우에만 기존 `senderId`로 안전하게 폴백한다.
- [ ] 변경 사항에 대해 게이트웨이 소켓 핸들러, chat-service gRPC 매퍼 및 컨트롤러, 프론트엔드 채팅 컴포넌트 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 1:1 귓속말, 비공개 채팅방 개설, 사용자 차단 등 신규 채팅 도메인 기능은 추가하지 않는다.
- FlatBuffers 스키마(`chat.fbs`)의 호환성을 깨는 대규모 바이너리 레이아웃 변경은 수행하지 않으며 기존 `metadata` 문자열 필드를 호환 매개체로 활용한다.
- 채팅 무한 스크롤 UI 인터랙션(상단 스크롤 도달 시 자동 로딩) 전면 개편은 다루지 않으며, 게이트웨이 및 gRPC 계층의 `before` 커서 지원과 기본 페이징 계약 정상화에 집중한다.
- 사용자 프로필 이미지 등록 및 아바타 렌더링은 다루지 않는다.

## 백엔드
`public-server/libs/rpc/proto/chat.proto`
- `SaveMessageRequest`: `string sender_nickname = 4;` 필드 추가
- `GetMessagesRequest`: `int64 before = 3;` 필드 추가
- `ChatMessage`: `string sender_nickname = 5;` 필드 추가

`public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts`
- `getMessages` 메서드에서 `request.before` 값 검증: `request.before && Number(request.before) > 0 ? Number(request.before) : Date.now() * 1000` 로직 적용

`public-server/apps/chat-service/src/message/rpc/chat.grpc-mapper.ts`
- `toMessageBuffer`에서 `request.senderNickname`을 FlatBuffers `metadata` 인자로 전달
- `toGetMessagesResponse`에서 FlatBuffers `chatMsg.metadata()`를 `senderNickname`으로 매핑

`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts`
- `handleSendMessage`: `this.chatService.saveMessage` 인자에 `senderNickname: user.nickName` 전달
- `handleJoinRoom`: `HistoryMessage`에 `senderNickname: string` 추가 및 `reply.messages`의 `senderNickname` 전달

## 프론트엔드
`public-front/src/hooks/useChatSocket.ts`
- `HistoryMessage` 인터페이스에 `senderNickname?: string` 추가
- `join_room` 수신 메시지 매핑 시 `metadata: m.senderNickname || ''` 바인딩
- 실시간 수신 메시지(`new_messages`) 처리 시 디코딩된 `msg.metadata`를 유지

`public-front/src/components/ChatRoom.tsx`
- 발신자 라벨 렌더링부(`msg-sender-lbl`): `msg.metadata || msg.senderId` 형태로 닉네임 우선 노출

`public-front/src/components/ChatRoom.test.tsx` (또는 단위 테스트)
- 메시지에 닉네임(`metadata`)이 있을 때 닉네임이 정상 노출되는지 검증
- 닉네임이 없을 때 기존 `senderId`(UUID)로 폴백 렌더링되는지 검증

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 로그인하여 닉네임 "플레이어A"를 보유한 상태에서 When 채팅방에 새 메시지를 전송하면 Then 게이트웨이와 gRPC를 거쳐 FlatBuffers에 닉네임이 저장되고 채팅방 메시지 발신자 영역에 UUID 대신 "플레이어A"가 표시된다.
- Given 닉네임 메타데이터가 존재하지 않는 과거 레거시 메시지 버퍼가 조회된 상황(경계 케이스)에서 When 채팅 화면에 해당 메시지가 렌더링되면 Then 에러 없이 기존 `senderId` 문자열이 폴백으로 안전하게 표시된다.
- Given 특정 타임스탬프(`before = 1700000000000000`)를 인자로 `getMessages` gRPC RPC를 호출한 상황에서 When chat-service 컨트롤러가 요청을 처리하면 Then 현재 시각(`Date.now() * 1000`)이 아닌 전달받은 `before` 타임스탬프를 기준으로 이전 메시지 목록이 정상 조회된다.
- Given `request.before` 필드가 0이거나 생략된 상태로 `getMessages` RPC를 호출한 상황(경계 케이스)에서 When 컨트롤러가 동작하면 Then 기존과 동일하게 현재 시각(`Date.now() * 1000`)을 기본 상한선으로 삼아 최신 메시지 목록을 반환한다.
- Given 다른 사용자가 방에 입장하여 `join_room` 요청을 보냈을 때 When 이전 대화 이력 목록(`messages`)을 응답받으면 Then 각 메시지 객체에 `senderNickname`이 포함되어 있으며 이전 작성자의 닉네임이 화면에 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:247-255` (`handleSendMessage`)
- 고쳐야 할 자리: `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:47-51` (`ChatGrpcController.getMessages`)
- 고쳐야 할 자리: `public-server/apps/chat-service/src/message/rpc/chat.grpc-mapper.ts:13-22` (`ChatGrpcMapper.toMessageBuffer`)
- 고쳐야 할 자리: `public-front/src/components/ChatRoom.tsx:84-86` (`ChatRoom`)
- 소켓 대화 이력 수신부: `public-front/src/hooks/useChatSocket.ts:51-62` (`useChatSocket`)
- Proto 정의: `public-server/libs/rpc/proto/chat.proto:9-13` (`SaveMessageRequest`), `public-server/libs/rpc/proto/chat.proto:20-23` (`GetMessagesRequest`)
