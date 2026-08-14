# squash merge 는 부모 하나짜리 일반 커밋을 만든다 — 브랜치를 써도 갈래는 히스토리에 안 남는다

> merge commit 은 부모가 둘(기존 HEAD + 브랜치 끝)이라 `log --graph` 에 갈래가 그려진다.
> squash merge 는 브랜치 커밋들의 변경분만 합쳐 **부모 하나짜리 새 커밋**을 만들므로 main 은 계속 일직선이다.
> "브랜치 단위로 묶인 히스토리를 그래프로 보고 싶다"는 요구는 squash-only 정책과 **구조적으로 양립 불가**다 — 브랜치를 파서 작업해도 결과는 같다.

**상태**: 활성
**마지막 검증**: 2026-08-14

## 왜 그렇게 동작하나

- `[실측]` merge commit 은 커밋 오브젝트에 `parent` 줄이 두 개, squash 커밋은 한 개다 (아래 출력).
- `[문서]` `git merge --squash` 는 병합 결과를 워킹트리·인덱스에만 만들고 `MERGE_HEAD` 를 기록하지 않는다. 이후 `git commit` 은 평범한 단일 부모 커밋이 된다 (`git-merge(1)`). GitHub "Squash and merge" 도 같은 방식이다.
- `[문서]` 원본 브랜치의 커밋들은 main 도달 이력이 없으므로, 브랜치 ref 가 지워지면 GC 대상이다. 중간 과정은 PR 페이지(플랫폼 DB)에만 남는다.
- `[코드]` squash 커밋은 일반 커밋이라 `git revert <sha>` 가 바로 되지만, merge commit 은 `-m 1` 로 어느 부모 기준인지 지정해야 한다 — 어느 revert 를 자주 하느냐가 정책 선택의 실익 중 하나다. (revert 는 이번에 직접 돌리지 않았다)

## 확인하는 방법

```
$ git merge --no-ff feature -m M && git cat-file -p HEAD | grep ^parent
parent d9e92b4cf89b877e030ac3081b4a7927d83f9d12
parent 57f4c2cfbb37c4ab9dd8bab95e8e5fc0c08194c5

$ git reset --hard HEAD~1 && git merge --squash feature && git commit -m S
$ git cat-file -p HEAD | grep ^parent
parent d9e92b4cf89b877e030ac3081b4a7927d83f9d12

$ git log --graph --oneline
* 1960281 S
* d9e92b4 A
```

## 이 노트가 틀렸다면

- squash merge 로 만든 커밋에서 `git cat-file -p` 가 `parent` 를 두 줄 보여준다 → 핵심 주장이 틀렸다.
- squash 후 `git log --graph` 에 병합 갈래가 그려진다 → 핵심 주장이 틀렸다.
- 브랜치 ref 삭제 후에도 원본 커밋이 `git fsck --unreachable` 에 안 잡히고 main 에서 도달 가능하다 → GC 대상 주장이 틀렸다.

## 적용 범위

- git 2.x CLI 로컬 동작 실측 (2026-08-14, git for macOS). GitHub/GitHub Enterprise 의 "Squash and merge" 는 `[문서]` 근거.
- rebase merge(커밋들을 재작성해 일렬로 올리는 제3의 방식)는 이 노트 범위 밖이다.

## 함정

- ⚠️ 팀에서 "브랜치 만들어서 머지하자"는 제안이 나왔을 때, 레포가 squash-only 면 브랜치 전략만 바꿔도 히스토리 모양은 안 바뀐다. 논쟁의 실제 쟁점은 브랜치 운용이 아니라 **머지 방식 설정**이다.
- 💡 squash-only 를 유지하면서 "기능 단위로 보기"를 만족시키는 절충: PR 스코프를 좁게(디렉토리/모듈 하나) 강제 + `git log -- <dir>` 로 경로 필터 조회 + PR 제목 프리픽스 규칙.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-14 | 임시 레포에서 두 방식 실행, 부모 수·그래프 비교 | 최초 작성, 핵심 주장 `[실측]` |

## 관련

- [[worktree-one-repo-many-checkouts]]
