# 같은 소프트웨어라도 패키지 출처가 바뀌면 실행 유저가 바뀔 수 있다 — 파일 소유권이 지뢰가 된다

> 배포판 레포의 deb 와 벤더(업스트림) 레포의 deb 는 같은 프로그램이라도 systemd 유닛의 `User=` 가 다를 수 있다.
> 출처를 갈아타는 업그레이드/다운그레이드에서 기존 데이터·로그 파일의 소유자가 새 유저와 어긋나
> permission denied 로 기동이 실패한다. 양방향 모두에서 터진다.

**상태**: 활성
**마지막 검증**: 2026-08-20

## 왜 그렇게 동작하나

`[실측]` Ubuntu 배포판의 telegraf 패키지(1.21.4+ds1)는 `User=_telegraf` 로,
벤더(influxdata 계열) 패키지(1.31.2-1)는 `User=telegraf` 로 서비스를 돌린다.
배포판은 시스템 유저에 `_` 접두를 붙이는 관례(debian 정책 권고)를 따르고, 벤더는 자기 관례를
따르기 때문에 **같은 프로그램이 유저 두 개를 갖게 된다.**

`[실측]` 이 상태에서 출처를 갈아타면:

- **업그레이드** (배포판 → 벤더): 기존 로그 파일이 `_telegraf` 소유 → 새 유닛(`telegraf`)이
  열지 못해 기동 실패. 디렉터리는 새 패키지 postinst 가 chown 해도 **안의 기존 파일까지는
  건드리지 않는다** — 디렉터리 권한만 보고 "정상"이라 오판하기 쉽다.
- **다운그레이드** (벤더 → 배포판): 위에서 `chown -R telegraf` 로 고쳐둔 파일들이 이번엔
  `_telegraf` 가 못 열어 같은 방식으로 실패. **수습 자체가 반대 방향의 지뢰를 심는다.**

`[추정]` telegraf 특유의 문제가 아니라, 배포판/벤더 이중 패키징이 있는 데몬(grafana, nginx,
redis 등)에서 일반적으로 가능한 구조다 — 다른 패키지에서는 미확인.

## 확인하는 방법

갈아타기 전에 두 패키지의 유닛 유저와 기존 파일 소유자를 대조한다.

```
$ grep User= /lib/systemd/system/telegraf.service
User=_telegraf                        # 배포판 판

# 벤더 판 설치 후 같은 파일:
User=telegraf

$ ls -l /var/log/telegraf/
-rw-r--r-- 1 _telegraf _telegraf ... telegraf.log   # 소유자가 새 유닛 유저와 다르면 기동 실패 예정
```

실패 시 로그 (journalctl):

```
E! Unable to open /var/log/telegraf/telegraf.log (permission denied), using stderr
E! [agent] Failed to connect to [outputs.file] ... permission denied
E! [telegraf] Error running agent: connecting output outputs.file: context canceled
```

## 이 노트가 틀렸다면

- 배포판↔벤더 패키지 교체 후 chown 없이도 서비스가 정상 기동한다 → 해당 패키지 쌍은 유저가 같거나 postinst 가 재귀 chown 을 해준다는 뜻 (적용 범위 축소 — telegraf 외 패키지마다 확인 필요)
- 두 패키지의 유닛 `User=` 가 같다 → 이 노트의 전제가 그 패키지엔 해당 없음
- 디렉터리 chown 만으로(파일 제외) 기동이 성공한다 → "기존 파일까지 어긋난다" 주장이 그 케이스엔 과함 (로그를 append 가 아니라 새 파일로 여는 프로그램일 수 있음)

## 적용 범위

- deb 계열, systemd 유닛을 패키지가 소유하는 데몬. telegraf 1.21.4(Ubuntu 22.04 universe) ↔ 1.31.2(벤더 미러) 쌍에서 양방향 실측.
- rpm 계열·컨테이너 배포는 미확인.

## 함정

- ⚠️ **업그레이드 계획에 "chown 단계"를 명시할 것.** 설치 성공 ≠ 기동 성공이고, 이 실패는
  설치 로그에는 전혀 안 보인다.
- ⚠️ 수습으로 chown 한 뒤 롤백하면 **같은 문제가 반대 방향으로 재발**한다. 롤백 절차에도
  원래 유저로의 chown 을 포함해야 한다.
- 💡 판별 한 줄: `systemctl cat <svc> | grep User=` 를 교체 전후로 비교. 다르면 데이터·로그
  디렉터리 전체(`state`, `spool` 포함)를 새 유저로 chown.
- 사례 정황: 배포판 판을 쓰던 이유가 "옛 설정 키가 그 버전 문법이라서"처럼 버전 고정과
  얽혀 있을 수 있다 — 출처 교체는 보통 버전 점프와 함께 오므로 설정 호환성 검토도 세트다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-20 | 최초 작성 — 배포판(1.21.4, `_telegraf`)→벤더(1.31.2, `telegraf`) 업그레이드에서 로그 파일 permission denied 기동 실패, chown 후 정상. 이어 다운그레이드에서 역방향 재발, 원래 유저로 chown 후 정상 — 양방향 실측 | `[실측]` 확립 |
