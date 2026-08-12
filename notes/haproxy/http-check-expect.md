# `http-check expect` — 헬스체크 합격 조건

> `option httpchk` 는 **무엇을 요청할지**, `http-check expect` 는 **무엇을 합격으로 볼지** 정한다.
> `expect` 를 안 쓰면 **2xx·3xx 전부 합격**이고, 쓰는 순간 거기 적힌 것만 합격이 된다.
> 그래서 백엔드가 정상 응답 코드를 바꾸면 이 조건이 **정상을 거부**한다.

## 왜 그렇게 동작하나

두 줄은 역할이 다르다.

```
option httpchk GET /healthcheck HTTP/1.0\r\nHost:\ <host>   ← 요청: 경로·HTTP 버전·헤더
http-check expect rstatus ^(200|300)$                        ← 판정: 무엇을 합격으로 볼지
```

### 매처 종류

| 매처 | 검사 대상 | 예 |
| --- | --- | --- |
| `status <code>` | 상태코드 완전일치 (단일값) | `status 200` |
| `rstatus <정규식>` | 상태코드를 **정규식**으로 (`r` = regex) | `rstatus ^(200\|300)$` |
| `string <문자열>` | 응답 **본문**에 문자열 포함 | `string OK` |
| `rstring <정규식>` | 응답 **본문**을 정규식으로 | `rstring ^OK` |
| `!` 접두사 | 부정 | `! rstatus ^5` |

- **`expect` 줄이 없으면 기본 판정 = 2xx·3xx 합격, 4xx·5xx 불합격.** 그래서 버전 목록으로 `300` 을 주는
  루트 경로(`GET /`)를 체크에 쓰면 `expect` 없이도 통과한다.
- 여러 코드를 허용하려면 `rstatus` 정규식이 확실하다. `status` 에 콤마 목록(`status 200,300`)을 적는 형태는
  버전에 따라 지원이 갈리고, ⚠️ **문법 검증(`haproxy -c`)이 통과해도 실제 매칭이 되는지는 별개다**
  (1.8 계열에서 `-c` 만 통과하는 것을 확인, 실동작은 미확인).

### 상태를 읽는 값

| `check_status` | 뜻 |
| --- | --- |
| `L7OK` | L7 체크 합격 |
| `L7STS` | 응답은 왔는데 **코드가 조건 불일치** (로그: `Layer7 wrong status, code: N`) |
| `L4CON` | TCP 연결 자체 실패 (`Connection refused`) |
| `L7TOUT` | L7 응답 타임아웃 |

`act=N bck=M` 은 살아 있는 active 서버 N대, backup 서버 M대를 뜻한다.
active 가 0이 되면 backup 이 대신 받고 로그에 `Running on backup` 이 찍힌다.
전부 죽으면 `proxy <name> has no server available!` 이 남고 클라이언트는 **503 —
`No server is available to handle this request.`** 를 받는다. 이 문구가 보이면 앞단 LB 판정 문제이지
백엔드 애플리케이션 장애가 아닐 수 있다.

### backup 서버가 다른 코드를 준다면

같은 서비스라도 구세대(VM) 백엔드와 신세대(컨테이너) 백엔드가 **미등록 경로에 서로 다른 코드**를 줄 수 있다.
앞단 웹서버 설정 차이 때문이다 — 한쪽은 rewrite 로 루트와 동일하게 처리해 `300`, 다른 쪽은 그대로 `404`.

이때 판정 조건은 **양쪽을 다 커버**해야 한다. 한쪽만 맞추면 나머지가 조용히 빠지고,
그게 마지막 안전망(backup)이면 그대로 전면 장애가 된다.

## 확인하는 방법

**① 체크를 손으로 그대로 재현** — LB 가 보내는 것과 동일하게 (HTTP 버전·Host 헤더 포함):

```bash
curl -sS -o /dev/null -D- -m 5 --http1.0 -H 'Host: <host>' \
  "http://<backend-ip>:<port>/healthcheck"
```

**② 실패 사유는 로그가 가장 명확하다.**

```bash
grep -a "<backend-name>/" /var/log/haproxy.log \
  | grep -aE "is DOWN|is UP|no server available"
```

