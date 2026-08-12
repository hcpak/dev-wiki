# dev-wiki

일하다 막혀서 파고들어 이해한 것들을 정리한 개인 위키.

무엇을 했는지가 아니라 **왜 그렇게 동작하는가**를 남긴다.
같은 계열의 문제를 다시 만났을 때 재사용하는 것이 목적이다.

[`notes/`](notes/) 에는 개념 하나를 짧게 정리한 배경지식 노트가 들어 있다.
대부분 공개된 오픈소스(Kubernetes, OpenStack, HAProxy, Salt, Paste/PasteDeploy, eventlet, pip 등)의
**문서에 잘 안 적혀 있거나, 적혀 있어도 실제로 데어봐야 이해되는 지점**을 다룬다.

## 노트

### kubernetes

| 문서 | 한 줄 |
| --- | --- |
| [probe 3종과 기본값 함정](notes/kubernetes/probes.md) | startup/liveness/readiness가 실패 시 하는 조치가 다르고, `timeoutSeconds` 기본값은 1초다 |
| [ConfigMap subPath 마운트와 checksum 애노테이션](notes/kubernetes/configmap-subpath-checksum.md) | subPath로 얹은 파일은 자동 갱신되지 않고, 파드 재생성으로만 반영된다 |
| [파드 로그의 휘발성](notes/kubernetes/pod-log-volatility.md) | 경로에 파드 uid가 들어가서, 재생성되면 어제 로그가 사라진다. `RESTARTS 0` + 짧은 `AGE` = 재배포 |

### wsgi (PasteDeploy)

| 문서 | 한 줄 |
| --- | --- |
| [paste.ini 섹션 구조 — app / filter / pipeline / composite](notes/wsgi/pastedeploy-sections.md) | URL별로 앱을 조립하는 방식과, filter가 app 앞에서 요청을 가로채는 구조 |
| [Paste urlmap — 경로 접두사 라우팅](notes/wsgi/paste-urlmap.md) | 최장 접두사 매칭, catch-all `/`, 그리고 매핑된 앱을 로드 시점에 전부 만드는(eager) 성질 |

### python

| 문서 | 한 줄 |
| --- | --- |
| [py2 유니코드 — Encode/Decode의 대칭](notes/python/py2-unicode-symmetry.md) | 템플릿이 bytes면 encode, unicode면 decode. 둘 다 ascii로 하기 때문에 터진다 |
| [eventlet 그린스레드](notes/python/eventlet-greenthread.md) | 협력형 스케줄링이라 표준 블로킹 호출 하나가 서버 전체를 멈춘다 |
| [pip `--upgrade` 는 의존성까지 건드린다](notes/python/pip-upgrade-dependency-resolution.md) | 설치는 성공했는데 임포트가 깨진다. 재시작 전에 임포트를 확인해야 한다 |

### prometheus

| 문서 | 한 줄 |
| --- | --- |
| [라벨 값 이스케이프 순서](notes/prometheus/label-value-escaping.md) | 역슬래시를 먼저 두 배로 하지 않으면 뒤 단계가 넣은 백슬래시까지 재이스케이프된다 |

### openstack

| 문서 | 한 줄 |
| --- | --- |
| [neutron `utils.execute()`의 인코딩 사고 경로](notes/openstack/neutron-utils-execute.md) | 명령 출력 bytes를 oslo_i18n 유니코드 템플릿에 `%`로 병합하는 지점이 터진다 |
| [oslo.service 주기 루프는 콜백 예외에 죽는다](notes/openstack/oslo-loopingcall-dies-on-exception.md) | 예외 한 번에 그 항목의 루프만 조용히 멈추고, 재시작 전까지 안 살아난다 |

### haproxy

| 문서 | 한 줄 |
| --- | --- |
| [`http-check expect` — 헬스체크 합격 조건](notes/haproxy/http-check-expect.md) | `expect` 를 안 쓰면 2xx·3xx 전부 합격. 좁게 박아두면 백엔드가 정상 응답을 바꿨을 때 오히려 DOWN 된다 |
| [쿠키 기반 세션 지속성](notes/haproxy/cookie-session-persistence.md) | 쿠키는 LB 가 발급한다. 매핑표가 설정 파일이라 상태가 없고, 그래서 Active/Standby 가 바뀌어도 유지된다 |

### salt

| 문서 | 한 줄 |
| --- | --- |
| [state 와 pillar — 로직과 데이터의 분리](notes/salt/state-vs-pillar.md) | `file.managed` 파일의 정본은 저장소다. 노드에서 고치면 다음 적용 때 원복된다 |

### packaging

| 문서 | 한 줄 |
| --- | --- |
| [이미지 패키지 핀과 버전 스큐](notes/packaging/version-skew-in-image-pins.md) | 호출측·구현측을 나눠 만들면 한쪽 핀만 올라가도 빌드는 통과하고 런타임에 `AttributeError`로 터진다 |

### shell

| 문서 | 한 줄 |
| --- | --- |
| [expect 스크립트가 느릴 때](notes/shell/expect-timeout-diagnosis.md) | 총 소요가 `set timeout` 값과 비슷하면, 패턴 안 맞은 `expect` 하나가 조용히 다 태우는 것이다 |

## 작성 규칙

- 한 파일 = 한 개념. 파일명은 kebab-case.
- 페이지 구성: `한 줄 요약` → `왜 그렇게 동작하나` → `확인하는 방법` → `함정`.
- 실측으로 확인한 것과 추론을 구분한다. 사실은 표시 없이, 추론·제안은 💡, 주의는 ⚠️.
- 환경 고유값(호스트명·주소·조직 내부 식별자)은 적지 않는다. 자리표시자(`<pod>`, `<chart>`)로 쓴다.
  사례는 식별자 없이 상황만 서술한다.

---

개인 학습 노트이며 특정 조직의 구성이나 운영 정보를 담지 않는다.
내용은 작성 시점의 이해를 기록한 것이라 최신 버전의 동작과 다를 수 있다.
