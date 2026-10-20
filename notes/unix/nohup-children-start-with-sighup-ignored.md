---
status: 활성
verified: 2026-10-19
---
# nohup 아래에서 띄운 프로세스는 SIGHUP 을 무시한 채 태어난다 — 그 안에서 한 SIGHUP 시험은 무효다

> nohup 은 SIGHUP 을 SIG_IGN 으로 바꾼 뒤 명령을 exec 하고, 무시 상태는 exec 와 fork 를 지나 자손에게 그대로 넘어간다.
> SIG_IGN 을 물려받으면 존중하도록 짠 프로그램은 물론, SIGHUP 을 전혀 다루지 않는 프로그램도 `kill -HUP` 에 살아남는다.
> 원격 실행 도구가 스크립트를 nohup 으로 떼어 돌리면, 그 스크립트 안의 "ssh 가 끊기면 정리되는가" 시험은 수정 전 빌드와 수정 후 빌드가 똑같이 "살아 있음" 으로 나온다.

## 왜 그렇게 동작하나

`[문서]` nohup 은 SIGHUP 을 무시하도록 설정한 뒤 대상 명령을 실행한다 (nohup(1)).

`[문서]` `execve` 는 핸들러로 받던 시그널만 기본 동작으로 되돌리고, 무시하던 시그널은 무시된 채로 둔다. fork 한 자식도 부모의 시그널 처리 상태를 물려받는다 (execve(2), fork(2)). 그래서 nohup 아래의 셸이 띄운 모든 자손이 SIGHUP 을 무시한다.

`[문서]` bash 는 시작할 때 이미 무시된 시그널을 `trap` 으로 바꿀 수 없다 ("Signals ignored upon entry to the shell cannot be trapped or reset"). 스크립트 안에서 `trap - HUP` 로 되돌리는 방법은 통하지 않는다.

`[실측]` nohup 으로 떼어 돌린 시험 스크립트 안에서, SIGHUP 처리가 없는 옛 빌드와 SIGHUP 핸들러를 넣은 새 빌드 모두 `kill -HUP` 후에도 살아 있었다. 셸의 `/proc/$$/status` 에서 `SigIgn` 의 비트 0(SIGHUP)이 1 이었다.

`[실측]` 시험 대상만 `env --default-signal=HUP <tool>` 로 띄우자 옛 빌드는 "Hangup" 으로 죽으며 설정을 남겼고, 새 빌드는 정리하고 끝났다. 기대한 차이가 그때서야 드러났다.

## 확인하는 방법

```
$ echo $(( 0x$(awk '/^SigIgn/{print $2}' /proc/$$/status) & 1 ))
1                         # 1 이면 이 셸은 SIGHUP 을 무시한 채 시작됐다

$ env --default-signal=HUP <tool> & kill -HUP $!
[1]+  Hangup   <tool>     # 기본 동작으로 돌아가 HUP 에 죽는다
```

## 이 노트가 틀렸다면

- nohup 아래에서 띄운 자손의 `SigIgn` 비트 0 이 0 이다 → 그 nohup 구현이나 중간 프로그램이 무시 상태를 되돌린 것이다
- SIGHUP 처리가 없는 프로그램이 nohup 아래에서 `kill -HUP` 에 죽는다 → 무시 상태가 상속되지 않은 것이다
- bash 스크립트 안에서 `trap - HUP` 뒤 띄운 자식이 HUP 에 죽는다 → 이 bash 는 진입 시 무시된 시그널도 되돌린다

## 적용 범위

`[실측]` Ubuntu 22.04 의 bash 와 coreutils `env` 에서 확인했다.
`[추정]` `env --default-signal` 은 비교적 최근 coreutils 에 들어온 옵션이라 오래된 배포판에는 없을 수 있다 — 도입 버전은 확인하지 않았다. 없으면 `perl -e '$SIG{HUP}="DEFAULT"; exec @ARGV' <tool>` 이 같은 일을 한다(확인 안 함).

## 함정

- ⚠️ 원격 실행 래퍼가 세션 타임아웃을 피하려고 `nohup … &` 로 스크립트를 떼어 돌리면, 그 안의 SIGHUP 시험은 전부 무시 상태에서 돈다. 결과가 "수정 전도 수정 후도 살아 있음" 으로 나오면 버그가 아니라 이 상속을 먼저 의심한다.
- 💡 SIGHUP 핸들러를 넣는 프로그램은 시작할 때 이미 SIG_IGN 인지 보고, 그렇다면 그대로 두는 것이 관례다. 사용자가 nohup 으로 띄운 의도(터미널이 닫혀도 계속 돌기)를 존중하기 위해서다. 이 관례를 지킨 프로그램일수록 시험 환경의 nohup 에 더 쉽게 속는다.
- 같은 래퍼가 stdin 을 먹는 문제는 [[wrappers-that-read-stdin-swallow-piped-scripts]] 에 있다. 둘 다 "도구가 조용히 실행 조건을 바꾼다" 는 계열이다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-10-19 | 물리 노드에서 nohup 으로 떼어 돌린 SIGHUP 시험이 두 빌드 모두 생존으로 나옴, `env --default-signal=HUP` 로 재측정해 차이 확인 | `[실측]` 최초 작성 |

## 관련

- [[ignored-sigpipe-keeps-writer-running]]
- [[wrappers-that-read-stdin-swallow-piped-scripts]]
