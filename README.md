# dev-wiki

일하다 막혀서 파고들어 이해한 것들, 그리고 일하는 방식에 대해 정리한 개인 위키.

무엇을 했는지가 아니라 **왜 그렇게 동작하는가**, **왜 그렇게 하기로 했는가**를 남긴다.
같은 계열의 문제를 다시 만났을 때 재사용하는 것이 목적이다.

두 가지가 들어 있다.

| 디렉토리 | 성격 | 길이 |
| --- | --- | --- |
| [`notes/`](notes/) | 개념 하나를 정리한 배경지식 노트 | 짧음 |
| [`posts/`](posts/) | 경험과 판단을 서술한 글 | 김 |

`notes/`는 대부분 공개된 오픈소스(Kubernetes, OpenStack, HAProxy, Salt, Paste/PasteDeploy, eventlet, pip 등)의
**문서에 잘 안 적혀 있거나, 적혀 있어도 실제로 데어봐야 이해되는 지점**을 다룬다.

`posts/`는 그보다 넓다. 도구를 만들면서 내린 결정, 자동화의 경계를 어디에 그었는지 같은 것들이다.

## 글

| 편 | 글 | 한 줄 | 발행 |
| --- | --- | --- | --- |
| 1 | [자동화 문서를 계속 고치다 보니, 절차서가 아니라 사고 기록부가 되어 있었다](posts/01-2026-08-12-skills-as-failure-log.md) | AI에게 반복 업무를 맡기며 알게 된 것 — 문서의 가치는 절차가 아니라 금지 목록에 있었다 | [velog](https://velog.io/@hugh0703/skills-as-failure-log) |
| 2 | [상태를 보여주는 패널을 만들었더니, 무엇을 안 보여줄지가 일이 됐다](posts/02-2026-08-13-what-not-to-show.md) | 몇 줄짜리 화면에서 시간을 쓴 곳은 전부 "무엇을 뺄까"였다 | [velog](https://velog.io/@hugh0703/what-not-to-show) |
| 3 | [일을 맡기면 대화가 막혔다 — 그래서 나눠 돌리기 시작했다](posts/03-2026-08-13-delegating-in-parallel.md) | 병렬 위임의 실제 — 실행은 병렬이 되는데 검토는 병렬이 안 된다 | (초안) |

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
| [venv 는 디렉터리를 옮기면 조용히 죽는다](notes/python/venv-breaks-when-repo-moves.md) | 셔뱅이 절대경로라 bad interpreter 로 전멸하는데, 에디터 lint 실종으로만 나타난다 |

### git

| 문서 | 한 줄 |
| --- | --- |
| [pull --rebase 는 미푸시 로컬 커밋을 원격 위에 "재생성" 한다](notes/git/pull-rebase-replays-unpushed-commits.md) | 이동이 아니라 새 커밋 생성 — 해시가 바뀌고 reflog 에 원본이 남는다. 다시 쓰는 범위는 미푸시분뿐이라 다른 클론과 어긋나지 않고, 충돌은 커밋 단위로 난다 |
| [worktree — 저장소 하나에 작업 디렉터리 N개](notes/git/worktree-one-repo-many-checkouts.md) | .git 은 공유, HEAD·index·작업 파일만 분리. stash 왕복과 .pyc 오염이 사라지고, ignored 파일 미상속이 도입 비용이다 |
| [stash 는 커밋이다 — ref 를 붙이면 drop 후에도 복원된다](notes/git/stash-archive-with-protected-refs.md) | update-ref 아카이브는 gc 에 살아남고, 원시점 복원은 `^1`, 현재 브랜치 apply no-op 은 "이미 반영됨" 판별법이다 |
| [squash merge 는 부모 하나짜리 일반 커밋을 만든다](notes/git/squash-merge-produces-a-single-parent-commit.md) | 브랜치를 써도 main 은 일직선. "갈래 보이는 히스토리" 요구는 squash-only 정책과 양립 불가 — 쟁점은 브랜치 운용이 아니라 머지 방식 설정이다 |
| [한 번의 git push 에서 refspec 들은 독립적으로 성공하고 실패한다](notes/git/push-refspecs-succeed-independently.md) | 브랜치가 거부돼도 함께 보낸 태그는 올라간다. exit code 는 "하나라도 실패" 만 알려주고, 태그 트리거 CI 는 이미 돌기 시작한다 |

### prometheus

| 문서 | 한 줄 |
| --- | --- |
| [라벨 값 이스케이프 순서](notes/prometheus/label-value-escaping.md) | 역슬래시를 먼저 두 배로 하지 않으면 뒤 단계가 넣은 백슬래시까지 재이스케이프된다 |

### openstack

| 문서 | 한 줄 |
| --- | --- |
| [neutron `utils.execute()`의 인코딩 사고 경로](notes/openstack/neutron-utils-execute.md) | 명령 출력 bytes를 oslo_i18n 유니코드 템플릿에 `%`로 병합하는 지점이 터진다 |
| [oslo.service 주기 루프는 콜백 예외에 죽는다](notes/openstack/oslo-loopingcall-dies-on-exception.md) | 예외 한 번에 그 항목의 루프만 조용히 멈추고, 재시작 전까지 안 살아난다 |
| [소프트 삭제는 청소 명령을 따로 돌려야 사라진다](notes/openstack/soft-delete-needs-a-reaper.md) | `deleted=1` 로 표시만 될 뿐 행은 남는다. 유니크 제약도 계속 점유한다 |
| [롤백 중 터진 2차 예외가 원인 예외를 덮어쓴다](notes/openstack/rollback-masks-original-exception.md) | `except: cleanup(); raise` 에서 `cleanup()` 이 던지면 `raise` 에 도달하지 못한다 |
| [지연 실체화 — 조회 요청이 쓰기를 유발한다](notes/openstack/lazy-activation-on-read.md) | `GET` 하나가 리소스를 생성하고 실패 시 롤백까지 한다. "조회는 안전하다"가 깨진다 |
| [공인 IP 하나로 소유자를 역추적하는 체인](notes/openstack/floatingip-to-owner-chain.md) | floating IP는 자기가 누구 것인지 모른다. `port_id` 만이 유일한 링크다 |
| [커밋 이후 훅에서 실패하면 자식 레코드가 고아로 남는다](notes/openstack/after-commit-hook-failure-orphans-children.md) | `AFTER_*` 예외는 커밋된 삭제를 못 되돌린다. 그 뒤 삭제 API 가 영구 실패한다 |
| [비관리자 조회는 tenant_id 로 자동 필터되어, 행이 있어도 NotFound 가 된다](notes/openstack/tenant-scoped-query-hides-existing-rows.md) | 같은 코드가 admin 토큰이면 성공하고 프로젝트 토큰이면 실패한다. DB 에 행이 있는데 "없다"는 응답이 오면 권한 범위를 의심한다 |
| [`AFTER_*` 콜백의 예외는 재전파되지 않고 남은 단계가 통째로 누락된다](notes/openstack/after-callback-exception-skips-the-rest.md) | API 는 200 으로 끝나고 예외 지점 뒤의 통지·정리만 조용히 빠진다. 유일한 흔적은 로그의 `Error during notification for` |

### haproxy

| 문서 | 한 줄 |
| --- | --- |
| [`http-check expect` — 헬스체크 합격 조건](notes/haproxy/http-check-expect.md) | `expect` 를 안 쓰면 2xx·3xx 전부 합격. 좁게 박아두면 백엔드가 정상 응답을 바꿨을 때 오히려 DOWN 된다 |
| [쿠키 기반 세션 지속성](notes/haproxy/cookie-session-persistence.md) | 쿠키는 LB 가 발급한다. 매핑표가 설정 파일이라 상태가 없고, 그래서 Active/Standby 가 바뀌어도 유지된다 |

### salt

| 문서 | 한 줄 |
| --- | --- |
| [state 와 pillar — 로직과 데이터의 분리](notes/salt/state-vs-pillar.md) | `file.managed` 파일의 정본은 저장소다. 노드에서 고치면 다음 적용 때 원복된다 |

### observability

| 문서 | 한 줄 |
| --- | --- |
| [히스테리시스 — 복귀 문턱은 진입 문턱보다 높게](notes/observability/hysteresis-in-state-transitions.md) | 경계에서 흔들리는 지표에 대칭 문턱을 쓰면 발생↔복구 알림이 핑퐁한다. 발생은 빠르게, 복구는 오래 정상일 때만 |

### packaging

| 문서 | 한 줄 |
| --- | --- |
| [이미지 패키지 핀과 버전 스큐](notes/packaging/version-skew-in-image-pins.md) | 호출측·구현측을 나눠 만들면 한쪽 핀만 올라가도 빌드는 통과하고 런타임에 `AttributeError`로 터진다 |
| [constraints 핀은 원본 잠금과 함께 늙는다](notes/packaging/constraints-pin-outlives-its-lock.md) | 잠금을 안 읽는 런처에 넘긴 사본이 상류 상향과 모순되면, 재시작하는 쪽부터 기동 불가 |
| [apt 는 의존성을 후보 버전으로만 자동 해석한다](notes/packaging/apt-resolves-deps-at-candidate-only.md) | 저장소에 신버전이 올라오는 순간 rc 핀 세트가 깨진다 — versioned Depends 는 해석 실패로, Recommends 는 에러 없이 조용히 탈락 |
| [같은 소프트웨어, 다른 실행 유저](notes/packaging/same-software-different-runtime-user.md) | 배포판 deb 와 벤더 deb 는 유닛 `User=` 가 다를 수 있다 — 출처를 갈아타면 기존 파일 소유권 탓에 기동 실패, 양방향 모두 |

### shell

| 문서 | 한 줄 |
| --- | --- |
| [expect 스크립트가 느릴 때](notes/shell/expect-timeout-diagnosis.md) | 총 소요가 `set timeout` 값과 비슷하면, 패턴 안 맞은 `expect` 하나가 조용히 다 태우는 것이다 |
| [OSC 8 — 보이는 글자와 링크 대상 분리하기](notes/shell/osc8-terminal-hyperlinks.md) | 이스케이프는 화면 폭을 안 먹는다. 미지원 터미널에서도 라벨만 남고 안 깨진다 |
| [glob 은 문자열 매칭이 아니라 파일시스템 질의다](notes/shell/glob-queries-the-filesystem-not-a-string.md) | 디렉터리를 열거하며 `stat()` 하기 때문에 속성으로 거르는 퀄리파이어(`*(.m-1)`)가 가능하고, 0건이 빈 결과가 아니라 에러가 된다 |
| [glob 과 regex 는 기호를 공유하지만 다른 언어다](notes/shell/glob-and-regex-are-different-languages.md) | glob `*`=regex `.*`, glob `.`은 특수문자가 아니다. 확장 주체도 셸 vs 프로그램이라, 따옴표를 빠뜨리면 패턴이 파일명으로 바뀌어 전달된다 |

### networking

| 문서 | 한 줄 |
| --- | --- |
| [`100.64.0.0/10` — 공인도 사설도 아닌 제3의 대역](notes/networking/rfc6598-shared-address-space.md) | RFC 6598 공유 주소 공간. 고객 사설 대역과 겹치지 않는 사업자 내부 배관용 |
| [SNI 가 필요한 이유 — 인증서를 골라야 할 때 `Host:` 헤더는 아직 오지 않았다](notes/networking/sni-exists-because-tls-precedes-the-host-header.md) | TLS 협상이 HTTP 요청보다 먼저 끝나야 하는데 `Host:` 는 그 암호화 채널 안에 있다. SNI 는 도메인명을 ClientHello 에 평문으로 얹어 이 순환을 끊는다 |
| [자물쇠가 정상인데 404 — TLS 판정과 L7 라우팅은 다른 계층이다](notes/networking/tls-handshake-success-is-not-routing-success.md) | TLS 는 SNI 로, 라우팅은 `Host:` 헤더로 따로 판정한다. 도메인 이전에서 인증서만 맞추고 규칙을 놓치면 이 증상이 나온다 |

### api

| 문서 | 한 줄 |
| --- | --- |
| [응답을 못 받은 것과 요청이 실패한 것은 다르다](notes/api/lost-response-is-not-a-failed-write.md) | 재시도 전에 상태를 조회한다. 멱등하지 않은 요청의 자동 재시도는 장애를 데이터 오염으로 바꾼다 |
| [잘못 배포된 자격증명이 유효하면 실패가 아니라 남의 데이터가 된다](notes/api/valid-credential-wrong-tenant-succeeds.md) | 멀티테넌트 push 게이트웨이는 자격증명으로 귀속을 정한다. 이웃 환경 값이 들어가면 2xx 무증상으로 데이터만 사라진다 |

### filesystem

| 문서 | 한 줄 |
| --- | --- |
| [birthtime 은 "언제 만들어졌나" 를 알려주지 않는다](notes/filesystem/birthtime-is-not-creation-identity.md) | 원자적 교체가 새 inode 를 만들어, 내용만 고쳐도 생성 시각이 갱신된다 |

### ops

| 문서 | 한 줄 |
| --- | --- |
| [워크로드가 이관된 호스트는 옛 버전을 진실처럼 말한다](notes/ops/migrated-workload-leaves-a-lying-host.md) | 조회가 실패하지 않고 **성공한다는 점**이 함정이다. 정지 누락 시 잔존 프로세스가 트래픽까지 오염시킨다. 유닛 상태부터 본다 |
| [오래 떠 있는 프로세스는 버려진 프로세스가 아니다](notes/ops/process-age-does-not-mean-abandoned.md) | 나이는 소유자를 말해주지 않는다. 고아의 신호는 경과 시간이 아니라 부모 부재이고, 삭제 전 참조는 커맨드라인으로 센다 |
| [정본에 반영 안 된 핫픽스는 다음 배포가 조용히 되돌린다](notes/ops/hotfix-outside-source-of-truth-gets-reverted.md) | 노드 수정은 드리프트로 취급돼 배포가 옛 값으로 원복한다. 배포 직후 회귀는 코드보다 conf mtime·정본 이력부터 대조 |

### whisper

| 문서 | 한 줄 |
| --- | --- |
| [Whisper 는 무음에서 "없음"을 출력하지 않는다](notes/whisper/whisper-invents-sentences-on-silence.md) | 무발화 녹음이 학습 잔재 문장으로 채워진다. 반복 검사로는 못 잡으니 volumedetect 의 mean/max 를 병행 — max 정상·mean 극저면 장비가 아니라 발화 부재다 |

### search

| 문서 | 한 줄 |
| --- | --- |
| [리랭커는 정직하게 실패하지 않는다](notes/search/reranker-does-not-fail-honestly.md) | BM25 는 못 찾으면 "결과 없음", 리랭커는 오답에도 88%. 작은 코퍼스의 하이브리드 검색은 자신 있게 틀린다 |

### security

| 문서 | 한 줄 |
| --- | --- |
| [접두사 허용 규칙은 권한 경계가 아니다](notes/security/prefix-allowlist-is-not-a-capability-boundary.md) | 문자열 비교일 뿐 플래그를 해석하지 않는다. 읽기 전용인 줄 알았던 하위 명령이 쓰기까지 연다 |

### knowledge

| 문서 | 한 줄 |
| --- | --- |
| [요약 인덱스는 원문과 조용히 어긋난다](notes/knowledge/summary-index-drift.md) | 요약이 훌륭할수록 원문을 안 열게 되어, 어긋났다는 사실 자체가 드러나지 않는다 |

### llm

| 문서 | 한 줄 |
| --- | --- |
| [에이전트 세션의 고정 오버헤드는 첫 질문 전에 이미 토큰을 쓴다](notes/llm/session-overhead-spends-tokens-before-work.md) | 매 세션 자동 주입물이 조용히 누적된다. %는 창 상대값이라 절대 토큰으로 비교하고, 지연 로드 도구는 비용 0에 가깝다 |
| [한 세션은 모델을 하나만 갖는다 — 서브에이전트가 티어를 바꾸는 유일한 통로다](notes/llm/session-model-is-fixed-subagents-are-the-only-tier-lever.md) | effort 는 호출로 못 넘기고 에이전트 파일에만 박힌다. 속도의 주 레버는 모델 크기가 아니라 effort 이고, 도구 집약 작업은 약한 모델이 오히려 비싸다 |

### software-design

| 문서 | 한 줄 |
| --- | --- |
| [조회 API 의 "없음"이 None 인지 예외인지는 시그니처에 없다](notes/software-design/absence-contract-none-or-exception.md) | `is not None` 가드가 도달 불가 코드로 남는다. 정상 입력에서는 증상이 없어 리뷰·테스트를 통과한다 |

## 작성 규칙

**공통**

- 실측으로 확인한 것과 추론을 구분한다. 사실은 표시 없이, 추론·제안은 💡, 주의는 ⚠️.
- 환경 고유값(호스트명·주소·조직 내부 식별자)은 적지 않는다. 자리표시자(`<pod>`, `<chart>`)로 쓴다.
  사례는 식별자 없이 상황만 서술한다.

**notes/**

- 한 파일 = 한 개념. 파일명은 kebab-case.
- 페이지 구성: `한 줄 요약` → `왜 그렇게 동작하나` → `확인하는 방법` → `이 노트가 틀렸다면`
  → `적용 범위` → `함정` → `검증 이력`.
- 주장마다 근거 강도를 붙인다 — `[실측]`(출력을 봤다) · `[코드]`(읽고 추론했다) ·
  `[문서]`(공식 문서에 있다) · `[추정]`(확인하지 않았다). 문서 단위가 아니라 주장 단위다.
- **`이 노트가 틀렸다면` 절에 노트를 뒤집을 관측을 미리 적는다.** 반증 조건을 쓸 수 없는
  주장은 노트에 넣지 않는다. 검증이 "생각하기"가 아니라 "명령 한 번"이 되게 하는 장치다.
- 검증 이력은 덮어쓰지 않고 쌓는다. 틀린 것으로 판명된 노트도 지우지 않고 상태만 `폐기` 로 바꾼다.
- 새 형식은 2026-08-13 부터 적용한다. 그 전에 쓴 노트는 해당 주제를 다시 만날 때 전환한다.

**posts/**

- 파일명은 `NN-YYYY-MM-DD-slug.md` — 앞의 두 자리가 편 번호다.
- 결론만 적지 않는다. 처음에 무엇을 잘못 생각했는지, 어떻게 판단이 바뀌었는지를 같이 적는다.
- 아직 해결하지 못한 것은 해결한 척하지 않고 그대로 적는다.

---

개인 학습 노트이며 특정 조직의 구성이나 운영 정보를 담지 않는다.
내용은 작성 시점의 이해를 기록한 것이라 최신 버전의 동작과 다를 수 있다.
