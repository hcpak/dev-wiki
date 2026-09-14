---
status: 활성
verified: 2026-09-14
---
# `git describe` 는 annotated 태그만 본다 — 한 커밋에 lightweight 와 annotated 를 섞으면 도구마다 다른 태그를 읽는다

> `git describe --exact-match` 는 `--tags` 를 주지 않는 한 **annotated 태그만 후보**로 본다.
> 같은 커밋에 annotated `1.2.0rc1` 과 lightweight `1.2.0` 이 함께 있으면 describe 는 `1.2.0rc1` 을,
> `git log --decorate` 는 둘 다 돌려준다. 두 경로를 섞어 쓰는 버전 도구(pbr)는 "히스토리상 버전은
> 1.2.0 인데 목표 버전은 1.2.0rc1" 이라며 빌드를 중단한다. 한 커밋 위의 태그는 종류를 통일해야 한다.

## 왜 그렇게 동작하나

`[문서]` git-describe 매뉴얼: "By default (without --all or --tags) git describe only shows annotated tags."
lightweight 태그는 커밋을 가리키는 ref 일 뿐 tag 객체가 없어서 describe 의 기본 후보에서 빠진다.

`[실측]` 같은 커밋에 annotated 가 여러 개면 describe 는 **tagger 날짜가 최신인 것**을 고른다.
오래된 annotated `1.2.0rc1` 옆에 새 annotated `1.2.0` 을 찍자 describe 결과가 `1.2.0` 으로 바뀌었다.
(`--tags` 를 줘도 annotated 가 lightweight 보다 우선이라, lightweight `1.2.0` 은 여전히 선택되지 않았다.)

`[실측]` pbr 은 이 차이에 걸린다. 빌드 로그의 트레이스백:

```
File ".../pbr/packaging.py", line 642, in _get_version_from_git_target
ValueError: git history requires a target version of pbr.version.SemanticVersion(1.2.0),
            but target version is pbr.version.SemanticVersion(1.2.0.0rc1)
```

`[추정]` 트레이스백의 함수명과 에러 문구로 역추론한 pbr 의 동작 (이번에 pbr 소스를 열어 확인하지는 않았다):
목표 버전(target)은 `git describe --exact-match` 로 → annotated `1.2.0rc1`,
히스토리 버전(new)은 `git log --decorate` 에 보이는 태그 전부 중 최대 semver 로 → lightweight `1.2.0`.
new > target 이면 "히스토리가 요구하는 버전보다 목표가 낮다" 고 판단해 ValueError 를 던진다.

## 확인하는 방법

```
$ git tag --points-at HEAD
1.2.0            ← lightweight (cat-file -t → commit)
1.2.0a2          ← annotated   (cat-file -t → tag)
1.2.0rc1         ← annotated

$ git describe --exact-match HEAD
1.2.0rc1                                  ← lightweight 1.2.0 은 후보가 아니다
$ git describe --tags --exact-match HEAD
1.2.0rc1                                  ← --tags 를 줘도 annotated 우선

$ git tag -d 1.2.0 && git tag -a 1.2.0 HEAD -m "release"
$ git describe --exact-match HEAD
1.2.0                                     ← annotated 끼리는 tagger date 최신이 이긴다
```

이전 릴리스들이 lightweight 태그로도 잘 빌드됐다면, 같은 커밋의 rc 태그도 lightweight 였는지 본다.
그 경우 describe 가 아예 실패해(annotated 없음) pbr 의 target 검사 자체가 건너뛰어진 것이다.

```
$ for t in $(git tag --points-at <이전 릴리스 커밋>); do echo "$t $(git cat-file -t $t)"; done
1.1.0     commit
1.1.0rc5  commit          ← 둘 다 lightweight → describe 실패 → 검사 없음 → "됐던 것"
```

## 이 노트가 틀렸다면

- `--tags` 없이 `git describe --exact-match` 가 lightweight 태그를 돌려주는 관측 → 첫 주장이 틀림 (git 버전·alias·`describe.*` 설정을 먼저 의심)
- 같은 커밋의 annotated 두 개 중 **tagger 날짜가 오래된 쪽**을 describe 가 고르는 관측 → "최신 우선" 은 틀리고 다른 기준(이름순 등)이 있다는 뜻
- lightweight 릴리스 태그 + annotated rc 태그 조합에서 pbr 빌드가 정상 통과하는 관측 → pbr 버전별로 검사가 다르다는 뜻, 적용 범위를 그 버전으로 좁힌다

## 적용 범위

- git 2.50.1 로컬 실측. describe 의 annotated 우선 규칙은 오래된 동작이라 버전 의존이 낮다고 보지만, 다른 버전에서는 확인하지 않았다.
- pbr: python2.7 `dist-packages` 의 pbr (정확한 버전은 미확인). `setup.cfg` 에 `version` 이 없어 태그에서 버전을 뽑는 구성.
- 버전을 태그에서 계산하는 다른 도구(setuptools-scm, versioneer 등)는 describe 옵션이 달라 이 노트를 그대로 적용하지 않는다.

## 함정

- ⚠️ "이전엔 lightweight 로 잘 됐다" 는 근거가 되지 않는다. 그때는 rc 도 lightweight 라 검사가 생략됐을 뿐, 규칙이 바뀐 것이 아니다.
- 💡 한 커밋 위의 태그는 종류를 통일한다 — rc 를 `git tag -a` 로 찍었으면 릴리스도 `git tag -a`. 그러면 describe 가 최신 태그를 고른다.
- ⚠️ 복구하려고 같은 이름의 태그를 지웠다 다시 밀면, 태그 push 로 트리거되는 CI 잡이 **다시 돌지 않을 수 있다** (Jenkins 태그 잡에서 1회 실측 — 수동 실행이 필요했다). 재태깅 뒤에는 빌드가 실제로 시작됐는지 본다.
- 💡 실패한 첫 빌드는 산출물을 올리지 않으므로 버전 번호는 소모되지 않는다 — 같은 버전으로 재빌드하면 된다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-09-14 | 최초 작성 — lightweight 릴리스 태그가 pbr 빌드를 깨는 것을 빌드 로그로, describe 후보 규칙을 로컬 재현으로 확인 | `[실측]` (pbr 내부 동작은 `[추정]`) |

## 관련

- 한 번의 git push 에서 refspec 들은 독립적으로 성공하고 실패한다 (`git/`) — 태그가 빌드를 낳는 경로의 다른 함정
- 폴링 빌드 서버에서 트리거 실패는 빌드 실패가 아니다 (`packaging/`) — 반대로 웹훅형 CI 는 트리거 유실이 곧 빌드 누락이다
