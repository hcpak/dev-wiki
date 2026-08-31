# `AFTER_*` 콜백의 예외는 재전파되지 않고, 그 콜백의 남은 단계가 통째로 누락된다

> 콜백 매니저는 `BEFORE_*` 와 `PRECOMMIT_*` 의 예외만 다시 던진다. `AFTER_*` 는 스택을 로그로 남기고 넘어간다. 그래서 API 는 성공으로 끝나고, 예외가 난 지점 **뒤에 있던 콜백 코드가 전부 실행되지 않는다.** 캐시 동기화 통지나 정리 작업이 여기 딸려 있으면 아무 신호 없이 빠진다.

**상태**: 활성
**마지막 검증**: 2026-08-28

## 왜 그렇게 동작하나

`[코드]` `neutron_lib/callbacks/manager.py` 의 통지 경로가 이벤트 종류로 갈린다.

```python
errors = self._notify_loop(resource, event, trigger, **kwargs)
if errors:
    if event.startswith(events.BEFORE):
        self._notify_loop(resource, abort_event, trigger, **kwargs)
        raise exceptions.CallbackFailure(errors=errors)
    if event.startswith(events.PRECOMMIT):
        raise exceptions.CallbackFailure(errors=errors)
    # AFTER_* 는 여기서 아무 일도 일어나지 않는다
```

```python
for callback_id, callback in callbacks:
    try:
        callback(resource, event, trigger, **kwargs)
    except Exception as e:
        abortable_event = (event.startswith(events.BEFORE) or
                           event.startswith(events.PRECOMMIT))
        if not abortable_event:
            LOG.exception("Error during notification for ...")
        else:
            LOG.debug(...)
        errors.append(...)
```

- `[코드]` `AFTER_*` 예외는 `LOG.exception` 으로만 남는다. 호출자에게는 성공으로 보인다.
- `[코드]` 손실은 두 겹이다. ① 그 콜백이 하려던 일 ② **예외 지점 이후의 모든 문장**. 하나의 콜백 함수가 여러 후처리를 순서대로 하고 있으면 뒤쪽이 통째로 사라진다.

```python
def _after_create_port(self, resource, event, trigger, context, port, **kwargs):
    if self._validate(context, port):        # ← 여기서 예외가 나면
        self._create_child_records(context, port)
    self._notify_cache_sync(context, port)   # ← 이 줄도 실행되지 않는다
```

- `[코드]` 예외가 `if` 조건식에서 나면 조건 판정뿐 아니라 함수 전체가 중단된다. 조건식에 함수 호출을 넣은 코드가 특히 취약하다.
- `[코드]` 반대로 `BEFORE_*`/`PRECOMMIT_*` 에 붙인 훅은 같은 예외가 `CallbackFailure` 로 올라가 요청을 실패시킨다. **어느 이벤트에 붙였는지가 관측 가능성을 결정한다.**

## 확인하는 방법

`[코드]` 로그에서 매니저가 남기는 고정 문구를 찾는다. 사용자 눈에 안 보이는 실패의 유일한 흔적이다.

```
grep "Error during notification for" <서비스 로그>
```

`[추정]` 재현하려면 콜백 안에서 의도적으로 예외를 던지고 ① API 응답이 성공인지 ② 콜백 뒷부분의 부수효과가 남았는지를 본다. — 이 실험 자체는 돌려보지 않았다. 위 코드 경로에서 추론한 절차다.

## 이 노트가 틀렸다면

- **`AFTER_*` 콜백에서 예외를 던졌는데 API 가 실패하면** → 그 버전의 매니저는 재전파한다. 이 노트의 전제가 깨진다.
- **로그에 `Error during notification for` 가 없는데 뒷단계만 누락됐으면** → 예외가 아니라 조건 분기·조기 반환이 원인이다.
- **예외를 잡아 정상 반환으로 바꿨는데도 뒷단계가 여전히 안 돌면** → 누락 원인이 예외 전파가 아니다.

## 적용 범위

- `neutron_lib.callbacks` 레지스트리를 쓰는 코드. 다른 이벤트 프레임워크는 재전파 정책이 다를 수 있다.
- 이벤트 이름이 `AFTER_` 로 시작하는 경우에만 해당한다. 판정이 **문자열 접두사**로 이루어지므로 커스텀 이벤트 이름을 쓰면 의도와 다르게 분류될 수 있다.

## 함정

- ⚠️ **"에러가 없었으니 정상"이 성립하지 않는다.** API 는 200 을 주고 사용자는 아무것도 못 느낀다. 캐시가 안 맞거나 정리가 안 된 상태만 나중에 발견된다.
- ⚠️ **원인 예외가 다른 곳에서 온 것일 수 있다.** 콜백이 호출한 공통 검증 함수가 던진 예외라면, 그 검증을 고치는 순간 여러 콜백의 누락이 한꺼번에 해소된다 — 반대로 말하면 증상만 보고는 범위를 알 수 없다.
- 💡 후처리가 여러 개인 콜백은 **부수효과 순서**를 의심한다. 실패해도 남아야 하는 통지는 앞으로 빼거나 각각 try 로 감싼다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-28 | 콜백 매니저의 재전파 분기와 루프 코드 확인 | 재전파 정책 `[코드]`, 재현 절차는 `[추정]` |

## 관련

- [커밋 이후 훅에서 실패하면 자식 레코드가 고아로 남는다](after-commit-hook-failure-orphans-children.md) — 이 예외 삼킴이 DB 에 남기는 결과(고아 행)
- [조회 API 의 "없음"이 None 인지 예외인지는 시그니처에 없다](../software-design/absence-contract-none-or-exception.md) — 콜백 안에서 이런 예외를 만들어내는 전형적인 원인
- [oslo.service 주기 루프는 콜백 예외에 죽는다](oslo-loopingcall-dies-on-exception.md) — 예외 하나가 조용히 기능을 멈추는 다른 사례
