# 롤백 중 터진 2차 예외가 원인 예외를 덮어쓴다

> `except: cleanup(); raise` 구조에서 `cleanup()` 이 새 예외를 던지면 원래 원인은 사라진다. 클라이언트는 자기가 요청하지도 않은 리소스를 지목하는 에러를 받는다.

## 왜 그렇게 동작하나

OpenStack 서비스 플러그인에서 흔한 롤백 관용구다.

```python
try:
    self._activate(context, obj)      # ① 여기서 원인 예외가 난다
except Exception:
    LOG.error("Fail to activate %s, rolls back", obj.id, exc_info=True)
    self._deactivate(context, obj)    # ② 여기서 또 예외가 나면
    raise                             # ③ 이 raise 에는 도달하지 못한다
```

`raise` 는 인자가 없으면 **처리 중인 예외를 다시 던진다**. 그런데 ②가 새 예외를 던지면 `except` 블록이 그 예외로 빠져나가고, ③은 실행되지 않는다. Python 2 에는 예외 연결(`__context__` 체인)이 없어 원인 예외는 로그에만 남고 호출자에게는 전달되지 않는다. Python 3 라면 체인은 남지만, API 레이어는 **맨 마지막 예외 하나만** HTTP 상태 코드로 매핑하므로 클라이언트가 받는 것은 여전히 2차 예외다.

정리 작업이 원인보다 더 시끄러운 예외를 던지기 쉬운 이유가 있다. 롤백은 **방금 만든 것을 지우는** 일인데, 삭제는 "아직 누가 쓰고 있으면 거부"라는 검사를 통과해야 한다. 실패한 요청과 **동시에 진행된 다른 요청**이 그 리소스를 붙잡고 있으면, 롤백은 정상 동작으로서 거부당한다. 즉 2차 예외는 버그가 아니라 경합의 흔적이다.

실제 사례에서는 이런 모습이었다.

```
23:08:24 ERROR ... Fail to activate <child-obj> <id>, rolls back
         <Custom>IpAddressGenerationFailure: No more IP addresses available
         on subnets [<external-subnet-a>, <external-subnet-b>]      ← 진짜 원인
23:08:33 ERROR ... Fail to activate all ... deactivate subnets and <parent> <id>
         SubnetInUse: Unable to complete operation on subnet <subnet-b>:
         One or more ports have an IP allocation from this subnet.  ← 클라이언트가 본 것
```

같은 로그 파일에서 클라이언트로 나간 예외를 세어보면 비율이 드러난다. 원인 예외 33건, 2차 예외 7건 — **대부분은 원인이 그대로 보이고, 하필 다른 작업과 겹친 소수만 둔갑한다.** 그래서 재현이 산발적이고, 리포트에는 오해를 부르는 쪽만 올라온다.

## 확인하는 방법

**① 에러 메시지가 지목한 리소스가 요청에 없으면 2차 예외를 의심한다.** 요청 본문에 없는 ID 가 응답에 나오는 것은 그 자체로 강한 신호다. 정상적인 검증 실패는 요청에 있는 값을 지목한다.

**② 응답의 요청 ID 로 서버 쪽 실제 API 를 특정한다.** 클라이언트 로그가 `POST /foo` 실패로 기록해도, 응답에 실린 서버 요청 ID 는 다른 API 의 것일 수 있다(앞단 게이트웨이가 내부 호출의 에러를 그대로 중계하는 경우).

```
$ zgrep -h <server-request-id> <logdir>/*.log* | tail -30
... [TIMER] Completed GET /networks in 19.9483 sec        ← POST 가 아니었다
```

**③ 그 요청 ID 의 로그를 시간순으로 훑어 첫 ERROR 를 찾는다.** 마지막 ERROR 가 아니라 **첫 ERROR** 가 원인이다. 롤백 로그(`rolls back`, `deactivate`, `cleanup`)가 그 사이에 끼어 있으면 마스킹이 일어난 구간이다.

**④ 예외별 발생 건수를 비교해 어느 쪽이 원인인지 가늠한다.**

```
$ zgrep -hc "Execution Error: <원인예외클래스>" <logdir>/<log>.1.gz
$ zgrep -hc "Execution Error: <2차예외클래스>"  <logdir>/<log>.1.gz
```

원인 예외 쪽이 압도적으로 많으면, 2차 예외는 특수 조건(경합)에서만 나타나는 표면이라는 뜻이다.

## 함정

- ⚠️ **2차 예외를 그대로 버그 제목으로 삼으면 엉뚱한 컴포넌트가 조사 대상이 된다.** 위 사례에서 리포트는 "로드밸런서 생성이 `SubnetInUse` 로 실패"였지만, 로드밸런서 코드에는 서브넷을 삭제하는 경로가 아예 없었다. 💡 의심 컴포넌트의 코드에서 **그 예외를 던질 수 있는 지점을 전수 확인**(`grep -rn <ExceptionClass>`)해 0건이면, 그 컴포넌트는 용의자에서 빠진다.
- ⚠️ **재현 조건에 경합이 포함돼 있으면 "간헐적"으로 보인다.** 동시에 다른 리소스를 만들지 않으면 롤백이 조용히 성공해 원인 예외가 정직하게 노출된다. 같은 코드가 다른 환경에서 통과하는 것도 마찬가지 이유일 수 있다 — 코드 차이가 아니라 자원 여유·타이밍 차이다.
- ⚠️ **`exc_info=True` 로 남긴 로그가 있다고 안심할 수 없다.** 로그에는 원인이 있지만 클라이언트·상위 서비스·티켓에는 2차 예외만 전파된다. 사용자 관점의 증상과 서버 로그의 첫 ERROR 를 항상 따로 확인해야 한다.
- 💡 고칠 지점은 `_deactivate()` 를 `try/except` 로 감싸 정리 실패를 로그로만 남기고 원인 예외를 `raise` 로 되살리는 것이다. 정리 실패를 삼키는 게 불편하면 원인 예외를 명시적으로 감싸 던진다(`raise Wrapped(original) from cleanup_error` 형태). 정리 실패가 원인을 대체하게 두는 것만 피하면 된다.

## 관련

- [지연 실체화 — 조회 요청이 쓰기를 유발한다](lazy-activation-on-read.md)
- [백엔드가 지원해도 앞단 API가 막는다](api-layer-resource-whitelist.md)
