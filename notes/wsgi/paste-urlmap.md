# Paste urlmap — 경로 접두사 라우팅

> `[composite:main] use = egg:Paste#urlmap`은 요청 경로의 앞부분을 보고 어느 앱으로 보낼지 정하는 라우터다.
> **최장 접두사가 이기고**, `/`는 아무것도 안 맞았을 때만 쓰이는 최후 수단이다.
> 그리고 표에 적힌 앱들을 **요청 전에 전부 만들어 둔다**(eager).

## 왜 그렇게 동작하나

```ini
[composite:main]
use = egg:Paste#urlmap
/: version_app               ← catch-all
/v1: api-keystone
/healthcheck: healthcheck
```

분기 결과:

| 요청 | 매칭 | 처리 | 응답 |
| --- | --- | --- | --- |
| `GET /v1/<resource>/<id>` | `/v1` | keystone_authtoken → context → apiapp | 200 |
| `GET /healthcheck` | `/healthcheck` | healthcheck 앱 | 200 |
| `GET /` | catch-all | version_app | **300** |
| `GET /nonexistent` | catch-all | version_app | 404 |

- 모든 경로가 `/`로 시작하지만 `/`는 길이 0의 접두사이므로 **더 긴 규칙이 먼저 잡는다.**
  따라서 `/healthcheck` 한 줄을 추가해도 `/v1`이나 `/`의 동작은 바뀌지 않는다.
- 매칭된 접두사는 잘려서 `SCRIPT_NAME`으로 옮겨지고, 앱은 나머지를 `PATH_INFO`로 받는다.
  (`/v1/secrets/abc` → 앱에는 `/secrets/abc`로 전달)
- **`/`가 300을 주는 이유**: catch-all에 붙는 앱은 사용 가능한 API 버전 목록을 돌려주는 앱이고,
  OpenStack 관례상 버전 목록 응답이 `300 Multiple Choices`("선택지가 여럿이다")다.

### eager 구성

urlmap은 표에 적힌 앱들을 **설정을 읽는 시점에 전부 생성한다**(반대는 lazy = 첫 요청 때 생성).

그래서 `/healthcheck` 요청 하나만 들어와도 composite 전체(인증 미들웨어 + 메인 앱까지)의
생성이 강제된다. 실질적 의미:

- import 오류·설정 오류로 앱이 깨진 경우 → 500이 되어 probe가 **잡는다**
- DB가 죽은 경우 → 앱 생성 자체는 성공하므로 **못 잡는다**

## 확인하는 방법

**① eager인지 확인** — 존재하지 않는 factory를 매핑해두고 요청 없이 로드만 해본다:

```bash
cat > /tmp/eager.ini <<EOF
[composite:main]
use = egg:Paste#urlmap
/healthcheck: healthcheck
/bad: broken

[app:healthcheck]
use = egg:oslo.middleware#healthcheck
oslo_config_project = <svc>

[app:broken]
paste.app_factory = <svc>.api.app:no_such_factory
EOF
python2 -c "
from paste.deploy import loadapp
try:
    loadapp('config:/tmp/eager.ini', name='main')
    print('lazy')
except Exception as e:
    print('eager ->', type(e).__name__, str(e)[:80])
"
```

요청을 보내지 않았는데 로드만으로 터진다 → eager:

```
eager -> ImportError 'module' object has no attribute 'no_such_factory'
```

**② 어떤 파이프라인을 탔는지는 로그로 판별된다.** `/v1`을 탔다면 filter들이 남긴 흔적이 붙는다:

```
INFO <svc>.api.controllers.<resource> [req-… <user-id> - - default default] …
INFO <svc>.api.middleware.context […] Processed request: 200 OK - GET …/v1/<resource>/<id>
```

대괄호 안 사용자·프로젝트 ID는 `keystone_authtoken`이 채운 값이고, `middleware.context` 로그는
`context` filter를 거쳤다는 증거다. catch-all로 갔다면 인증 없이 버전 목록만 돌려주고 끝난다.

## 함정

- ⚠️ **매핑을 추가하기 전에 그 경로로 요청하면 404가 정상이다.** catch-all의 버전 앱이 모르는 경로이기 때문.
  "설정을 넣기 전에 이미 200이 나왔다"는 검증 결과가 있다면 그 검증이 잘못된 것이다
  (실제로 리뷰에서 이 불일치를 발견한 적이 있다 — 같은 이미지에서 `GET /healthcheck` → 404, `GET /` → 300).
- ⚠️ **같은 패키지를 써도 배포 형태에 따라 미등록 경로의 응답이 갈린다.** 앞단 웹서버(Apache 등) 설정에
  rewrite 가 있으면 `/healthcheck` 가 루트와 동일하게 처리되어 **300**, 없으면 catch-all 앱이 받아 **404** 다.
  실제로 같은 deb 패키지를 쓰는 VM 배포본과 컨테이너 배포본이 각각 300 / 404 를 반환하는 것을 확인했다
  (VM 쪽은 `/healthcheck` 와 `/` 의 응답이 Content-Length 까지 동일 → rewrite 의 흔적).
  그래서 "이 이미지는 404 를 준다" 는 사실을 다른 환경에 그대로 옮겨 쓰면 안 된다.
- 접두사 뒤에 슬래시가 붙어도(`/healthcheck/`) 같은 앱으로 간다.
- 💡 새 경로를 추가하면 ingress가 `path: /`로 넘기는 구성에서는 서비스 도메인으로도 도달 가능해진다.
  진단 정보를 노출하는 옵션(healthcheck의 `detailed`)은 켜지 않는 편이 안전하다.

## 관련

- [paste.ini 섹션 구조](pastedeploy-sections.md)
- [probe 3종과 기본값 함정](../kubernetes/probes.md)
- [`http-check expect` — 헬스체크 합격 조건](../haproxy/http-check-expect.md) — 앞단 LB 가 이 응답 코드를 어떻게 판정하는지
