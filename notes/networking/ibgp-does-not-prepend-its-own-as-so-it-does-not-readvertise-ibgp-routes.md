---
status: 활성
verified: 2026-11-05
---
# iBGP 는 AS_PATH 에 자기 AS 를 붙이지 않는다 — 그래서 iBGP 로 받은 경로를 다른 iBGP 상대에게 다시 광고하지 않는다

> iBGP 는 AS 번호가 같은 장비끼리 맺는 BGP 세션이다(다르면 eBGP).
> 같은 AS 안에서는 AS_PATH 에 자기 AS 를 붙이지 않으므로, AS_PATH 로 루프를 잡을 수 없다.
> 그래서 표준은 "iBGP 로 받은 경로는 다른 iBGP 상대에게 넘기지 않는다" 는 규칙으로 루프를 막는다.
> 결과적으로 ToR 둘과 iBGP 를 맺은 서버는 한쪽에서 받은 경로를 다른 쪽에 광고하지 않고, 두 ToR 사이의 중계 장비가 되지 않는다.

## 왜 그렇게 동작하나

- `[문서]` eBGP 상대에게 경로를 넘길 때만 AS_PATH 앞에 자기 AS 를 붙인다(RFC 4271 5.1.2). iBGP 상대에게는 받은 AS_PATH 를 그대로 넘긴다.
- `[문서]` eBGP 는 받은 AS_PATH 에 자기 AS 가 있으면 버려서 루프를 막는다. iBGP 는 자기 AS 가 붙지 않으니 이 검사가 통하지 않는다.
- `[문서]` 그래서 내부 상대에게 받은 경로는 다른 내부 상대에게 광고하지 않는다(RFC 4271 9.2). 이 제한을 풀려면 route reflector(RFC 4456) 나 confederation 을 설정해야 한다.
- `[문서]` LOCAL_PREF(같은 AS 안에서 쓰는 경로 선호도. 높을수록 선호)는 iBGP 상대에게만 보낸다(RFC 4271 5.1.5). AS_PATH 길이보다 먼저 비교한다. 기본값 100 은 RFC 가 아니라 BIRD 같은 구현의 기본값이다.
- `[문서]` iBGP 는 NEXT_HOP 을 기본적으로 바꾸지 않는다(RFC 4271 5.1.3). 그래서 외부에서 받은 경로의 NEXT_HOP 이 내부 장비가 모르는 주소로 남을 수 있다. 이를 피하려고 경계 장비에 next-hop-self 를 설정하는 경우가 흔하다.

## 확인하는 방법

BIRD 노드(자기 AS = ToR AS, 이웃 둘, `export all`, route reflector 설정 없음)에서:

```
$ birdc show protocols all <peer1> | grep -E 'Neighbor AS|Local AS'
    Neighbor AS:      <AS 같음>
    Local AS:         <AS 같음>

$ birdc show route for <받은 대역> all
<받은 대역>  via <ToR1 링크 IP> on <uplink0> [peer1 ...] * (100) [AS<바깥 AS>i]
        BGP.as_path: <바깥 AS>
        BGP.next_hop: <ToR1 링크 IP>
        BGP.local_pref: 100
                     via <ToR2 링크 IP> on <uplink1> [peer2 ...] (100) [AS<바깥 AS>i]

$ birdc show protocols all <peer1> | grep Routes
  Routes:         21 imported, 3 exported, 21 preferred
$ birdc show protocols all <peer2> | grep Routes
  Routes:         21 imported, 3 exported, 0 preferred

$ birdc show route protocol <peer1> primary count
21 of 24 routes for 24 networks      ← peer1 에게 받은 경로 21개가 모두 best

$ birdc show route export <peer2> count
3 of 45 routes for 24 networks       ← 자기 lo /32 1개 + static 2개뿐
$ birdc show route export <peer1> count
3 of 45 routes for 24 networks
```

- `[실측]` AS_PATH 에 ToR 의 AS(= 자기 AS)가 없고 바깥 AS 만 있다.
- `[실측]` peer1 에게 받은 best 경로 21개는 `export all` 인데도 peer2 로 나가지 않았다. peer2 로 나간 것은 노드 자신의 경로 3개뿐이다. best 가 아니라서 빠진 것이 아니므로, iBGP 재광고 금지가 걸린 것이다.
- 💡 `show route export <peer1>` 만 보면 이 규칙을 입증하지 못한다. peer1 쪽이 best 라서, 받은 상대에게 되돌려 보내지 않는 규칙만으로도 빠진다. best 를 준 상대가 아닌 **다른 상대** 쪽 export 를 봐야 한다.
- `[추정]` 받은 NEXT_HOP 이 ToR 자신의 링크 IP 인 것은 ToR 에 next-hop-self 가 설정돼 있기 때문으로 보인다. ToR 설정은 보지 못했다.

## 이 노트가 틀렸다면

- route reflector 설정 없이, peer1 에게 받은 best 경로가 `birdc show route export <peer2>` 에 보인다 → 이 노트가 틀렸거나, 그 BIRD 버전이 다른 규칙을 쓴다.
- 자기 AS 와 같은 AS 의 이웃에게 받은 경로의 AS_PATH 에 그 이웃의 AS 가 들어 있다 → 세션이 iBGP 가 아니라 eBGP 이거나 AS 설정이 다르다.
- ToR 설정에 next-hop-self 가 없는데 NEXT_HOP 이 ToR 링크 IP 다 → 위 `[추정]` 이 틀렸다. 그 경로가 ToR 자신이 만든 경로이거나 다른 정책 때문이다.

## 적용 범위

- RFC 4271 BGP-4. BIRD 1.6.8 노드에서 `[실측]`.
- 재광고 금지를 푸는 수단은 route reflector 와 confederation 이다. route reflector 나 confederation 을 쓰는 구성은 재광고 규칙이 다르다.
- `add-path`(RFC 7911)는 한 대역에 경로를 여러 개 실어 보내게 할 뿐이고, 이 금지를 풀지 않는다 `[문서]`.

## 함정

- ⚠️ `export all` 이라고 bird 테이블의 모든 경로가 나가는 것은 아니다. iBGP 재광고 금지는 `export all` 과 별개로 걸린다.
- ⚠️ iBGP 로 맺은 서버에서 AS_PATH 만 보면 ToR 을 거친 사실이 드러나지 않는다. AS_PATH 는 AS 경계를 센 것이지 장비 수가 아니다.
- 💡 "왜 eBGP 가 아니라 iBGP 로 설계했나" 는 이 노트가 답하지 않는다. 설계 의도는 확인하지 못했다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-11-05 | 최초 작성. ToR 둘과 iBGP 를 맺은 BIRD 1.6.8 노드에서 `birdc show protocols all`, `show route ... all`, 양쪽 `show route export` 조회 | AS_PATH 는 `[실측]`. 재광고 금지는 peer1 best 21개가 peer2 export 에 없는 것으로 `[실측]`. next-hop-self 는 `[추정]` |

## 관련

- [[next-hop-is-written-only-into-the-ethernet-destination]]
- [[anycast-selects-a-destination-it-does-not-replicate]]
