---
id: SPEC-098
title: 실시간 채팅 이전 대화 페이징 소켓 이벤트 연동 및 이전 대화 더보기 UI 구현
status: ready
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 실시간 채팅 서비스는 SPEC-094를 통해 백엔드 `chat-service`의 gRPC 및 도메인 계층에 `before` 커서 기반 과거 메시지 조회 기능(`public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:42-59` (`ChatGrpcController.getMessages`))이 구현되었다. 그러나 게이트웨이와 프론트엔드 간 웹소켓 통신 계층에 이를 호출할 이벤트 규격이 누락되어 있어 사용자가 실제 이전 대화를 전혀 불러올 수 없는 기능 단절 결함이 남아 있다.
첫째, 게이트웨이 `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:96-126` (`handleJoinRoom`)를 보면, 클라이언트가 방에 입장할 때 `this.chatService.getMessages({ roomId, limit })`만 1회 호출할 뿐, 특정 시점 이전의 대화 이력을 추가로 요청할 수 있는 웹소켓 이벤트(`fetch_history`)가 부재하다.
둘째, 프론트엔드 `public-front/src/hooks/useChatSocket.ts:46-64` (`useChatSocket`)에서도 소켓 연결 시 `join_room` 응답으로 전달받은 최근 20개의 메시지만 `setMessages(history)`로 저장한 뒤, 이전 대화를 추가로 요청하고 기존 목록에 병합할 수 있는 인터페이스와 상태(`hasMore`, `loadingMore`)가 구현되어 있지 않다.
셋째, 프론트엔드 `public-front/src/components/ChatRoom.tsx:72-95` (`ChatRoom`) 화면에서는 최초 수신된 20개 메시지만 고정 표시될 뿐, 과거 대화를 조회할 수 있는 UI 컴포넌트(더보기 버튼 등)나 사용자 피드백이 전무하여 사용자는 20건 이전의 대화 내용을 영구히 확인할 수 없다.
제품 방향상 채팅은 유지 대상이나 명백한 결함은 반드시 바로잡아야 하며, 이미 배송된 작업(SPEC-094)의 성과를 온전히 살리기 위해서도 게이트웨이에 `fetch_history` 소켓 이벤트를 신설하고 프론트엔드에 이전 대화 더보기 UI를 연동하여 이전 대화 페이징 흐름을 정상화해야 한다.
## 요구사항
- [ ] 게이트웨이 `ChatGateway`에 이전 대화 페이징 조회를 전담하는 `@SubscribeMessage('fetch_history')` 소켓 이벤트 핸들러(`handleFetchHistory`)를 추가한다.
- [ ] `fetch_history` 요청 페이로드(`{ roomId: string; before: number; limit?: number }`)를 검증하여 `roomId` 및 `before`가 유효하지 않거나 방에 입장하지 않은 소켓 클라이언트의 요청은 거부한다.
- [ ] 게이트웨이에서 gRPC `chatService.getMessages({ roomId, limit, before })`를 호출하여 이전 대화 목록을 조회하고, 클라이언트 화면 표시를 위해 시간순(오래된 순)으로 정렬하여 반환한다.
- [ ] 게이트웨이 응답 본문에 조회된 메시지 목록(`messages`)과 추가 과거 메시지 존재 여부(`hasMore`: 조회된 메시지 수가 요청 `limit` 이상인지 여부)를 포함하여 클라이언트에 전달한다.
- [ ] 프론트엔드 `useChatSocket` 훅에 이전 대화 페이징 상태(`hasMore: boolean`, `loadingMore: boolean`)와 조회 트리거 함수(`loadMoreMessages: () => Promise<void>`)를 추가한다.
- [ ] `useChatSocket`의 `loadMoreMessages` 실행 시 현재 대화 목록의 가장 오래된 메시지 타임스탬프(`messages[0].timestamp`)를 `before` 커서로 사용하여 소켓 `fetch_history` 이벤트를 호출한다.
- [ ] 수신된 과거 메시지들을 기존 메시지 목록의 앞부분에 중복 없이 안전하게 prepend 병합하고, `hasMore` 상태를 갱신한다.
- [ ] 프론트엔드 `ChatRoom.tsx`의 메시지 목록 상단에 `hasMore`가 `true`일 때 [이전 대화 더보기] 버튼(`btn-load-more`)을 렌더링한다.
- [ ] 이전 대화 로딩 중(`loadingMore === true`)일 때는 버튼 텍스트를 "불러오는 중..."으로 변경하고 비활성화(`disabled`)하여 중복 요청을 방지한다.
- [ ] 이전 대화가 더 이상 없거나(`hasMore === false`), 대화 메시지가 0건인 경우에는 [이전 대화 더보기] 버튼을 렌더링하지 않는다.
- [ ] 변경된 게이트웨이 `fetch_history` 핸들러, `useChatSocket` 훅의 과거 메시지 병합, `ChatRoom`의 더보기 버튼 렌더링 및 클릭 인터랙션을 검증하는 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 무한 스크롤(상단 스크롤 도달 시 자동 로딩) 등의 복잡한 DOM 스크롤 옵저버는 도입하지 않고, 직관적인 [이전 대화 더보기] 버튼 클릭 방식으로 단순화하여 안정성을 확보한다.
- 1:1 귓속말, 비공개 채팅방 개설, 메시지 검색 등 신규 채팅 도메인 기능은 추가하지 않는다.
- `chat-service`의 메시지 저장소(Redis ZSET) 구조나 gRPC proto 계약(`chat.proto`) 변경은 포함하지 않는다 (SPEC-094에서 이미 `before` 필드가 정의됨).
- FlatBuffers 직렬화 규격 변경은 포함하지 않는다.
## 백엔드
`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts`
- `fetch_history` 소켓 이벤트 핸들러 추가:
  - `@SubscribeMessage('fetch_history')`
  - `async handleFetchHistory(@ConnectedSocket() client: Socket, @MessageBody() data: { roomId: string; before: number; limit?: number })`
  - 요청자 검증: `roomId`와 `before` 유효성 확인 및 `client.rooms.has(roomId)` 검사
  - gRPC `this.chatService.getMessages({ roomId, limit: data.limit ?? DEFAULT_HISTORY_LIMIT, before: data.before })` 호출
  - 반환 규격: `{ success: true, messages: HistoryMessage[], hasMore: boolean }`
