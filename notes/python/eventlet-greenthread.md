# eventlet 그린스레드 — 블로킹 호출 하나가 서버 전체를 멈추는 이유

> eventlet은 **OS 스레드 하나를 공유하는 협력형 스케줄링**이다.
> 실행권을 자발적으로 넘기지 않는 표준 블로킹 호출을 하면, 그동안 서버 전체가 멈춘다.

## 왜 그렇게 동작하나

- **그린스레드**는 OS 스레드가 아니라 eventlet의 hub가 관리하는 사용자 수준 실행 단위다.
  전환은 "블로킹이 필요해진 지점에서 스스로 양보(yield)"할 때만 일어난다 — 그래서 **협력형**이다.
- 양보는 eventlet이 갈아끼운 green 버전 함수(green socket, green subprocess 등)를 통과할 때 일어난다.
- **표준 라이브러리의 블로킹 호출**(`time.sleep`, `socket.recv`, `subprocess.communicate` 등)을 그대로
  부르면 OS 스레드 자체가 멈춘다 → hub가 다른 그린스레드로 전환할 기회를 얻지 못한다 →
  **같은 프로세스의 모든 요청이 그동안 정지**한다.

두 가지 해결책이 있다.

| 방법 | 하는 일 |
| --- | --- |
| `eventlet.monkey_patch()` | 표준 라이브러리를 런타임에 green 버전으로 **통째로 교체** |
| `from eventlet.green import subprocess` | 필요한 모듈만 green 버전으로 **명시 지정** |

monkey patch를 하지 않는 프로그램에서는 후자를 직접 챙겨야 한다.

## 실제로 판단이 필요했던 경우

Prometheus exporter가 eventlet wsgi 위에서 도는데 `monkey_patch()`는 호출하지 않는 구조였다.

```python
import eventlet
from eventlet import wsgi
...
wsgi.server(eventlet.listen((host, port)), self.app)
```

이 exporter는 지표 수집을 위해 외부 명령을 실행한다(`pidstat`은 1초 샘플링, `netstat` 등).
표준 `subprocess`로 실행하면 **그 1초 동안 exporter 전체가 응답하지 못한다** — 스크레이프 타임아웃과 직결된다.
그래서 명령 실행부를 고칠 때 `from eventlet.green import subprocess`를 명시했다.

## 확인하는 방법

monkey patch 여부부터 본다 — 없으면 green 모듈을 직접 써야 한다는 신호다.

```bash
grep -rn "monkey_patch\|from eventlet.green" <패키지 소스>
```

블로킹 여부는 이렇게 드러난다. green 버전이면 두 그린스레드가 겹쳐 돌아 총 1초,
표준 버전이면 순차 실행되어 2초가 걸린다.

```bash
python2 -c "
import eventlet, time
from eventlet.green import subprocess as gsub
def job(): gsub.call(['sleep', '1'])
t = time.time()
p = eventlet.GreenPool(); p.spawn(job); p.spawn(job); p.waitall()
print('green  :', round(time.time() - t, 2), 's')
"
```

## 진단 — 노드가 한가한데 프로세스는 멈춰 있다

여기서 나오는 판단 오류가 하나 있다. **워커 프로세스를 N 개 띄웠어도 각 워커는 단일 스레드다.**
그래서 노드 전체 지표(load average, CPU 사용률)는 한가하게 보이는데 특정 워커 안에서는
아무것도 진행되지 않는 상태가 성립한다. "서버가 놀고 있으니 여기가 병목일 리 없다" 는 배제가 틀린다.

한 사례에서 워커를 여러 개 띄운 서버가 노드 지표상 한가했는데(load 에 여유, CPU steal·iowait 거의 0),
한 요청은 같은 시각에 초 단위 로그 공백을 만들며 멈춰 있었다.
노드 지표만 보고 서버 측을 배제했다면 원인을 놓쳤을 것이다.

**드러나는 방식은 에러가 아니라 로그 공백이다.** 그린스레드가 실행되지 못하는 동안에는
로그도 안 찍히므로, 타임스탬프 사이의 빈 구간이 유일한 흔적이다.

```bash
# 한 프로세스의 로그에서 인접 줄 사이 간격이 큰 구간을 찾는다
grep " <pid> " <로그> | <타임스탬프 차이가 임계 이상인 구간 추출>
```

⚠️ **로그 공백만으로 "프로세스 전체가 멈췄다" 고 단정하지 말 것.** 그 구간에 애초에 처리할
요청이 없었어도 같은 모양이 된다. 같은 프로세스의 **다른 요청** 로그가 그 사이에 찍히는지 함께 본다 —
찍힌다면 프로세스는 돌고 있고 그 요청만 막힌 것이다. 실제 사례에서는 두 양상이
섞여 있어(완전 공백 구간과 다른 요청이 찍힌 구간) 어느 쪽인지 확정하지 못했다.

## 이 노트가 틀렸다면

- **monkey patch 가 적용돼 있고 green 모듈만 쓰는데도** 같은 지연이 나면 → 원인이 협력형 스케줄링이
  아니다. 다른 곳(브로커, 네트워크, DB)을 봐야 한다
- 위의 「green 1 초 / 표준 2 초」 비교에서 **둘 다 1 초가 나오면** → 그 환경에는 이미 monkey patch 가
  걸려 있다는 뜻이다. green 모듈을 명시할 이유가 없어진다
- 로그 공백 구간에 **그 프로세스로 들어온 요청이 있었음을 다른 관측점(액세스 로그 등)으로 확인했는데도
  아무 로그가 없으면** → 프로세스 정지가 실증된다. (이 확인은 해보지 않았다)

## 함정

- ⚠️ **동작은 하는데 느려지는 유형의 사고**라 눈에 잘 안 띈다. 에러가 아니라 지연·타임아웃으로 나타난다.
- ⚠️ `monkey_patch()`는 **import 순서에 민감하다.** 다른 모듈이 표준 함수를 이미 참조해둔 뒤에 패치하면
  그 참조는 갈리지 않는다. 그래서 보통 진입점 최상단에서 호출한다.
- 💡 라이브러리 코드에서는 monkey patch를 하지 말고(호출자 환경을 바꿔버리므로) green 모듈을 명시하는 편이 안전하다.
- eventlet 위에서 도는 코드에 외부 명령 실행을 추가할 때는 **항상 green subprocess인지 확인**할 것.

## 관련

- [neutron utils.execute의 인코딩 사고 경로](../openstack/neutron-utils-execute.md)
