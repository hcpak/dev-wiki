# neutron `utils.execute()`가 한글 로케일에서 터지는 경로

> `execute()`는 명령 출력을 **oslo_i18n 유니코드 템플릿에 `%`로 병합**한다.
> py2에서 출력이 bytes이고 거기 non-ascii가 섞여 있으면 그 지점에서 암묵 ascii 디코드가 일어나 터진다.

## 왜 그렇게 동작하나

`neutron/agent/linux/utils.py`의 `execute()` 흐름:

```python
obj, cmd = create_process(cmd, run_as_root=run_as_root, addl_env=addl_env)
_stdout, _stderr = obj.communicate(_process_input)      # py2 에서는 bytes
...
msg = _("Exit code: %(returncode)d; Stdin: %(stdin)s; "
        "Stdout: %(stdout)s; Stderr: %(stderr)s") % {   # ← oslo_i18n Message(unicode)
            'returncode': returncode, 'stdin': process_input or '',
            'stdout': _stdout, 'stderr': _stderr}
```

`_()`가 돌려주는 것은 유니코드 계열 객체다. 거기에 bytes를 `%`로 넣으면 파이썬이 bytes를
ascii로 **decode**하려 하고, 한 바이트라도 non-ascii면 `UnicodeDecodeError`가 난다
(→ [py2 유니코드 대칭](../python/py2-unicode-symmetry.md)의 "방향 2").

**트리거**: `LANG=ko_KR.UTF-8` 환경에서 `ps -ef`의 `STIME` 열이 `6월29`처럼 한글로 렌더링된다.
평소엔 ascii만 나오던 명령이 로케일 때문에 갑자기 non-ascii를 뱉는 것이다.

상위 버전 neutron에는 가드가 들어가 있다(upstream `Make sure we return unicode strings for process output`):

```python
_stdout = helpers.safe_decode_utf8(_stdout)
_stderr = helpers.safe_decode_utf8(_stderr)
```

⚠️ 이 가드가 있는 버전을 쓰는 코드와 아닌 코드가 섞여 있을 수 있다.
**"neutron에는 고쳐졌다"가 내가 쓰는 패키지에도 고쳐졌다는 뜻은 아니다.** 실제로 가드가 없는 다른
패키지에서 같은 사고가 다시 났다.

## 우회: subprocess 직접 사용

`execute()`도 내부는 결국 `Popen`이다. 차이는 그 **위에 얹힌 로그·예외 조립 계층**이므로, 그 계층을
걷어내고 우리가 정한 코덱으로만 디코드하면 자동 변환이 끼어들 틈이 없다.

```python
from eventlet.green import subprocess

def execute_as_root(cmd):
    root_helper = agent_config.get_root_helper(oslo_cfg.CONF)
    full_cmd = shlex.split(root_helper) + list(cmd)
    proc = subprocess.Popen(full_cmd, stdout=subprocess.PIPE,
                            stderr=subprocess.PIPE)
    stdout, stderr = proc.communicate()
    if proc.returncode != 0:
        LOG.error('Command %s failed (rc=%s): %s', full_cmd, proc.returncode,
                  stderr.decode('utf-8', 'replace'))
        raise subprocess.CalledProcessError(proc.returncode, full_cmd)
    return stdout.decode('utf-8', 'replace')
```

설계 포인트 세 가지:

1. **권한 동작 보존** — neutron `create_process()`와 똑같이 `AGENT.root_helper`를 `shlex.split`해 앞에 붙인다.
2. **`check_output`이 아니라 `Popen`** — `check_output`은 stderr를 따로 캡처할 수 없고,
   `CalledProcessError`는 종료 코드만 들고 간다. 실패 원인을 로그에 남기려면 `Popen`으로 내려가야 한다.
   (`execute()`는 예외 메시지에 stdout/stderr를 넣어줬는데, 그 편의를 대체하는 부분)
3. **`eventlet.green.subprocess`** — → [eventlet 그린스레드](../python/eventlet-greenthread.md)

## 함정

- ⚠️ `AGENT.root_helper_daemon`이 설정된 환경에서는 neutron이 rootwrap **데몬** 경로로 실행한다.
  위 우회는 항상 `root_helper` 경로(매번 fork)로 실행하므로 결과는 같고 비용만 다르다.
  적용 전에 대상 노드의 설정을 확인할 것.

  ```bash
  grep -rn "root_helper_daemon" /etc/neutron/
  ```
- 💡 로그 인자를 `LOG.error('… %s', value)` 형태(포맷을 로거에 위임)로 넘기면 로그 레벨이 꺼져 있을 때
  포맷 자체가 일어나지 않아 사고 확률이 준다.
