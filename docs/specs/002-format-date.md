---
id: SPEC-002
title: 날짜 표시 통일
status: done
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제

날짜를 화면에 찍는 코드가 곳곳에서 제각각이다. `toLocaleDateString()` 을 그대로
쓰면 실행 환경의 로케일 설정에 따라 `2026. 8. 19.` 이 되기도 하고 `8/19/2026` 이
되기도 한다. 서버와 브라우저의 결과가 어긋나면 같은 데이터가 화면과 로그에서
다르게 보인다.

표시용 날짜 문자열을 만드는 함수를 한 곳에 두고, 형식을 고정한다.

## 요구사항

- [ ] `formatDate(value)` 함수를 만들어 내보낸다
- [ ] 유효한 날짜는 `YYYY-MM-DD` 형식으로 반환한다 (예: `2026-08-19`)
- [ ] 월과 일은 항상 두 자리로 채운다 (`2026-01-05`, `2026-1-5` 가 아님)
- [ ] `Date` 객체와 ISO 8601 문자열을 모두 받는다
- [ ] **유효하지 않은 입력은 빈 문자열 `""` 을 반환한다.** 예외를 던지지 않는다
- [ ] 로케일 설정에 결과가 좌우되지 않는다
- [ ] 위 규칙을 검증하는 테스트를 함께 작성한다

## 비요구사항 (Out of scope)

- 시각(시:분:초) 표시
- 시간대 변환 — 넘어온 값의 시간대를 그대로 쓴다
- 상대 시간 표기 ("3일 전")
- 기존 코드에서 날짜를 찍는 곳을 찾아 교체하는 작업. 이번에는 유틸만 만든다
- 날짜 파싱·검증 라이브러리 도입

## 백엔드

`public-server` 의 `libs/common/src/utils/date.util.ts` 에 만든다.
같은 디렉토리의 `money.util.ts` 와 같은 모양으로 맞춘다.

테스트는 `libs/common/src/utils/date.util.spec.ts` 에 둔다.

## 프론트엔드

`public-front` 의 `src/utils/date.ts` 에 만든다.
같은 디렉토리의 `money.ts` 와 같은 모양으로 맞춘다.

테스트는 `src/utils/date.test.ts` 에 vitest 로 둔다.

두 저장소는 코드를 공유하지 않으므로 구현을 각각 두되, **규칙은 완전히 같아야
한다.** 경계 케이스를 양쪽 테스트에 나란히 넣어 한쪽만 바뀌면 드러나게 한다.

## 수용 기준 (Acceptance Criteria)

- Given `new Date("2026-08-19T10:30:00Z")` When `formatDate` 를 부르면 Then `"2026-08-19"` 를 반환한다
- Given 문자열 `"2026-01-05"` When `formatDate` 를 부르면 Then `"2026-01-05"` 를 반환한다
- Given `new Date("아무거나")` (Invalid Date) When `formatDate` 를 부르면 Then `""` 을 반환한다
- Given `null` When `formatDate` 를 부르면 Then `""` 을 반환하고 예외를 던지지 않는다
- Given 빈 문자열 `""` When `formatDate` 를 부르면 Then `""` 을 반환한다

## 참고

- 같은 목적으로 먼저 만든 금액 유틸: `libs/common/src/utils/money.util.ts`, `src/utils/money.ts`
