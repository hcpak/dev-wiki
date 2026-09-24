---
status: 활성
verified: 2026-09-24
---
# 감시자의 재시작이 서비스 초기화보다 빠르면 그 서비스는 영원히 뜨지 못한다 — 죽인 쪽은 launchd 로그의 caller 체인이 말해준다

> "백엔드에 연결할 수 없다" 는 화면 뒤에서, 연결 실패 때마다 백엔드를 내리고 다시 올리는 클라이언트가
> 갓 뜬 백엔드를 초기화 도중 매번 죽이고 있었다. 백엔드가 준비되기까지 수 초가 걸리는데 재시작 간격이
> 그보다 짧으면 루프는 수렴하지 않는다. 멀쩡히 며칠을 돌던 인스턴스도 같은 루틴이 죽였다.
> 원인은 백엔드가 아니라 **감시자의 재시작 정책**이고, launchd 의 `booting out service: caller = …`
> 줄이 누가 내렸는지를 프로세스 체인으로 적어 준다.

## 왜 그렇게 동작하나

- `[실측]` launchd 가 백엔드를 spawn 한 뒤 1.5 초 만에 같은 서비스에 대한 `booting out service` 가
  찍혔고, caller 체인은 `launchctl <- bash <- <frontend>` 였다. 다른 회차에는 spawn 0.15 초 뒤였다.
  백엔드는 장치 열거까지만 3.5 초가 걸리므로, 어느 회차도 리스너를 열기 전에 SIGTERM 을 받았다.
- `[실측]` 며칠째 정상 동작하며 포트를 열고 있던 백엔드도, 프론트가 기동 직후 접속에 실패하자 같은
  체인으로 내려갔다. 즉 이 루틴은 "죽은 백엔드 살리기" 가 아니라 "접속 실패 시 무조건 재시작" 이다.
- `[코드]` 프론트 번들에서 백엔드 유닉스 소켓 접속의 `reconnectInterval` 이 100 ms 였고, 접속 실패
  경로에 `launchctl bootout` → `bootstrap` 을 실행하는 재시작 루틴이 있다. 두 값이 이 간격으로 묶여
  있는지(재시도 N 회 뒤 재시작인지)는 코드를 끝까지 읽지 않았다.
- `[실측]` 커널이 프론트 프로세스를 "33 초 동안 CPU 를 45,001 번 깨웠다" 고 기록했다(초당 약 1,360 회).
  100 ms 재접속만으로는 초당 10 회이므로 다른 폴링이 섞여 있다. 무엇이 깨우는지는 확인하지 않았다.
- `[실측]` 프론트를 먼저 끄고 백엔드만 launchd 에 올린 뒤 프론트를 켰을 때는 더 이상 `booting out` 이
  찍히지 않았다. 이번 1 회 단독 기동에서는 백엔드가 살아남았다.
- `[실측]` 그 단독 기동 인스턴스가 느렸던 이유는 별개다 — 의존하는 업데이터 데몬이 응답하지 않아 기동
  요청이 워치독 경고(`has not completed in a long time`)만 남기며 3 분 22 초 뒤에 끝났다. 감시자가
  없었기에 그동안 살아 있을 수 있었고, 프론트가 붙어 있었다면 이 인스턴스도 1.5 초 안에 내려갔을 것이다.
  프론트는 이 지연을 "백엔드 연결 문제" 로 표시했다. **의존 서비스의 무응답은 바깥에서 죽음이 아니라
  느린 시작으로 보인다.**

## 확인하는 방법

launchd 의 spawn·bootout 이벤트는 debug 레벨이라 `--debug` 없이는 `removing service` 만 보인다.

```
$ /usr/bin/log show --start "<HH:MM:SS>" --end "<HH:MM:SS>" --info --debug --style compact \
    --predicate 'process == "launchd" AND eventMessage CONTAINS "<label>"'
11:35:16.897 launchd [gui/501/<label> [50194]:] Successfully spawned <agent>[50194] because speculative
11:35:18.450 launchd [gui/501/<label> [50194]:] booting out service: caller = launchctl[50232]<-bash[50227]<-<frontend>[48512], value = 0x0
11:35:18.450 launchd [gui/501/<label> [50194]:] signaled service: Terminated: 15
11:35:23.461 launchd [gui/501 [100019]:] removing service: <label>
```

