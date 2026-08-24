# Prometheus 라벨 값 이스케이프 — 순서가 중요하고, 한 줄 깨지면 스크레이프 전체가 버려진다

> 라벨 값은 큰따옴표로 감싸이므로 값 안의 `"`가 값을 조기 종료시킨다.
> 이스케이프는 **역슬래시 → 큰따옴표 → 개행** 순서로 해야 하고, 이는 인코딩과 무관한 **출력 문법** 문제다.
> 게다가 소비자(telegraf 등)는 깨진 한 줄만 버리는 게 아니라 **응답 전체를 폐기**하므로, 리소스 하나의 이름이 그 exporter의 모든 지표를 유실시킨다.

**상태**: 활성
**마지막 검증**: 2026-08-19

## 왜 그렇게 동작하나

`[문서]` Prometheus text exposition format의 한 줄은 이런 모양이다.

```
resource_info{resource_id="abc", resource_name="내 서비스", tenant_id="t1"} 1
```

값이 `"`로 감싸이므로, 사용자가 지은 리소스 이름에 `"`가 들어 있으면 값이 거기서 끊기고 뒤가
문법 오류가 된다. 개행이 들어가면 한 줄이 두 줄로 쪼개져 다음 줄이 별개 메트릭으로 파싱된다.
따라서 `\`, `"`, 개행 세 가지를 이스케이프해야 한다.

`[실측]` 파서는 한 줄 단위로 관대하지 않다 — telegraf(inputs.prometheus)는 파싱 에러가 나면
그 라인만 건너뛰지 않고 **그 스크레이프 응답 전체를 버린다.** exporter가 200으로 멀쩡히 응답해도
소비 측에서 전량 유실되므로, "exporter는 정상인데 지표가 안 온다"는 형태로 나타난다.

### 순서가 중요한 이유

`[실측]` 이스케이프는 **역슬래시를 먼저 두 배로** 만들어야 한다. 나중에 하는 치환들이 자기 백슬래시를
새로 넣기 때문에, 역슬래시 치환을 마지막에 하면 **앞 단계가 넣은 백슬래시까지 다시 이스케이프**한다.

```
입력:  내"이름
잘못된 순서 ("  →  \)
  1) " → \"      →  내\"이름
  2) \ → \\      →  내\\"이름      ← 앞 단계가 넣은 \ 가 두 배가 되어 " 가 다시 노출됨
올바른 순서 (\  →  ")
  1) \ → \\      →  내"이름        (역슬래시 없음, 변화 없음)
  2) " → \"      →  내\"이름       ✅
```

py2에서 쓴 구현(값이 bytes로 들어올 수 있으므로 경계에서 decode까지 함께 처리):

```python
def escape_label_value(value):
    if not isinstance(value, basestring):
        value = unicode(value)
    elif isinstance(value, str):
        value = value.decode('utf-8', 'replace')
    return (value.replace(u'\\', u'\\\\')
                 .replace(u'"', u'\\"')
                 .replace(u'\n', u'\\n'))
```

## 확인하는 방법

노출 결과를 직접 본다 — 깨진 라벨은 exporter 출력과 소비자 로그 양쪽에서 확인된다.

```bash
$ curl -s http://<host>:<port>/metrics | grep -n '\\|"' | head
# 이스케이프 안 된 예 (값 안의 " 와 \ 가 날것으로 노출):
resource_info{resource_id="...", resource_name="lb-한글-say"quoted"-back\slash-..."} 1
```

소비자 쪽(telegraf 로그, 매 수집 주기 반복):

```
E! [inputs.prometheus] Error in plugin: error reading metrics for "http://<host>:<port>/metrics":
   decoding response failed: text format parsing error in line NNNN: unexpected end of label value "lb-한글-say"
```

이때 exporter의 다른 정상 라인들도 함께 유실됐는지는 소비자 저장소(TSDB) 쪽에 해당 구간
데이터 포인트가 있는지로 확인한다.

## 이 노트가 틀렸다면

- 이스케이프 없는 `"` 포함 라벨을 노출했는데 telegraf가 그 라인만 버리고 나머지 지표는 정상 적재된다 → "응답 전체 폐기" 주장이 틀렸거나 telegraf 버전·설정(`metric_version` 등)에 따라 달라진다는 뜻
- `\ → " → 개행` 순서가 아닌 구현이 모든 입력에서 올바른 출력을 낸다 → 순서 주장이 틀렸다는 뜻 (단, 치환이 아니라 정규식 단일 패스 구현이면 순서 자체가 무의미해질 수 있다 — 적용 범위 축소)
- 한글 등 비ASCII가 이스케이프 없이 파싱 에러를 낸다 → "한글은 이스케이프 대상이 아니다" 주장이 틀렸다는 뜻

## 적용 범위

- Prometheus text exposition format (0.0.4) 기준. OpenMetrics·protobuf 노출은 별도.
- "응답 전체 폐기"는 telegraf 1.31 inputs.prometheus(`metric_version = 2`)에서 실측. 다른 소비자(prometheus 서버 자체 스크레이프 등)의 관용도는 미확인 — `[추정]` 비슷하게 전체를 버릴 것.

## 함정

- ⚠️ **인코딩 문제와 섞어 보지 말 것.** 한글은 이스케이프 대상이 아니다(UTF-8 그대로 유효).
  py2 유니코드 에러(→ [py2 유니코드 대칭](../python/py2-unicode-symmetry.md))는 값을 **만들다** 터지는 문제이고,
  이스케이프는 값을 **출력하는** 문법 문제다. py2/py3 공통으로 필요하다.
- ⚠️ 파급 반경이 "그 리소스"가 아니라 "그 exporter를 스크레이프하는 단위 전체"다. 피해자와 원인 리소스가
  다른 것이라, 정상 리소스의 지표 유실로 제보되면 같은 노드의 **다른** 리소스 이름부터 의심할 것.
- 라벨 값을 만드는 곳이 여러 군데면 전부 같은 헬퍼를 태워야 한다. 리소스 정보 라벨, 지표 라인 라벨,
  다른 드라이버가 만드는 라벨까지 각각 반영이 필요했다.
- 💡 이름은 사용자가 짓는 값이므로 "설마 이런 문자를"이 통하지 않는다. 라벨에 들어가는 사용자 입력은
  기본적으로 이스케이프를 거치게 두는 편이 안전하다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-05 | 최초 작성 (py2 exporter 수정 작업에서 순서 버그 재현) | `[실측]` 순서 주장 확립 |
| 2026-08-19 | 미수정 exporter가 있는 노드에서 `"`·`\` 포함 이름의 실사고 관측 — telegraf가 매분 파싱 에러를 내며 응답 전체를 폐기, 같은 노드의 정상 리소스 지표까지 3일간 유실. 원인 리소스 삭제 즉시 복구 확인 | 유지 + "응답 전체 폐기" `[실측]` 승격, 템플릿 형식 전환 |

## 관련

- [py2 유니코드 — Encode/Decode의 대칭](../python/py2-unicode-symmetry.md)
