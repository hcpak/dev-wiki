# oslo.service 주기 루프는 콜백 예외 한 번에 영구히 멈춘다

> `FixedIntervalLoopingCall` 의 콜백에서 예외가 밖으로 나가면 그 루프는 종료되고, 프로세스를 재시작하기 전까지 다시 돌지 않는다.

## 왜 그렇게 동작하나

`oslo_service.loopingcall` 의 `_run_loop` 는 `LoopingCallDone` 만 정상 종료로 취급한다. 그 외 예외는 로그를 남기고 **다시 던지며**, 예외가 greenthread 밖으로 빠져나가는 순간 그 루프는 끝난다. 재시도도, 자동 재기동도 없다.

```python
loop = loopingcall.FixedIntervalLoopingCall(self._do_work, ctx, item_id)
loop.start(60, initial_delay=delay)
self.loops[item_id] = loop
```

이런 구조에서 두 가지가 겹치면 복구 불가능한 상태가 된다.

- **재등록 조건이 컨테이너 존재 여부로만 판정된다.** `if item_id not in self.loops:` 로 새 루프를 만드는 코드가 흔한데, 죽은 루프 객체는 dict 에 그대로 남아 있다. 조건이 계속 거짓이라 루프는 영영 재생성되지 않는다. 객체가 살아 있는지는 아무도 확인하지 않는다.
- **콜백 안에서 동기 RPC(`.call()`)를 호출한다.** 브로커나 서버가 한 번만 느려도 `MessagingTimeout` 이 나고, 그게 곧바로 루프를 죽인다. `except` 절이 `requests.exceptions.ConnectionError` 처럼 특정 네트워크 예외로만 좁혀져 있으면 이 예외는 걸리지 않는다.

루프는 **항목별로 독립**이므로, 죽는 것도 그 순간 예외를 맞은 항목뿐이다. 같은 프로세스의 다른 항목들은 멀쩡히 계속 돈다. 그래서 "서비스가 죽었다"는 신호가 전혀 나타나지 않는다.

## 확인하는 방법

로그에 실패 흔적이 남는다. 프로세스는 살아 있으므로 systemd 상태로는 알 수 없다.

```
$ grep "Fixed interval looping call" <logfile>
... ERROR oslo.service.loopingcall [-] Fixed interval looping call
    '<module>.<Class>._do_work' failed: oslo_messaging.exceptions.MessagingTimeout:
    Timed out waiting for a reply to message ID ...
```

로그가 이미 로테이트돼 사라졌다면 **산출물로 판정**한다. 입력은 계속 쌓이는데 출력만 특정 시각부터 끊긴 항목을 찾는 방식이다.

```
$ for d in <basedir>/*/data; do
    p=${d%/data}
    [ $(stat -c %Y $d) -gt $(($(stat -c %Y $p)+3600)) ] && echo STUCK $p
  done
```

- 부모 디렉터리 mtime = 마지막으로 처리(집계·정리)된 시각, 하위 `data/` mtime = 마지막으로 수집된 시각.
- 둘의 간격이 크게 벌어져 있으면 "수집은 되는데 처리만 멈춘" 상태다.
- ⚠️ 이 방식은 **현재 진행형으로 멈춘 것만** 잡는다. 과거에 멈췄다가 재시작으로 이미 복구된 이력은 드러나지 않는다.
- ⚠️ 오래전에 버려진 잔재 디렉터리도 같이 걸린다. `data/` 의 mtime 도 함께 확인해서, 수집 자체가 멈춘 지 오래된 것은 제외해야 한다.

## 함정

- ⚠️ **예외를 잡되 DEBUG 로만 남기면 루프는 살아도 장애는 여전히 안 보인다.** 주기 작업의 실패는 ERROR 로 남겨야 운영 로그(INFO 레벨)에 흔적이 남는다. 루프를 살리는 것(`except`)과 흔적을 남기는 것(로그 레벨)은 별개의 결정이다.
- ⚠️ **상태 플래그를 원격에 보고하는 구조라면 마지막 값이 그대로 굳는다.** 정상(`ACTIVE`)으로 보고한 직후 루프가 죽으면, API 나 대시보드에는 계속 정상으로 보인다. 사용자도 운영자도 알 방법이 없는 무증상 장애가 된다.
- ⚠️ **타임아웃을 피하려고 `.call()` 을 `.cast()` 로 바꾸는 것은 대개 잘못된 처방이다.** 갱신 실패를 감지할 수 없게 되고, 어차피 다음 주기에 같은 값을 다시 보내므로 얻는 것이 없다. 💡 문제는 호출 방식이 아니라 예외가 루프를 죽인다는 것이므로, 고칠 지점은 `except` 범위다.
- **프로세스 재시작으로 저절로 복구되기 때문에 원인 파악이 늦어진다.** 배포·재기동이 잦은 환경에서는 증상이 우연히 사라졌다가 재발한다. 실제로 하루 전에 같은 예외가 났다가 다음 날 배포 재시작으로 조용히 복구된 뒤, 그 다음 발생 때 비로소 드러난 사례가 있다.
- 💡 재등록 로직을 추가하는 것보다 예외를 콜백 안에서 막는 편이 낫다. 루프가 죽지 않으면 죽은 객체가 남는 상태 자체가 생기지 않아, 재등록 코드는 실행되지 않는 죽은 코드가 된다.

## 관련

- [eventlet greenthread](../python/eventlet-greenthread.md)
