# paste.ini 섹션 구조 — app / filter / pipeline / composite

> OpenStack API 서비스는 `*-paste.ini`라는 조립 설명서로 "어떤 URL을 어떤 파이썬 객체가 처리할지" 정한다.
> `app`은 요청을 끝까지 처리하는 최종 처리자, `filter`는 그 앞에서 요청을 가로채는 층이다.

## 왜 그렇게 동작하나

OpenStack API는 파이썬 WSGI 앱이고, Apache(mod_wsgi)가 `app.wsgi`를 로드하면 그 안에서
**PasteDeploy**가 `/etc/<svc>/<svc>-api-paste.ini`를 읽어 앱을 조립한다.

섹션은 네 종류다.

| 섹션 | 역할 |
| --- | --- |
| `[app:NAME]` | 요청을 끝까지 처리해 응답을 만드는 최종 처리자. 다음 대상이 없다 |
| `[filter:NAME]` | app을 감싸 요청/응답을 가공하거나 중간에서 가로챈다. 반드시 다음 대상을 품는다 |
| `[pipeline:NAME]` | filter 여러 개 + 마지막 app을 순서대로 묶은 것 |
| `[composite:NAME]` | 여러 app/pipeline을 조합한다. 조합 방식은 `use`로 정하고, 경로 분기는 보통 urlmap |

전형적인 구성:

```ini
[pipeline:api-keystone]
pipeline = keystone_authtoken context apiapp
#          └── filter ──┘      └filter┘ └ app ┘

[filter:keystone_authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory

[app:apiapp]
paste.app_factory = <svc>.api.app:create_main_app
```

`GET /v1/<resource>`가 흐르는 순서:

```
요청 → [keystone_authtoken]  토큰 검증
          ├ 불량 → 여기서 401 응답 (apiapp 은 호출되지 않는다)
          └ 정상 → 통과
     → [context]  요청에 사용자/프로젝트 정보 부착
     → [apiapp]   실제 처리 → 응답
```

### 파이썬 함수를 지목하는 두 가지 방법

| 방식 | 예시 | 의미 |
| --- | --- | --- |
| 직접 지목 | `paste.app_factory = <svc>.api.app:create_main_app` | 이 모듈의 이 함수를 호출해 앱을 만들어라 |
| 엔트리포인트 참조 | `use = egg:oslo.middleware#healthcheck` | 그 패키지가 미리 등록해둔 이름을 찾아 호출해라 |

`paste.app_factory` / `paste.filter_factory`는 **역할(그룹) 이름**이다. 패키지는 설치 시
`entry_points.txt`에 그룹별로 이름→함수 매핑을 등록해둔다. oslo.middleware의 실제 등록 내용:

```
[paste.app_factory]
healthcheck = oslo_middleware:Healthcheck.app_factory     ← [app:...] 로 쓸 때

[paste.filter_factory]
healthcheck = oslo_middleware:Healthcheck.factory         ← [filter:...] 로 쓸 때
```

같은 `healthcheck` 이름이 두 그룹에 있으므로, **어느 섹션에 쓰느냐가 동작을 바꾼다.**

- `[filter:healthcheck]` — 기존 앱 앞에 껴서 "내가 담당하는 경로면 내가 응답, 아니면 통과". 담당 경로를 설정으로 알려줘야 한다.
- `[app:healthcheck]` — 자기에게 도달한 요청은 무조건 헬스체크 응답. 경로 판단은 urlmap이 대신하므로 서비스 conf를 건드릴 필요가 없다.

## 확인하는 방법

**① 엔트리포인트가 그 그룹에 등록돼 있는지** (없으면 앱 로드가 실패해 전 경로 500):

```bash
sed -n '/paste/,/^$/p' /usr/lib/python2.7/dist-packages/oslo.middleware*.egg-info/entry_points.txt
```

**② 앱이 실제로 어떤 응답을 주는지** — 돌아가는 서비스를 건드리지 않고 임시 ini로 로드해 호출한다:

```bash
cat > /tmp/hc.ini <<EOF
[app:healthcheck]
use = egg:oslo.middleware#healthcheck
oslo_config_project = <svc>
EOF
python2 -c "
import webob
from paste.deploy import loadapp
app = loadapp('config:/tmp/hc.ini', name='healthcheck')
for m in ('GET', 'HEAD'):
    print(m, webob.Request.blank('/', method=m).get_response(app).status)
"
```

oslo.middleware 3.34.0 기준 실행 결과:

```
GET  200 OK        (본문 비어 있음)
HEAD 204 No Content
```

⚠️ mod_wsgi는 보통 서비스 전용 계정으로 앱을 돌린다(`WSGIDaemonProcess user=<svc>`).
설정 파일 읽기 권한까지 확인하려면 `su -s /bin/sh <svc> -c "..."`로 같은 계정에서 실행해봐야 한다.

## 함정

- `oslo.middleware#healthcheck`를 `[app:]`에 쓰려면 **`paste.app_factory` 그룹**에 등록돼 있어야 한다.
  오래된 버전에 그룹이 없으면 요청 시점이 아니라 **앱 로드 시점**에 깨져 전 경로가 500이 된다.
- `oslo_config_project = <svc>`는 "설정을 읽을 때 이 프로젝트로 취급하라"는 뜻이다
  (= `/etc/<svc>/<svc>.conf`의 `[healthcheck]` 섹션을 찾는다). 섹션이 없으면 기본값으로 동작한다.
- healthcheck에 `backends`를 주지 않으면 검사 항목이 없어 **항상 200**이다. "프로세스가 응답할 수 있다"까지만 보증한다.
