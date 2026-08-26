---
id: SPEC-003
title: 일시 표시 형식 통일
status: done
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 시스템은 날짜 표기를 위해 `formatDate` 유틸리티를 적용하여 `YYYY-MM-DD` 형식으로 통일하고 있으나, 시/분/초를 포함한 일시(DateTime) 표기는 각기 다른 방식을 사용하고 있다. 예를 들어, 프론트엔드의 채팅 화면([ChatRoom.tsx](file:///Users/seungwonkang/Documents/work/public-project/public-front/src/components/ChatRoom.tsx#L78-L81))에서는 `toLocaleTimeString`을 사용하고 있어 로케일에 따라 표시 형식이 달라지고 일자 정보가 생략된다. 또한 서버 로그나 DB 데이터 디버깅 시 일시 정보의 형식이 일관되지 않아 정확한 시간 비교에 어려움이 있다. 따라서 일관된 포맷(`YYYY-MM-DD HH:mm:ss`)으로 안전하게 일시 정보를 변환하는 `formatDateTime` 함수가 필요하다.
## 요구사항
- [ ] `formatDateTime(value)` 함수를 만들어 내보낸다.
- [ ] 유효한 입력값에 대해 `YYYY-MM-DD HH:mm:ss` 형식의 문자열을 반환한다 (예: `2026-08-19 14:30:05`).
- [ ] 연, 월, 일, 시, 분, 초는 항상 두 자리로 채운다 (예: `2026-01-05 09:05:00`).
- [ ] `Date` 객체와 ISO 8601 문자열을 모두 입력으로 받을 수 있다.
- [ ] 입력값의 로컬 시간대를 기준으로 포맷팅하며 로케일 설정에 결과가 영향을 받지 않아야 한다.
- [ ] 유효하지 않은 입력은 빈 문자열 `""` 을 반환하며 예외를 던지지 않는다.
- [ ] 해당 규칙을 검증하는 테스트 코드를 작성한다.
## 비요구사항 (Out of scope)
- 시간대(Timezone) 변환 - 넘어온 값의 시간대를 그대로 사용한다.
- 상대 시간 표기 ("방금 전", "3분 전" 등)
- 기존 화면의 시각 표시부를 찾아 일괄 수정하는 작업 (유틸리티 함수와 테스트만 구현)
- 외부 날짜/시간 라이브러리 도입 (JavaScript 기본 내장 API를 활용하여 직접 구현)
## 백엔드
`public-server` 의 `libs/common/src/utils/date.util.ts` 에 `formatDateTime` 함수를 구현하여 내보낸다.
테스트는 `libs/common/src/utils/date.util.spec.ts` 에 작성한다.
## 프론트엔드
`public-front` 의 `src/utils/date.ts` 에 `formatDateTime` 함수를 구현하여 내보낸다.
테스트는 `src/utils/date.test.ts` 에 vitest 로 작성한다.
## 수용 기준 (Acceptance Criteria)
- Given `new Date(2026, 7, 19, 14, 30, 5)` When `formatDateTime` 을 호출하면 Then `"2026-08-19 14:30:05"` 를 반환한다.
- Given ISO 8601 문자열 `"2026-08-19T14:30:05"` When `formatDateTime` 을 호출하면 Then `"2026-08-19 14:30:05"` 를 반환한다.
- Given `new Date("Invalid")` (유효하지 않은 날짜 객체) When `formatDateTime` 을 호출하면 Then `""` 을 반환한다.
- Given `null` 또는 `undefined` When `formatDateTime` 을 호출하면 Then `""` 을 반환하고 예외를 던지지 않는다.
- Given 빈 문자열 `""` When `formatDateTime` 을 호출하면 Then `""` 을 반환한다.
