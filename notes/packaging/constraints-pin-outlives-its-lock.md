# constraints 핀은 원본 잠금과 함께 늙는다 — 어제 되던 실행이 오늘 해석 실패로 죽는다

> 잠금 파일(lock)을 읽지 않는 실행기에는 별도의 `constraints.txt` 를 만들어 버전을 고정한다.
> 그런데 이 파일은 **잠금의 사본**이라서, 원본 프로젝트가 의존성 하한을 올리면 조용히 모순이 된다.
> 코드도 설정도 안 건드렸는데 **다음 재시작부터** 프로그램이 아예 안 뜬다.

## 왜 그렇게 동작하나

`uvx`(그리고 유사한 "저장소에서 바로 실행" 런처)는 대상 저장소의 `uv.lock` 을 사용하지 않는다.
매 실행마다 의존성을 새로 해석하므로, 재현성을 원하면 잠금을 **requirements 형식으로 내보내
`-c` 로 넘기는** 우회를 쓴다.

```bash
uvx --from git+<repo> -c constraints.txt <entrypoint>
```

이 시점에는 두 파일이 같은 사실을 말한다.

| | 값 |
| --- | --- |
| 저장소 `pyproject.toml` | `<sdk>>=1.0,<2.0` |
| 내보낸 `constraints.txt` | `<sdk>==1.28.1` |

문제는 저장소가 **메이저 상향**을 하는 순간이다.

```
pyproject.toml:  <sdk>>=2.0.0,<3.0.0     ← 상류가 올렸다
constraints.txt: <sdk>==1.28.1           ← 사본은 그대로
```

교집합이 공집합이므로 해석기가 답을 못 찾는다. 설치 단계에서 끝나고, 프로그램은 한 줄도 실행되지 않는다.

```
× No solution found when resolving tool dependencies:
╰─▶ Because <pkg>==3.2.0 depends on <sdk>>=2.0.0,<3.0.0 and <sdk>==1.28.1,
    we can conclude that <pkg>==3.2.0 cannot be used.
```

`--from git+<repo>` 는 보통 브랜치 끝(HEAD)을 가리킨다. 즉 **핀한 것은 의존성뿐이고 대상 자체는
안 핀했다.** 어느 쪽도 내 손을 거치지 않은 채, 상류가 커밋한 순간부터 조합이 깨진다.

### 왜 "어제까지는 됐는데"가 되나

실행기는 해석 결과를 캐시하고, 이미 떠 있는 프로세스는 자기 가상환경을 계속 붙들고 있다.

- 이미 돌던 프로세스 → **그대로 정상 동작**한다. 옛 해석 결과 위에 서 있다
- 재시작·새 환경·캐시가 없는 동료 → **즉시 실패**한다

그래서 깨진 시점과 드러나는 시점이 사람마다 다르고, 재시작이 잦은 쪽부터 순차적으로 터진다.
"내 것은 되는데" 라는 말이 진심인 상태가 만들어진다.

## 확인하는 방법

먼저 constraints 를 떼고 돌려서 **핀이 원인인지** 가른다. 한 줄로 판정된다.

```bash
uvx --from git+<repo> <entrypoint> </dev/null      # 성공 → 핀이 원인
```

상류가 무엇을 요구하는지 직접 읽는다. 잠금 파일에도 같은 사실이 적혀 있다.

```bash
git clone --depth 20 <repo> && cd <repo>
grep -nE '<sdk>' pyproject.toml uv.lock
git log --date=iso --pretty='%h %ad %s' -5     # 상향 커밋 시각 = 깨진 시각
```

실제로 돌고 있는 프로세스가 **어느 환경**을 쓰는지도 본다. 이게 "누구는 되고 누구는 안 되는"
현상의 근거가 된다.

```bash
ps -eo pid,etime,command | grep <entrypoint>
<그 프로세스의 python> -c "import importlib.metadata as m; print(m.version('<sdk>'))"
```

고치는 방법은 사본을 다시 뜨는 것뿐이다. 파일 첫 줄에 **생성 명령을 주석으로 남겨두면**
이 상황에서 그대로 재실행할 수 있다.

```bash
uv export --directory <repo> --format requirements-txt \
  --no-hashes --no-dev --no-emit-project --no-annotate --no-header > constraints.txt
```

## 함정

- **설치 실패는 로그를 안 남기는 실패다.** 프로그램이 뜨지 않았으니 애플리케이션 로그가 비어 있고,
  런처의 stderr 를 직접 봐야 문구가 나온다. 감독 프로세스가 stderr 를 버리면 "그냥 안 뜬다"로만 보인다.
- ⚠️ **핀을 배포하는 사람과 상류를 올리는 사람이 다를 때 새어 나간다.** 배포물 안의 constraints 는
  상류 저장소의 CI 가 검사하지 않는다. 상향 커밋에 "사본 재생성"이 딸려오게 만들지 않으면 계속 재발한다.
- ⚠️ **대상은 HEAD, 의존성은 고정**이라는 비대칭이 근본 원인이다. 재현성이 목적이라면 대상도
  태그·커밋으로 핀해야 한다. 한쪽만 고정하면 재현성은 얻지 못하고 모순 위험만 얻는다.
- 💡 배포물을 설치한 직후 **엔트리포인트를 한 번 띄워보는 것만으로** 걸러진다. 프로토콜을 타는
  프로그램이면 최초 핸드셰이크 한 번을 주고받아 응답을 확인하는 스모크 테스트가 가장 싸다.
- 하위 호환 상향(`>=1.5` → `>=1.6`)에서는 안 터진다. **메이저 상한이 움직일 때만** 모순이 되므로
  몇 달간 조용하다가 한 번에 드러난다.

## 관련

- [이미지 패키지 핀과 버전 스큐](version-skew-in-image-pins.md) — 여러 핀을 손으로 맞출 때 한쪽만 올라가는 사고
- [pip `--upgrade` 는 의존성까지 건드린다](../python/pip-upgrade-dependency-resolution.md) — 설치가 성공해도 임포트가 깨질 수 있다
