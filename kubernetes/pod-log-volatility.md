# 파드 로그의 휘발성 — 재배포되면 사건이 사라진다

> 컨테이너 로그는 노드의 `/var/log/pods/<ns>_<pod>_<uid>/<container>/N.log` 에 쌓인다.
> 파드가 **다시 만들어지면 uid가 바뀌어 디렉터리째 새로 생기고**, 어제 로그를 볼 방법이 없어진다.

## 왜 그렇게 동작하나

경로에 파드 **uid**가 들어간다. 재시작(restart)과 재생성(recreate)은 다르다.

| 상황 | `RESTARTS` | 로그 |
| --- | --- | --- |
| 컨테이너만 재시작 (크래시·liveness 실패) | 증가 | 같은 디렉터리에 `0.log`, `1.log` … 로 쌓임 |
| 파드 재생성 (rollout·스케줄 변경) | **0** | uid가 바뀌어 **새 디렉터리**, 이전 것은 정리됨 |

그래서 `RESTARTS 0` 인데 `AGE`가 짧으면 **크래시가 아니라 배포가 있었다**는 뜻이다.
이 둘을 구분하지 못하면 "파드가 죽어서 로그가 없다"고 오진하게 된다.

로테이션도 별개로 돈다. kubelet의 `containerLogMaxSize`·`containerLogMaxFiles`(기본 10Mi × 5)를
넘으면 오래된 것부터 버려진다. 트래픽이 많은 API 파드는 몇 시간치만 남기도 한다.

## 확인하는 방법

파드가 언제 생겼고 재시작이 있었는지부터 본다.

```bash
kubectl get pods -n <ns> -o wide
```
```
NAME                    READY   STATUS    RESTARTS   AGE
<svc>-api-xxxx-aaaaa    1/1     Running   0          100m   ← 100분 전 재배포
<svc>-api-xxxx-bbbbb    1/1     Running   0          100m
```

여러 파드의 `AGE`가 **동시에** 짧고 `RESTARTS`가 전부 0이면 rollout이다.

노드에서 실제 로그 파일의 시간 범위를 본다.

```bash
ls -la --time-style=long-iso /var/log/pods/<ns>_<pod>_*/<container>/
# 0.log 만 있고 mtime이 오늘이면, 그 이전 기록은 없다
```

`kubectl logs --previous` 는 **같은 파드의 이전 컨테이너**만 보여준다.
파드가 재생성된 경우에는 쓸 수 없다.

## 함정

- ⚠️ **"로그가 없다"와 "에러가 없다"를 혼동하지 말 것.** 사건 시각 로그가 사라진 것을
  "그때 문제가 없었다"로 읽으면 조사가 통째로 어긋난다.
- 재현이 가능한 버그라면, 로그를 찾아 헤매는 것보다 **한 번 재현해서 새 로그를 만드는 편이 빠르다.**
  요청 단위 식별자(request id)를 응답에서 받아두면 곧바로 추적할 수 있다.
- 💡 애초에 중앙 로그 수집이 있으면 이 문제를 겪지 않는다. 없다면 **재현 우선** 전략이 현실적이다.
- 서비스가 여러 파드로 떠 있으면 요청은 그중 **한 곳**에만 기록된다. 전 파드를 훑어야 한다.

  ```bash
  # 어느 파드가 그 요청을 처리했는지부터 특정
  for n in <node-a> <node-b> <node-c>; do
    ssh $n "grep -c <req-id> /var/log/pods/<ns>_<pod>_*/<container>/*.log"
  done
  ```

## 관련

- [probe 3종과 기본값 함정](probes.md) — probe 실패로 인한 재시작은 `RESTARTS`가 증가한다
- [이미지 패키지 핀과 버전 스큐](../packaging/version-skew-in-image-pins.md)