판별 기준 두 가지.

1. `booting out … caller = …` 가 있으면 **launchctl 을 거쳐 외부가 내린 것**이다. 체인의 마지막
   프로세스가 감시자다. 이 줄이 없고 `exited due to …` 만 있으면 launchctl 을 거치지 않은 외부 `kill`
   이거나 자체 크래시다 — 크래시 리포트(DiagnosticReports, 앱 자체 크래시 DB)의 유무로 가른다.
2. spawn 과 bootout 의 간격이 그 서비스의 콜드 스타트보다 짧으면 루프다. 콜드 스타트는 감시자를
   멈춘 채 혼자 올려 리스너가 열리는 시각(`lsof -iTCP -sTCP:LISTEN -p <pid>`)으로 잰다.

루프를 끊는 순서: 감시자 종료 → 서비스만 기동 → 리스너 확인 → 감시자 기동.

## 이 노트가 틀렸다면

- 백엔드가 리스너를 연 뒤에도 프론트가 접속하지 못하고 bootout 이 이어진다면 → 루프는 결과이고 원인은
  접속 경로(소켓 경로·권한·프로토콜)다. 이 노트로 설명되지 않는다.
- `booting out` 줄 없이 `exited due to signal` 이나 크래시 리포트가 있다면 → 감시자가 아니라 서비스
  자체가 죽는 것이다.
- 감시자가 재시작 전에 준비 상태(소켓 open, 헬스 응답)를 확인하는 구현이라면 → 이 함정이 없다.
  적용 범위가 "접속 실패·타이머만 보고 재시작하는 감시자" 로 좁아진다.
- 콜드 스타트가 재시작 간격보다 항상 짧다면 → 루프는 수렴한다. 이 노트는 의존 서비스 지연처럼
  콜드 스타트가 늘어난 조건에서만 발현한다.

## 적용 범위

- 확인한 것은 macOS launchd 사용자 에이전트 + 접속 실패 시 `launchctl bootout/bootstrap` 을 부르는
  Electron 프론트다.
- 💡 같은 구조라면 systemd 의 `Restart=` + 짧은 `RestartSec` 을 클라이언트가 `systemctl restart` 로 흔드는
  경우, k8s 의 startupProbe `failureThreshold × periodSeconds` 가 콜드 스타트보다 짧은 경우에도 성립할
  것으로 본다([[probes]]). 이쪽은 실측하지 않았다.
- launchd 자체의 `KeepAlive` 재시작 스로틀(기본 10 초)은 이 사건과 무관했다. 죽인 것은 앱 레벨 루틴이다.

## 함정

- ⚠️ 프로세스가 "지금 없다" 는 것만 보고 크래시로 단정하기 쉽다. 크래시 리포트가 없고 앱의 크래시 DB 런
  디렉터리가 정리되지 않은 채 남아 있으면(비정상 종료 흔적) 밖에서 SIGTERM/SIGKILL 을 받은 쪽을 먼저
  의심하고, 위 판별 기준 1 로 launchctl 경유 여부를 가른다.
- ⚠️ 감시자가 죽인 인스턴스와 자체 종료한 인스턴스가 한 시간대에 섞인다. pid 별로 spawn→bootout/exit 를
  짝지어 봐야 한다.
- ⚠️ 서비스가 "start -- end" 같은 기동 완료 로그를 남겨도 프론트 접속 리스너가 그 시점에 열린다는 보장은
  없다. 준비 판정은 리스너·소켓으로 한다.
- 💡 관련 도구 함정: zsh 에서 `log` 는 내장 명령이라 `/usr/bin/log` 를 절대경로로 불러야 한다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-09-24 | 최초 작성 — Logi Options+ 프론트가 백엔드 에이전트를 반복 bootout 하는 launchd 로그 대조, 감시자 정지 후 단독 기동으로 생존 확인 | `[실측]`, 프론트 재시작 조건은 `[코드]` |

## 관련

- [[probes]] — startupProbe 가 콜드 스타트보다 짧을 때 같은 루프가 CrashLoopBackOff 로 나타난다
- [[lost-response-is-not-a-failed-write]] — 응답이 없다는 관측과 실패했다는 판정은 다르다
- [[process-age-does-not-mean-abandoned]] — 오래 뜬 프로세스를 함부로 내리지 않는 쪽의 짝
