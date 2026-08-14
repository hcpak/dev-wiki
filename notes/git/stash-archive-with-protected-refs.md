# stash 는 커밋이다 — ref 를 붙이면 drop 후에도 복원된다

> `git stash` 항목 하나하나가 커밋 객체라서, `update-ref` 로 이름을 붙여두면
> `stash clear` 뒤에도 gc 에 살아남고 `stash apply <ref>` 로 되살릴 수 있다.
> 수십 개 쌓인 옛 stash 를 "지우긴 불안한데 두긴 지저분한" 상태에서 꺼내는 방법.

**상태**: 활성
**마지막 검증**: 2026-08-14

## 왜 그렇게 동작하나

- `[실측]` `git update-ref refs/stash-archive/<이름> <stash-sha>` 로 보호 ref 를 만들면, `stash clear` 후에도 `git stash apply refs/stash-archive/<이름>` 이 동작한다. ref 로 도달 가능한 커밋은 gc 가 지우지 않는다.
- `[실측]` stash 커밋의 1번 부모(`<ref>^1`)가 **stash 를 만들던 시점의 HEAD** 다. 현재 브랜치에 apply 하다 충돌하면, `git worktree add --detach <tmp> <ref>^1` 로 원래 시점을 열고 거기에 apply 하면 원형 그대로 살아난다.
- `[실측]` 현재 브랜치에 apply 했는데 **no-op(working tree clean)이면 그 내용이 이미 브랜치에 들어가 있다는 뜻**이다. 실측한 사례에서는 커밋 82초 뒤에 만들어진 stash 였다 — 같은 편집을 다른 브랜치 작업 트리에서 치우면서 남긴 중복 스냅샷. 오래 방치된 stash 의 상당수가 이 유형이라, drop 전 "살아있는 작업인지" 판별에 쓸 수 있다.
- `[문서]` 커스텀 네임스페이스 ref 는 clone 이 가져가지 않는다 (기본 fetch refspec 은 `refs/heads/*` + tags). 새 clone 에서도 필요하면 `stash show -p` 로 뽑은 패치 파일을 병행 보관해야 한다.

## 확인하는 방법

```
$ git update-ref refs/stash-archive/00-2026-02-06 <stash-sha>
$ git stash clear && git stash list | wc -l
0
$ git cat-file -t refs/stash-archive/00-2026-02-06
commit
$ git worktree add --detach /tmp/restore $(git rev-parse refs/stash-archive/00-2026-02-06^1)
$ git -C /tmp/restore stash apply refs/stash-archive/00-2026-02-06
$ git -C /tmp/restore diff --stat | tail -1
 1 file changed, 56 insertions(+), 5 deletions(-)   # 원래 시점에서 원형 복원
```

## 이 노트가 틀렸다면

- `stash clear` + `git gc --prune=now` 후 아카이브 ref 의 apply 가 실패하면 → "ref 가 gc 를 막는다"가 틀렸다
- clone 한 리포에 `refs/stash-archive/*` 가 보이면 → 미전파 주장이 틀렸다 (fetch 설정을 확인할 것)
- 현재 브랜치 apply 가 no-op 인데 그 코드가 브랜치에 없으면 → 중복 스냅샷 판별법이 틀렸다
- `<ref>^1` 이 stash 생성 시점 HEAD 가 아닌 사례가 나오면 → 복원 절차의 전제가 무너진다

## 적용 범위

- 2026-08-14, 11개 리포 stash 44개(2023-05 ~ 2026-04)로 실측. 백업 → clear → 복원 리허설까지 왕복 확인
- untracked 파일을 포함한 stash(`-u`)는 3번째 부모에 들어가는데, 이번 실측 대상에 없었다 — `-u` stash 의 복원은 미확인

## 함정

- ⚠️ `git stash drop`/`clear` 는 reflog 에도 안 남는다 — 아카이브는 반드시 **clear 전에**
- ⚠️ 패치 파일 백업은 `stash show -p` 기준이라 바이너리·untracked 는 빠질 수 있다. ref 백업을 정본으로, 패치는 사람이 읽는 용도로
- 💡 아카이브 이름에 날짜를 넣으면(`00-2026-02-06`) 나중에 manifest 없이도 ref 목록만으로 훑을 수 있다

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-14 | 44개 백업 → clear → ref apply·원시점 복원·no-op 판별 실측 | 최초 작성, clone 미전파만 `[문서]` |

## 관련

- [[worktree-one-repo-many-checkouts]]
