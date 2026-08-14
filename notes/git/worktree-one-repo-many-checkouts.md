# git worktree — 저장소 하나에 작업 디렉터리를 N개 붙인다

> 브랜치 전환은 하나뿐인 작업 디렉터리를 갈아끼우는 일이라 stash 왕복을 강제한다.
> worktree 는 같은 `.git` 을 공유하는 체크아웃을 여러 개 만들어 이 왕복 자체를 없앤다.
> 오브젝트·refs 는 공유되고, HEAD·index·작업 파일만 디렉터리별로 분리된다.

**상태**: 활성
**마지막 검증**: 2026-08-14

## 왜 그렇게 동작하나

- `[실측]` 브랜치 3개를 동시에 열고 **같은 파일을 서로 다르게** 편집해도 섞이지 않는다. 각 worktree 가 자기 index 와 작업 파일을 가지므로 미커밋 변경이 완전히 독립이다.
- `[실측]` 한 worktree 의 커밋은 fetch 없이 다른 worktree 에서 즉시 조회된다 — 오브젝트 저장소와 refs 가 하나이기 때문.
- `[실측]` 같은 브랜치를 두 worktree 에 체크아웃하는 것은 git 이 `fatal: '<branch>' is already used by worktree` 로 막는다. 단, 그 브랜치를 **base 로 삼아 새 브랜치를 따는 것**은 된다.
- `[실측]` 빌드 산출물(`__pycache__`, `.pyc`)이 디렉터리별로 격리된다. 한쪽에서 `compileall` 을 돌려도 다른 쪽 `.pyc` 는 mtime 조차 안 변한다. 브랜치 전환 방식에서는 이전 브랜치의 `.pyc` 가 남아 삭제된 모듈이 import 되는 유령 버그가 가능하다.
- `[실측]` **untracked·ignored 파일은 새 worktree 에 없다.** venv, 로컬 conf, 캐시는 git 밖이라 따라오지 않는다. 리포마다 "새 worktree 에서 바로 작업이 되나"의 세팅 비용이 다르다.

## 확인하는 방법

```
$ git -C <main> worktree add <path-b> -b task-b origin/development
$ echo "# edit-B" >> <path-b>/mod/core.py       # A/B/메인에 서로 다른 편집
$ git -C <main> stash list | wc -l               # 전환 없이 동시 작업 → stash 불변
9
$ git -C <path-b> commit -am tmp && git -C <main> log -1 --format=%h <sha>
53e93ba                                          # fetch 없이 즉시 보임
$ git -C <main> worktree add /tmp/dup <main-branch>
fatal: '<main-branch>' is already used by worktree at '<main>'
```

## 이 노트가 틀렸다면

- 두 worktree 에서 같은 브랜치가 동시에 체크아웃되면 → 이중 체크아웃 차단 주장이 틀렸다 (`--force` 없이 그랬다면)
- 한쪽 커밋이 fetch/pull 없이는 다른 쪽에서 안 보이면 → 오브젝트 공유 주장이 틀렸다
- 새 worktree 에 `.gitignore` 대상 파일이 나타나면 → 미상속 주장이 틀렸다
- 한 worktree 의 미커밋 편집이 다른 worktree 파일에 반영되면 → 격리 주장이 틀렸고 이 노트 전체를 폐기해야 한다

## 적용 범위

- macOS, git 2.x 에서 실측 (2026-08-14). `git worktree` 는 2.5+ 기능. 정확한 실측 버전은 기록 안 함 — 미확인
- Orca 등 worktree 관리 도구를 얹어도 밑은 같은 메커니즘

## 함정

- ⚠️ ignored 파일 미상속이 실질 도입 비용이다. venv·로컬 conf 가 필요한 리포는 세팅 자동화(훅) 없이는 worktree 마다 재구성해야 한다
- ⚠️ 정리 습관이 없으면 stash 쌓이던 문제가 **worktree·브랜치 쌓이는 문제로 형태만 바뀐다**
- ⚠️ worktree 디렉터리를 rm 으로만 지우면 메타데이터가 남는다 — `git worktree remove` + `git worktree prune`
- 💡 worktree 가 값을 하는 조건은 "추적 중인 태스크가 여러 개"가 아니라 **"편집 중인 코드가 동시에 여러 개"** 다. 순차 작업이면 오버헤드만 는다

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-14 | 3중 동시 편집·오브젝트 공유·이중 체크아웃 차단·pyc 격리·미상속 5항목 실측 | 최초 작성, 전부 `[실측]` |

## 관련

- [[stash-archive-with-protected-refs]]
- [[venv-breaks-when-repo-moves]]
