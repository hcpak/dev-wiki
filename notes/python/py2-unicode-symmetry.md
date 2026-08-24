# Python 2 유니코드 — Encode/Decode 에러는 같은 성질의 양방향

> py2에서 `str`(bytes)과 `unicode`를 섞으면 파이썬이 **ascii로 암묵 변환**을 시도한다.
> 어느 쪽이 템플릿이냐에 따라 `UnicodeEncodeError`와 `UnicodeDecodeError`로 갈릴 뿐, 원인은 하나다.

**상태**: 활성
**마지막 검증**: 2026-08-20

## 왜 그렇게 동작하나

py2에는 문자열 타입이 둘 있다.

| 타입 | 리터럴 | 내용물 |
| --- | --- | --- |
| `str` | `'abc'` | 바이트 열 |
| `unicode` | `u'abc'` | 코드포인트 열 |

`[문서]` 두 타입을 한 연산에 섞으면 파이썬이 한쪽을 다른 쪽으로 자동 변환하는데, 이때 쓰는 코덱이
**`sys.getdefaultencoding()` = ascii**다. 방향만 다른 두 사고가 여기서 나온다.

### 방향 1 — Encode 에러

`[실측]` **bytes 템플릿**에 unicode 값을 넣으면, 결과를 `str`로 만들어야 하므로 값을 ascii로 **encode**한다.

```python
'name="{}"'.format(u'한글이름')
# UnicodeEncodeError: 'ascii' codec can't encode characters ...
```

### 방향 2 — Decode 에러

`[실측]` **unicode 템플릿**에 bytes 값을 넣으면, 값을 unicode로 만들어야 하므로 bytes를 ascii로 **decode**한다.

```python
u'Stdout: %s' % b'6\xec\x9b\x9429'    # 한글 로케일에서 ps -ef 의 STIME 열
# UnicodeDecodeError: 'ascii' codec can't decode byte 0xec ...
```

한 줄로: **"바깥(템플릿)이 bytes면 안쪽을 encode, 바깥이 unicode면 안쪽을 decode"** 하고, 둘 다 ascii로 한다.

`[실측]` 이 에러는 값이 **변환되는 시점**(포맷/조립)에 나는 것이지 저장된 데이터가 오염된 것이 아니다.
따라서 포맷하는 쪽 코드만 고치면 기존 데이터는 손대지 않아도 즉시 정상 동작한다 — 조회 시점에 라벨을
조립하는 exporter류에서 특히 그렇다(수정본 배포만으로 기존 리소스의 지표 수집이 재개된다).

## 확인하는 방법

```bash
python2 -c "print('x={}'.format(u'한글'))"      # UnicodeEncodeError
python2 -c "print(u'x=%s' % '한글')"            # UnicodeDecodeError (bytes 리터럴)
python2 -c "import sys; print(sys.getdefaultencoding())"   # ascii
```

## 이 노트가 틀렸다면

- py2에서 `sys.getdefaultencoding()`이 ascii가 아닌 값으로 나오면 → 그 환경은 `sitecustomize` 등으로 기본 인코딩이 변조된 것 — 이 노트의 재현식이 통과하더라도 주장이 틀린 게 아니라 **적용 범위 밖**이다
- 로케일(`LANG`/`LC_ALL`)만 바꿨는데 위 재현식의 에러가 사라지면 → "로케일 무관" 주장이 틀림
- 조립에 참여하는 템플릿을 전부 `u''`로 바꿨는데도 같은 조립 체인에서 Encode 에러가 재현되면 → "전부 전환하면 해결" 주장이 틀림 (경계에서 decode 안 한 bytes 입력을 먼저 의심할 것)
- 포맷 코드 수정·배포 후에도 **기존** 데이터에 대해서만 에러가 계속 나면 → "저장 데이터 무관, 변환 시점 문제" 주장이 틀림

## 적용 범위

- CPython 2.7, 기본 인코딩 미변조(ascii) 환경
- py3는 해당 없음 — 암묵 변환 자체가 없어 str/bytes 혼합은 `TypeError`나 repr 삽입(`b'...'`)으로 나타난다

## 함정

- ⚠️ **로케일(`LANG`/`LC_ALL`) 변경으로는 해결되지 않는다.** py2의 기본 인코딩은 로케일과 무관하게 ascii다.
  로케일은 "한글이 출력에 섞이는" 트리거를 바꿀 뿐, 변환 규칙을 바꾸지 못한다.
- ⚠️ **템플릿 부분 전환은 실패한다.** 조립 체인 중 하나라도 bytes 템플릿이면 그 지점에서 다시 터진다.
  메트릭 본문 템플릿만 `u''`로 바꾸고 footer 템플릿을 남겼다가 그대로 재현된 적이 있다 —
  **조립에 참여하는 템플릿 전부**를 바꿔야 한다.
- ⚠️ 예외 처리가 없는 루프에서 터지면 피해 범위가 커진다. 사용자가 지은 이름 하나 때문에 노드의
  전체 지표가 유실된 사례가 있다(exporter가 `/metrics` 500). 같은 코드라도 루프마다 try/except가
  있던 쪽은 해당 라벨 한 줄만 빠졌다.
- 💡 코드를 못 고치는 상황의 임시 완화로 `sitecustomize.py`에서 기본 인코딩을 utf-8로 바꾸는 우회가
  있다(실제 사용 사례 있음). 동작하지만 프로세스 전역을 바꾸는 것이라 수정본 배포 후 반드시 걷어낸다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-07 | 최초 작성 — exporter 한글 라벨 장애 분석·수정에서 도출 | `[실측]` 기반 |
| 2026-08-20 | "수정본 배포만으로 기존 리소스 해소되나"라는 QA 질문에 이 노트로 답변(변환 시점 문제, 저장 데이터 무관) | 유지 · 변환 시점 절 추가 |
| 2026-08-21 | 템플릿 형식 전환, 반증 조건·적용 범위 추가 | 유지 |

## 관련

- [Prometheus 라벨 값 이스케이프](../prometheus/label-value-escaping.md)
- [neutron utils.execute의 인코딩 사고 경로](../openstack/neutron-utils-execute.md)