```
Server <be>/<srv> is DOWN, reason: Layer7 wrong status, code: 200,
    info: "HTTP status check returned code <3C>200<3E>", ...
Backup Server <be>/<srv> is UP, reason: Layer7 check passed, code: 300, ... Running on backup.
```

**③ 현재 상태는 stats 소켓으로 본다.** `socat` 이 없는 노드에서는 python3 로 충분하다:

```bash
python3 - <<'PY'
import socket, glob
for path in sorted(glob.glob("/var/run/haproxy_*.sock")):
    s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM); s.settimeout(5); s.connect(path)
    s.sendall(b"show stat\n")
    buf = b""
    while True:
        d = s.recv(65536)
        if not d: break
        buf += d
    lines = buf.decode("utf-8", "replace").splitlines()
    hdr = [h.strip("# ") for h in lines[0].split(",")]
    want = ["svname", "status", "act", "bck", "chkfail", "chkdown", "check_status", "check_code"]
    print("==", path)
    for ln in lines[1:]:
        c = ln.split(",")
        if len(c) < 5 or c[0] != "<backend-name>": continue
        print("  ", " ".join("%s=%s" % (n, c[hdr.index(n)]) for n in want if n in hdr))
PY
```

- 필드는 **번호가 아니라 헤더 이름으로 찾는다** — 버전마다 열이 늘어난다.
- ⚠️ 프로세스를 여러 개 띄우는 구성(`nbproc`)에서는 **프로세스마다 독립적으로 체크하고 소켓도 따로**다.
  한 소켓만 보고 판단하지 말 것.

**④ 문법만 확인**: `haproxy -c -f <cfg>` → `Configuration file is valid`

**⑤ 비침습 실험** — 운영 인스턴스를 건드리지 않고 조건을 바꿔가며 판정을 관찰할 수 있다.
별도 설정 파일로 `127.0.0.1` 고포트에 두 번째 인스턴스를 띄운다:

```bash
haproxy -f /tmp/try.cfg -D -p /tmp/try.pid     # 종료: kill $(cat /tmp/try.pid)
```

⚠️ 이때 프런트엔드로 curl 하면 Host 헤더가 `127.0.0.1:<port>` 가 된다. 호스트 기반 라우팅을 하는
백엔드(ingress·gateway)는 이 요청에 404 를 준다. 그래서 **503 인지 아닌지로만 UP/DOWN 을 판정**하고,
그 밖의 코드는 "백엔드는 UP" 으로 읽어야 한다. (이걸 헷갈려서 UP 을 DOWN 으로 오독한 적이 있다.)

## 함정

- ⚠️ **조건을 좁게 박으면 백엔드가 "더 정상적인" 응답을 하게 됐을 때 오히려 DOWN 이 된다.**
  헬스 엔드포인트를 새로 붙여 200 을 돌려주게 만드는 변경이 대표적이다.
  기존 조건이 `^(300|404)$` 였다면 그 개선이 곧 장애다.
- ⚠️ **판정 조건에 특정 코드가 들어 있는 이유를 모르면 지우지 말 것.**
  그 코드는 다른 백엔드(구세대·backup)가 주는 정상 응답일 수 있다.
- ⚠️ 리로드하면 체크 카운터(`chkfail`/`chkdown`)가 0으로 초기화된다.
  "실패 이력 0" 이 곧 "한 번도 안 죽었다" 는 뜻은 아니다.
- ⚠️ 응답 헤더의 `Server:` 로 응답 주체를 판별할 때, 중간 프록시가 업스트림 헤더를 **보존**할 수 있다.
  `Server: Apache` 가 보였다고 프록시를 안 거친 것은 아니다.
- 💡 헬스체크 경로를 바꿀 때는 판정 조건도 같이 본다. 경로만 바꾸고 조건을 두면
  "조건이 옛 경로의 응답에 맞춰진" 상태가 남는다.

## 관련

- [probe 3종과 기본값 함정](../kubernetes/probes.md) — 같은 엔드포인트를 kubelet 이 볼 때의 기준
- [Paste urlmap — 경로 접두사 라우팅](../wsgi/paste-urlmap.md) — 백엔드가 `300`/`404`/`200` 을 주는 이유
- [state 와 pillar](../salt/state-vs-pillar.md) — 이 설정 파일의 정본이 어디인지
