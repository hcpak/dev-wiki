# Python 2 유니코드 — Encode/Decode 에러는 같은 성질의 양방향

> py2에서 `str`(bytes)과 `unicode`를 섞으면 파이썬이 **ascii로 암묵 변환**을 시도한다.
> 어느 쪽이 템플릿이냐에 따라 `UnicodeEncodeError`와 `UnicodeDecodeError`로 갈릴 뿐, 원인은 하나다.

## 왜 그렇게 동작하나

py2에는 문자열 타입이 둘 있다.

| 타입 | 리터럴 | 내용물 |
| --- | --- | --- |
| `str` | `'abc'` | 바이트 열 |
| `unicode` | `u'abc'` | 코드포인트 열 |

두 타입을 한 연산에 섞으면 파이썬이 한쪽을 다른 쪽으로 자동 변환하는데, 이때 쓰는 코덱이
**`sys.getdefaultencoding()` = ascii로 고정**되어 있다. 방향만 다른 두 사고가 여기서 나온다.

### 방향 1 — Encode 에러

**bytes 템플릿**에 unicode 값을 넣으면, 결과를 `str`로 만들어야 하므로 값을 ascii로 **encode**한다.

```python
'name="{}"'.format(u'한글이름')
# UnicodeEncodeError: 'ascii' codec can't encode characters ...
```

### 방향 2 — Decode 에러

**unicode 템플릿**에 bytes 값을 넣으면, 값을 unicode로 만들어야 하므로 bytes를 ascii로 **decode**한다.

```python
u'Stdout: %s' % b'6\xec\x9b\x9429'    # 한글 로케일에서 ps -ef 의 STIME 열
# UnicodeDecodeError: 'ascii' codec can't decode byte 0xec ...
```

한 줄로: **"바깥(템플릿)이 bytes면 안쪽을 encode, 바깥이 unicode면 안쪽을 decode"** 하고, 둘 다 ascii로 한다.

## 확인하는 방법

```bash
python2 -c "print('x={}'.format(u'한글'))"      # UnicodeEncodeError
python2 -c "print(u'x=%s' % '한글')"            # UnicodeDecodeError (bytes 리터럴)
python2 -c "import sys; print(sys.getdefaultencoding())"   # ascii
```

## 고치는 방법

1. **템플릿을 전부 `u''`로 바꾼다.** 그리고 외부에서 들어온 bytes는 경계에서 명시적으로 decode한다.

   ```python
   if isinstance(value, str):
       value = value.decode('utf-8', 'replace')
   ```

2. 외부 명령 출력처럼 코덱을 확신할 수 없는 입력은 `errors='replace'`로 받아 죽지 않게 한다.

## 함정

- ⚠️ **로케일(`LANG`/`LC_ALL`) 변경으로는 해결되지 않는다.** py2의 기본 인코딩은 로케일과 무관하게 ascii다.
  로케일은 "한글이 출력에 섞이는" 트리거를 바꿀 뿐, 변환 규칙을 바꾸지 못한다.
- ⚠️ **템플릿 부분 전환은 실패한다.** 조립 체인 중 하나라도 bytes 템플릿이면 그 지점에서 다시 터진다.
  메트릭 본문 템플릿만 `u''`로 바꾸고 footer 템플릿을 남겼다가 그대로 재현된 적이 있다 —
  **조립에 참여하는 템플릿 전부**를 바꿔야 한다.
- 예외 처리가 없는 루프에서 터지면 피해 범위가 커진다. 사용자가 지은 이름 하나 때문에 노드의
  전체 지표가 유실된 사례가 있다(exporter가 `/metrics` 500). 같은 코드라도 루프마다 try/except가
  있던 쪽은 해당 라벨 한 줄만 빠졌다.

## 관련

- [Prometheus 라벨 값 이스케이프](../prometheus/label-value-escaping.md)
- [neutron utils.execute의 인코딩 사고 경로](../openstack/neutron-utils-execute.md)
