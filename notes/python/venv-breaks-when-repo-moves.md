# venv 는 디렉터리를 옮기면 조용히 죽는다

> venv 안의 실행 스크립트(pip 포함)는 셔뱅에 **생성 시점의 절대경로**를 박는다.
> 리포를 다른 경로로 옮기면 전부 `bad interpreter` 로 죽는데, 에러를 띄우는 게 아니라
> 에디터 lint·포맷·자동완성이 **말없이 사라지는 형태**로 나타난다.

**상태**: 활성
**마지막 검증**: 2026-08-14

## 왜 그렇게 동작하나

- `[실측]` `.venv/bin/pip` 의 셔뱅이 `#!<옛 경로>/.venv/bin/python3.13` 처럼 절대경로다. 리포를 옮긴 뒤 실행하면 `bad interpreter: ... no such file or directory`.
- `[코드]` `pyvenv.cfg` 의 `home =` 도 절대경로라 심볼릭 링크 재생성으로도 완전히 복구되지 않는다 — 셔뱅만 고쳐서 살리는 방법은 검증 안 했다.
- `[실측]` 죽은 venv 는 디스크에 그대로 남아 정상처럼 보인다. 실측 사례에서는 117MB 짜리가 몇 달간 방치돼 있었고, 내용물(에디터 툴링 19종)이 전부 실행 불능이었다.

## 확인하는 방법

```
$ head -1 .venv/bin/pip
#!/Users/<user>/<옛-경로>/<repo>/.venv/bin/python3.13
$ .venv/bin/pip list
bad interpreter: ... no such file or directory
```

## 이 노트가 틀렸다면

- 옮긴 venv 의 `pip`·스크립트가 정상 실행되면 → 틀렸다 (상대경로 셔뱅을 쓰는 생성 도구였는지 확인 — 이 노트는 표준 `venv` 모듈 기준)
- `python -m venv --upgrade` 등으로 이동 후 복구가 되는 사례가 확인되면 → "재생성이 유일한 답"이라는 함정 항목의 범위가 좁아진다

## 적용 범위

- 표준 라이브러리 `venv` (Python 3.13 생성분 실측). virtualenv·uv 등 다른 도구는 미확인

## 함정

- ⚠️ **조용히 죽는다**가 핵심이다. import 에러가 아니라 "에디터가 요즘 lint 를 안 해주네" 로 나타나서 원인을 venv 에서 찾기 어렵다
- 💡 리포 이동·이름 변경 후에는 `head -1 .venv/bin/pip` 한 줄로 즉시 판별된다
- 💡 고치려 들지 말고 지우고 재생성이 답이다. 내용물은 `pip list` 가 안 돌면 `ls .venv/lib/python*/site-packages` 로 파악

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-14 | 이동된 리포의 죽은 venv 실측 (셔뱅·bad interpreter·방치 규모) | 최초 작성 |

## 관련

- [[worktree-one-repo-many-checkouts]] — worktree 는 venv 를 물려주지 않아서 이 함정을 새 체크아웃마다 만난다
