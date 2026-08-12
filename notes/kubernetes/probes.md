# probe 3종과 기본값 함정

> probe는 kubelet이 컨테이너를 주기적으로 찔러보는 건강검진이다.
> 종류마다 **실패했을 때 쿠버네티스가 취하는 조치가 다르고**, `timeoutSeconds`를 안 적으면 기본 1초다.

## 왜 그렇게 동작하나

| 종류 | 묻는 것 | 실패하면 |
| --- | --- | --- |
| `startupProbe` | 아직 뜨는 중인가, 다 떴나? | 성공할 때까지 나머지 두 probe를 **보류**. 한도를 넘기면 컨테이너 재시작 |
| `livenessProbe` | 살아 있나? | 컨테이너를 **재시작** |
| `readinessProbe` | 트래픽 받을 준비가 됐나? | 서비스 엔드포인트에서 **빼기만** 함 (재시작 없음, 회복하면 자동 복귀) |

검사 방법은 `httpGet`(경로에 GET 요청), `tcpSocket`(포트 열림), `exec`(컨테이너 안에서 명령) 중 선택.

**`httpGet`은 2xx와 3xx를 성공으로 본다.** 3xx도 성공이지만 kubelet은 리다이렉트를 따라가지 않고
`ProbeWarning` 이벤트에 응답 본문을 남긴다 → 로그가 시끄러워진다.

### 주요 옵션과 기본값

| 옵션 | 뜻 | 기본값 |
| --- | --- | --- |
| `periodSeconds` | 검사 주기 | 10초 |
| `timeoutSeconds` | 응답 대기 한도 | **1초** |
| `failureThreshold` | 몇 번 연속 실패하면 조치할지 | 3 |
| `initialDelaySeconds` | 첫 검사까지 대기 | 0 |

mod_wsgi로 OpenStack API를 띄우는 차트에서 쓰던 설정 예시:

```yaml
startupProbe:   { path: /healthcheck, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 18 }
livenessProbe:  { path: /healthcheck, periodSeconds: 20, timeoutSeconds: 3 }
readinessProbe: { path: /healthcheck, periodSeconds: 10, timeoutSeconds: 3 }
```

- startup은 10초 × 18회 = **최대 180초** 기동 여유. mod_wsgi가 첫 요청에 앱을 로드하느라 느린 것을 감안한 값이다.
  startupProbe가 있으면 `initialDelaySeconds`는 따로 필요 없다.
- liveness는 20초 주기 × 3회 실패(기본값) = 약 60초 후 재시작.

## 실제 사례 — 경고 소음에 진짜 실패가 묻힌 경우

OpenStack API 컨테이너에서 세 probe가 모두 루트 `/`를 때리고 있었다. 그 경로의 앱은 API 버전 목록을
`300 Multiple Choices`로 응답한다(OpenStack 관례).

```
ProbeWarning  Readiness probe warning: Probe terminated redirects, Response body: {"versions": …}
Unhealthy     Readiness probe failed: Get "http://<pod-ip>:<port>/": context deadline exceeded
```

1. 300은 성공으로 집계되지만 `ProbeWarning`이 계속 쌓인다 → 로그 소음
2. 그 소음에 두 번째 줄의 **실제 실패**가 묻혔다. `readinessProbe`에 `timeoutSeconds`가 없어 기본 1초였고,
   WSGI 워커가 1개인 데다 CPU 요청이 낮아 잠깐 바쁘면 1초를 넘긴다 → 파드가 순간적으로 서비스에서 빠짐
3. 조치: probe 경로를 200을 주는 `/healthcheck`로 바꾸고, readiness에 `timeoutSeconds: 3` 추가
4. ⚠️ 근본 원인(워커 수 / CPU 요청 부족)은 남는다. timeout 완화는 증상 완화다

## 함정

- `timeoutSeconds` 미지정 = 1초. 워커 수가 적은 WSGI API에서는 이것만으로 간헐적 not-ready가 생긴다.
- 3xx는 실패가 아니다. "probe가 실패한다"와 "ProbeWarning이 쌓인다"는 다른 문제이므로 원인을 섞지 말 것.
- probe 경로를 얕은 healthcheck로 바꿔도 앱 자체가 깨진 경우는 잡힌다
  (urlmap이 앱을 eager 구성하므로 → [Paste urlmap](../wsgi/paste-urlmap.md)). 단 DB 장애는 잡지 못한다.
- Helm 차트에서 probe 기본값을 바꿀 때는 **환경별 values가 probe를 override하고 있지 않은지** 먼저 확인해야 한다.
  override 중인 환경은 기본값 변경이 먹지 않는다.

  ```bash
  grep -rn "Probe" values/*/<chart>/ charts/<chart>/values.*.yaml
  ```
