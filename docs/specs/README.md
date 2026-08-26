# specs

SpecFlow 파이프라인이 읽는 spec 문서를 두는 곳이다.

매일 01:00에 이 디렉토리를 훑어 **`status: ready` 이면서 아직 처리하지 않은** spec을 찾고,
백엔드 → 프론트엔드 → QA 순으로 구현·검증한 뒤 feature 브랜치와 draft PR을 만든다.

## 새 spec 쓰기

```bash
cp ~/.specflow/templates/spec.template.md docs/specs/002-무언가.md
```

작성 중에는 `status: draft` 로 두고, 다 쓰면 `ready` 로 바꾼다.
`ready` 로 바꾸는 순간부터 다음 실행에서 집어간다.

## 쓰기 전에 확인

```bash
~/.specflow/lib/validate-spec.sh docs/specs/002-무언가.md
```

종료 코드 `0` 이면 통과다. 경고는 있어도 실행되지만, 특히
`## 비요구사항 (Out of scope)` 이 비어 있으면 에이전트가 주변 기능까지 건드릴 위험이 커진다.

형식 규칙은 `~/.specflow/SCHEMA.md` 에 있다.

## 주의

- `id` 는 한번 정하면 바꾸지 않는다. 브랜치명과 처리 이력의 키라서, 바꾸면 새 spec으로 인식되어 중복 실행된다.
- 이미 처리된 spec의 내용을 고쳐도 기본적으로 다시 실행되지 않는다. 열려 있는 PR을 덮어쓰지 않기 위해서다.
- 파일명은 자유지만 `NNN-슬러그.md` 형태를 권한다. 정렬이 편하고 우선순위가 같을 때 파일명 순으로 처리된다.
