# state 와 pillar — 로직과 데이터의 분리

> Salt 설정은 두 층이다. **state(`salt/` 아래 `.sls`)가 "무엇을 어떻게 할지"**,
> **pillar(`pillar/` 아래 `.sls`)가 "어떤 값으로 할지"**.
> 그리고 `file.managed` 로 관리되는 파일은 **노드에서 손으로 고쳐도 다음 적용 때 되돌아간다.**

## 왜 그렇게 동작하나

state 가 파일을 선언하고, pillar 의 값을 템플릿에 주입한다.

```jinja
# salt/<svc>/deploy/config.sls  — 로직
/etc/<svc>/<svc>.cfg:
  file.managed:
    - source: salt://<svc>/files/<svc>.cfg.tmpl
    - template: jinja
    - context:
        items: {{ pillar['<svc>']['items'] }}        ← pillar 데이터 주입
```

```jinja
# salt/<svc>/files/<svc>.cfg.tmpl  — 렌더링
{%- for m in item['monitor'] %}
    {{ m }}
{%- endfor %}
```

```yaml
# pillar/<env>/<svc>.sls  — 데이터
<svc>:
  items:
    - name: <backend>
      monitor:
        - option httpchk GET /healthcheck HTTP/1.0
        - http-check expect rstatus ^(200|300)$
```

→ 노드의 `/etc/<svc>/<svc>.cfg` 에 그 두 줄이 그대로 찍힌다.
즉 **노드에서 본 한 줄이 pillar 의 어느 줄에서 나왔는지 1:1로 역추적할 수 있다.**
"이 이상한 설정은 대체 어디서 온 거지?" 를 푸는 출발점이 이 대응 관계다.

- pillar 는 대상(minion)별로 다른 값을 안전하게 내려주는 데이터 저장소다. 비밀값도 여기 둔다.
- state 는 여러 환경이 공유하고, 환경 차이는 pillar 로만 낸다 — 그래서 환경별 동작 차이를 볼 때는
  state 가 아니라 **pillar 를 비교**해야 한다.
- ⚠️ **`file.managed` = 그 파일의 정본은 저장소다.** 긴급 조치로 노드를 직접 편집하면
  다음 `state.apply` 에 원복된다. 영구 반영은 pillar·템플릿을 고쳐야 한다.
- pillar 를 gitfs 로 서빙하는 구성에서는 마스터의 로컬 pillar 디렉터리를 편집해도 반영되지 않는다.
  저장소가 정본이다.

## 확인하는 방법

**① 노드의 설정 한 줄 → 저장소 역추적**

1. 저장소에서 그 문자열을 grep → 어느 pillar 파일인지 찾는다
2. 같은 키를 참조하는 state(`context:` / `pillar[...]`)를 찾는다
3. state 의 `source:` 로 템플릿을 열어 렌더링 규칙을 확인한다

**② 적용 전에 차이만 본다** — 전체 적용은 피하고 대상 항목만:

```bash
salt '<minion>' state.sls_id <state-id> <state-file> test=True
```

⚠️ 전체 `state.apply` 는 의도치 않은 항목까지 "정본대로" 되돌린다.
수동 운영 흔적이 있는 노드에서 특히 위험하다.

## 함정

- ⚠️ **기본 브랜치가 정본이 아닐 수 있다.** GitOps·브랜치 전략에 따라 실제 배포 대상이
  `develop` 같은 별도 브랜치인 경우가 있다. 기본 브랜치만 읽고 "노드와 저장소가 다르다" 고
  결론 내면 오진이다.
  (실제로 그렇게 잘못 판단해 "누가 노드를 손으로 고쳤다" 고 지적하려던 적이 있다 —
  기본 브랜치가 구버전이고 배포 브랜치가 최신이었다. 지적 전에 배포 브랜치를 확인해서 겨우 막았다.)
- ⚠️ 저장소 커밋 작성자가 사람이 아니라 **마스터 노드 계정**인 경우가 있다(노드에서 직접 커밋).
  이러면 커밋 메시지만으로 변경 의도를 알기 어렵고, 리뷰도 없이 들어갔을 수 있다.
- ⚠️ 같은 서비스의 pillar 가 환경별로 제각각인 것이 정상은 아니다. 한 환경에만 남은 조건은
  **아직 적용되지 않은 잠재 폭탄**일 수 있다 — 적용되는 순간 다른 환경에서 이미 겪은 장애가 재현된다.
- 💡 노드에서 급하게 고쳤다면 같은 내용을 저장소에도 반영해 둬야 원복 사고를 막는다.

## 관련

- [`http-check expect` — 헬스체크 합격 조건](../haproxy/http-check-expect.md) — 이 방식으로 관리되는 설정의 예