`public-server/apps/gateway/src/chat/chat-gateway.gateway.spec.ts`
- `handleFetchHistory`에 대한 단위 테스트 추가:
  - 유효하지 않은 `roomId` 또는 방 미참여 시 실패 응답 검증
  - `before` 타임스탬프와 `limit`을 gRPC 서비스에 올바르게 전달하는지 검증
  - 조회된 메시지 역순 정렬 및 `hasMore` 플래그 계산 검증
## 프론트엔드
`public-front/src/hooks/useChatSocket.ts`
- `FetchHistoryAck` 인터페이스 선언 (`{ success: boolean; error?: string; messages?: HistoryMessage[]; hasMore?: boolean }`)
- 상태 추가:
  - `hasMore: boolean` (초기값 `false`, `join_room` 응답 수신 시 메시지 수가 `limit` 이상이면 `true`로 설정)
  - `loadingMore: boolean` (초기값 `false`)
- `loadMoreMessages` 함수 구현:
  - `loadingMore`이거나 `!hasMore`, 또는 `messages.length === 0`일 때 조기 반환
  - `messages[0].timestamp`를 `before` 값으로 담아 `fetch_history` emit
  - 응답 수신 후 과거 메시지들을 기존 목록 앞에 prepend 병합 (메시지 ID 중복 제거)
  - `hasMore` 상태 갱신 및 로딩 상태 해제
- 훅 반환 객체에 `hasMore`, `loadingMore`, `loadMoreMessages` 추가
`public-front/src/components/ChatRoom.tsx`
- `chat-messages-container` 최상단에 `hasMore`가 `true`일 때 [이전 대화 더보기] 버튼 렌더링
  - `<button type="button" className="btn-load-more" onClick={loadMoreMessages} disabled={loadingMore}>`
  - 버튼 텍스트: `loadingMore ? '불러오는 중...' : '이전 대화 더보기'`
`public-front/src/components/ChatRoom.css`
- `.btn-load-more`: 상단 중앙 배치, 여백, 폰트 스타일, hover 스타일 및 disabled 비활성화 스타일 추가
`public-front/src/hooks/useChatSocket.test.ts` 및 `public-front/src/components/ChatRoom.test.tsx`
- `fetch_history` 호출 및 과거 메시지 prepend 병합 검증 단위 테스트
- [이전 대화 더보기] 버튼 렌더링, 클릭 시 `loadMoreMessages` 호출 및 로딩 비활성화 검증 단위 테스트
- 더 이상 과거 메시지가 없을 때(`hasMore === false`) 버튼 숨김 검증 단위 테스트 (경계 케이스)
## 수용 기준 (Acceptance Criteria)
- Given 사용자가 채팅방에 입장하여 초기 20개의 메시지가 표시되어 있고 추가 과거 메시지가 존재하는 상황(`hasMore === true`)에서 When 채팅 메시지 목록 상단을 확인하면 Then [이전 대화 더보기] 버튼이 정상적으로 렌더링된다.
- Given [이전 대화 더보기] 버튼이 표시된 상태에서 When 사용자가 버튼을 클릭하면 Then 현재 가장 오래된 메시지의 타임스탬프를 `before` 커서로 담아 게이트웨이 `fetch_history` 소켓 이벤트가 호출되고, 로딩 중에는 버튼 텍스트가 "불러오는 중..."으로 변경되며 비활성화(`disabled`)된다.
- Given 게이트웨이가 과거 메시지 10건과 `hasMore: false`를 응답한 상황에서 When 프론트엔드가 응답을 수신하면 Then 불러온 10건의 과거 메시지가 기존 대화 목록 상단에 시간순으로 prepend 추가되고, [이전 대화 더보기] 버튼은 화면에서 사라진다.
- Given 방에 총 메시지가 5건밖에 없어 최초 방 입장 시 모든 메시지가 로드된 상황(경계 케이스)에서 When 채팅 화면을 확인하면 Then `hasMore`가 `false`로 설정되어 [이전 대화 더보기] 버튼이 노출되지 않는다.
- Given 방에 참여하지 않은 소켓 클라이언트가 `fetch_history` 이벤트를 호출한 상황(경계 케이스)에서 When 게이트웨이가 요청을 수신하면 Then `join_room required` 에러와 함께 `success: false`를 반환한다.
## 참고
- 고쳐야 할 자리 (게이트웨이 소켓 핸들러): `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:96-126` (`ChatGateway.handleJoinRoom`)
- 고쳐야 할 자리 (채팅 소켓 훅): `public-front/src/hooks/useChatSocket.ts:46-64` (`useChatSocket`)
- 고쳐야 할 자리 (채팅 화면 컴포넌트): `public-front/src/components/ChatRoom.tsx:72-95` (`ChatRoom`)
- 백엔드 gRPC 과거 메시지 조회 구현부: `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:42-59` (`ChatGrpcController.getMessages`)
- Proto 계약 정의: `public-server/libs/rpc/proto/chat.proto:21-25` (`GetMessagesRequest`)
