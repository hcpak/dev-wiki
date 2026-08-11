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

## 함정

- ⚠️ **동작은 하는데 느려지는 유형의 사고**라 눈에 잘 안 띈다. 에러가 아니라 지연·타임아웃으로 나타난다.
- ⚠️ `monkey_patch()`는 **import 순서에 민감하다.** 다른 모듈이 표준 함수를 이미 참조해둔 뒤에 패치하면
  그 참조는 갈리지 않는다. 그래서 보통 진입점 최상단에서 호출한다.
- 💡 라이브러리 코드에서는 monkey patch를 하지 말고(호출자 환경을 바꿔버리므로) green 모듈을 명시하는 편이 안전하다.
- eventlet 위에서 도는 코드에 외부 명령 실행을 추가할 때는 **항상 green subprocess인지 확인**할 것.

## 관련

- [neutron utils.execute의 인코딩 사고 경로](../openstack/neutron-utils-execute.md)
