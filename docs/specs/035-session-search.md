---
id: SPEC-035
title: 대화 세션 목록 키워드 검색 지원 및 사이드바 검색창 연동
status: ready
targets: [python-server, server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
사용자가 이전 RAG 질의응답 세션을 다시 찾아 확인하려 할 때, 저장된 세션이 많아지면 원하는 세션을 빠르게 찾을 수 없다.

1. `public-python-server/src/ai_service/rag/repository.py:50-57` (`find_by_user_id`)에서 `userId` 조건으로만 쿼리(`find({"userId": user_id})`)하여 페이징 목록을 반환하고 있어, 대화 제목(`title`)에 대한 필터링 기능이 없다.
2. `public-python-server/src/ai_service/rag/router.py:16-24` (`get_sessions`) 및 `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:28-39` (`getSessions`)에 검색 키워드(`keyword` 또는 `q`) 쿼리 파라미터가 지원되지 않는다.
3. `public-front/src/components/AiService.tsx:832-867` (`chat-session-sidebar`)의 대화 이력 사이드바 UI에 검색창이 없어, 사용자는 스크롤을 통해 일일이 눈으로 이전 대화 제목을 찾아야 한다.
4. `public-front/src/api/ai.ts:297-300` (`getSessions`) 클라이언트 API 함수 또한 `keyword` 파라미터를 지원하지 않아 백엔드 검색 연동이 불가능하다.

이로 인해 이전 질문에서 확인했던 지식 근거나 답변 신뢰도 정보를 다시 열람하려 할 때 탐색 비용이 증가하여 답변을 검증하고 재확인하는 경험을 저해한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `find_by_user_id` 메서드에 `keyword: str | None = None` 선택적 매개변수를 추가한다.
- [ ] `keyword`가 전달된 경우, 소유자 격리(`userId: user_id`)를 유지하면서 `title` 필드에 대해 정규표현식 대소문자 무시 부분 일치(`{"$regex": re.escape(keyword), "$options": "i"}`) 검색을 수행한다.
- [ ] `public-python-server/src/ai_service/rag/service.py`의 `get_sessions` 및 `public-python-server/src/ai_service/rag/router.py`의 `get_sessions`에 `keyword: str | None = Query(default=None)` 쿼리 파라미터를 추가하여 전달한다.
- [ ] `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts`의 `getSessions`에서 `@Query('keyword') keyword?: string`을 받아 ai-service-py 호출 시 파라미터로 전달한다.
- [ ] `public-front/src/api/ai.ts`의 `getSessions(userId, page, limit, keyword)` 시그니처를 확장하여 `keyword` 파라미터를 게이트웨이로 전달한다.
- [ ] `public-front/src/components/AiService.tsx`의 대화 이력 사이드바(`chat-session-sidebar`) 상단에 세션 검색 입력창(placeholder: "대화 검색...")을 추가한다.
- [ ] 사용자가 검색어를 입력하면 `getSessions`를 호출하여 필터링된 대화 세션 목록을 실시간으로 갱신하여 렌더링한다.
- [ ] 검색어 입력값이 공백이거나 지워졌을 때는 전체 세션 목록을 조회하여 표시한다.
- [ ] 검색 결과가 없을 경우 "검색 결과가 없습니다" 안내 문구를 표시한다.
- [ ] 검색창 초기화(클리어) 버튼 또는 입력값 지우기 인터랙션을 제공한다.

## 비요구사항 (Out of scope)
- 대화 세션 내 개별 턴(메시지 본문)에 대한 전문(Full-text) 검색 엔진 구축 (Elasticsearch 도입이나 MongoDB `$text` 인덱스 생성은 하지 않으며 `title` 부분 일치 검색으로 한정).
- 답변 북마크(보관함) 신규 컬렉션 및 영속화 API 신설.
- 다른 사용자 간 대화 공유 및 권한 위임 기능.

## 백엔드
- `public-python-server/src/ai_service/rag/repository.py`:
  - `find_by_user_id(self, user_id: str, page: int, limit: int, keyword: str | None = None) -> list[ConversationSession]`
  - 쿼리 구성:
    ```python
    query: dict[str, Any] = {"userId": user_id}
    if keyword and keyword.strip():
        query["title"] = {"$regex": re.escape(keyword.strip()), "$options": "i"}
    ```
- `public-python-server/src/ai_service/rag/service.py`:
  - `get_sessions(self, user_id: str, page: int, limit: int, keyword: str | None = None)`
- `public-python-server/src/ai_service/rag/router.py`:
  - `get_sessions(service: SessionServiceDep, user_id: str = Query(alias="userId"), page: int = Query(default=DEFAULT_PAGE), limit: int = Query(default=DEFAULT_LIMIT), keyword: str | None = Query(default=None)) -> list[SessionOut]`
- `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts`:
  - `getSessions(@Req() req: AuthenticatedRequest, @Query('page') page?: string, @Query('limit') limit?: string, @Query('keyword') keyword?: string)`
  - ai-service-py 요청 파라미터에 `keyword` 전달.

## 프론트엔드
- `public-front/src/api/ai.ts`:
  - `getSessions(userId: string, page = 1, limit = 20, keyword?: string): Promise<SessionOut[]>`
- `public-front/src/components/AiService.tsx`:
  - `sessionKeyword` 상태 변수 추가.
  - 대화 이력 사이드바(`chat-session-sidebar`) 헤더 아래에 검색창 UI(`<input className="session-search-input" value={sessionKeyword} ... />`) 추가.
  - 검색어 변경 시 디바운스 또는 입력 핸들러를 통해 `getSessions(userId, 1, 20, keyword)` 호출.
  - 검색 결과가 비어있을 때 `sessionKeyword` 유무에 따라 "검색 결과가 없습니다" 또는 "저장된 대화가 없습니다" 메시지 분기 처리.
- `public-front/src/components/AiService.css`:
  - `.session-search-box`, `.session-search-input` 스타일 정의 (어두운 테마 및 포커스 스타일 적용).

## 수용 기준 (Acceptance Criteria)
- Given 사용자 A가 "매출 보고서 분석"과 "보안 가이드라인" 두 개의 대화 세션을 보유하고 있을 때
  When 사이드바 검색창에 "매출"을 입력하면
  Then 세션 목록에 "매출 보고서 분석" 세션만 노출되고 "보안 가이드라인"은 목록에서 제외된다.
- Given 대화 세션 목록이 검색어로 필터링된 상태에서
  When 검색창의 텍스트를 모두 지우거나 빈 공백을 입력하면 (경계 조건)
  Then 전체 세션 목록이 다시 로드되어 표시된다.
- Given 사용자 A가 보유한 세션 제목과 일치하지 않는 검색어(예: "존재하지않는제목")를 입력할 때 (경계 조건)
  When 검색 요청이 완료되면
  Then "검색 결과가 없습니다" 안내 텍스트가 표시되고 에러가 발생하지 않는다.
- Given 사용자 B가 "매출 보고서 분석"과 동일한 제목의 세션을 보유하고 있을 때
  When 사용자 A가 "매출"로 검색하면
  Then 사용자 B의 세션은 일체 포함되지 않고 오직 사용자 A 본인의 세션만 반환된다.

## 참고
- 리포지토리 쿼리 메서드: `public-python-server/src/ai_service/rag/repository.py:50-57` (`find_by_user_id`)
- 파이썬 세션 라우터: `public-python-server/src/ai_service/rag/router.py:16-24` (`get_sessions`)
- 게이트웨이 세션 프록시 컨트롤러: `public-server/apps/gateway/src/ai/proxy/rag-session-proxy.controller.ts:28-39` (`getSessions`)
- 프론트엔드 API 클라이언트: `public-front/src/api/ai.ts:297-300` (`getSessions`)
- 프론트엔드 사이드바 컴포넌트: `public-front/src/components/AiService.tsx:832-867` (`chat-session-sidebar`)
- 사이드바 CSS 스타일: `public-front/src/components/AiService.css:1114-1178` (`.chat-session-sidebar`)
