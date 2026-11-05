---
status: 활성
verified: 2026-11-05
---
# next-hop IP 는 패킷에 실리지 않는다 — ARP 로 바꾼 MAC 이 Ethernet 목적지에만 들어간다

> 라우팅 테이블의 `via <next-hop>` 은 "이 패킷을 다음에 넘길 장비" 를 정할 뿐이다.
> 커널은 next-hop IP 를 ARP 로 MAC 으로 바꿔 **Ethernet 목적지에만** 쓰고, IP 헤더의 목적지는 끝까지 최종 목적지로 둔다.
> next-hop 을 IP 목적지로 착각하면 "다음 장비의 IP 를 패킷 어딘가에 넣어야 하지 않나" 라는 질문에서 막힌다.
> next-hop 이 패킷에 값으로 실리는 것은 BGP UPDATE 의 NEXT_HOP 처럼 **경로 정보를 페이로드 데이터로 보낼 때**다. IP 헤더가 아니라 TCP 페이로드 안이다.

## 왜 그렇게 동작하나

- `[문서]` 라우터는 받은 패킷의 IP 목적지로 자기 라우팅 테이블을 다시 조회한다. 그래서 IP 목적지가 바뀌면 다음 장비는 그 패킷을 "자기 앞으로 온 것" 으로 받아 버리고, 원래 목적지로 보내지 않는다.
- `[문서]` 그래서 다음 장비를 가리키는 일은 링크 계층이 맡는다. 같은 링크 위의 장비는 MAC 으로 구분되므로, next-hop IP 는 ARP(IPv4) 로 MAC 을 알아내는 데만 쓰인다.
- `[실측]` 커널 경로 `via <ToR 링크 IP>` 인 패킷을 캡처하면 Ethernet 목적지는 ToR 의 MAC 이고, IP 목적지는 최종 목적지 그대로다(아래 출력).
- `[문서]` 반대로 BGP 의 NEXT_HOP 은 UPDATE 메시지의 **경로 속성(데이터)** 이다. "이 대역으로 가려면 이 IP 로" 를 알려 줄 뿐이고, 받은 쪽은 이 값으로 커널 경로의 `via` 를 만든다.

| | 제어 패킷 (BGP UPDATE, 이웃 → 나) | 지나가는 패킷 (나 → 먼 목적지) |
| --- | --- | --- |
| IP 목적지 | 세션을 맺은 나의 주소 (링크 주소로 맺었으면 링크 IP). 받는 장비가 곧 목적지 | 최종 목적지. 끝까지 바뀌지 않음 |
| Ethernet 목적지 | 나의 MAC | next-hop 을 ARP 로 푼 MAC |
| next-hop 이 있는 곳 | UPDATE 의 NEXT_HOP 속성 | 패킷에는 없음. 커널 경로 → ARP → Ethernet 목적지 |

비유: IP 목적지는 송장의 받는 사람 주소(끝까지 그대로)이고, Ethernet 목적지 MAC 은 다음 집하장(구간마다 새로 적음)이다.

## 확인하는 방법

```
$ ip route get <먼 목적지>
<먼 목적지> via <next-hop> dev <uplink> src <내 링크 IP>

$ ip -4 neigh show dev <uplink>
<next-hop> lladdr <next-hop MAC> REACHABLE

# 다른 터미널에서 ping -c1 <먼 목적지>
$ tcpdump -e -n -c 1 -i <uplink> icmp and dst host <먼 목적지>
<내 MAC> > <next-hop MAC>, ethertype IPv4, length 98:
    <내 링크 IP> > <먼 목적지>: ICMP echo request
```

- Ethernet 목적지가 `ip neigh` 의 next-hop MAC 과 같다.
- IP 목적지는 next-hop 이 아니라 먼 목적지다.
- 목적지가 응답하지 않아도 송신 캡처는 된다. 캡처 시점의 의미는 [[tx-capture-is-not-proof-of-transmission]].

## 이 노트가 틀렸다면

- 터널·NAT 없는 평범한 IP 포워딩에서, 송신 캡처의 IP 목적지가 next-hop IP 로 찍힌다 → 이 노트가 틀렸다.
- Ethernet 목적지가 `ip neigh` 의 next-hop MAC 과 다르다 → next-hop 이 아닌 다른 경로(정책 라우팅, 다른 테이블)를 탄 것이다. 먼저 `ip route get` 결과를 다시 확인한다.

## 적용 범위

- IPv4 이더넷 링크. Linux 노드가 직접 보낸 패킷에서 `[실측]`. 받아서 넘기는 transit 포워딩은 `[문서]` 이고 직접 캡처하지 않았다.
- IPv6 는 ARP 대신 NDP 를 쓰지만 같은 구조다 `[문서]`.
- 예외: IPv4 source routing 옵션(LSRR/SSRR)과 IPv6 Routing Header(SRv6)는 거쳐 갈 주소를 실제로 IP 헤더에 넣는다 `[문서]`.
- 터널(VXLAN·GRE·IPsec)이나 NAT 를 거치면 바깥 헤더의 IP 목적지가 다른 값이 된다. 그것은 next-hop 을 IP 에 넣은 것이 아니라 패킷을 한 번 더 감싼 것이다.

## 함정

- ⚠️ BGP 를 배우면서 "BGP 패킷의 IP 목적지는 이웃 IP" 를 서비스 패킷에 그대로 옮겨 적용하기 쉽다. 이웃 IP 가 IP 목적지에 오는 것은 이웃 **자신에게 보내는** 제어 패킷뿐이다.
- ⚠️ LB 처럼 VIP 나 터널 주소를 쓰는 서비스 트래픽은 출발지가 링크 IP 와 다를 수 있다. 위 tcpdump 출력의 출발지는 노드가 직접 보낸 시험 패킷 기준이다.
- 💡 "정말 첫 장비가 그 next-hop 인가" 는 `ping -c1 -t 1 <먼 목적지>` 로 본다. TTL 을 0 으로 깎은 첫 장비가 `Time to live exceeded` 를 돌려주며 자기 주소를 드러낸다. 응답 출발지는 장비 설정에 따라 next-hop IP 가 아니라 loopback 같은 다른 주소일 수 있다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-11-05 | 최초 작성. BGP 이웃이 둘인 Linux 노드에서 `ip route get`·`ip neigh`·`tcpdump -e` 로 ping 1회 캡처 | `[실측]` — Ethernet 목적지 = next-hop MAC, IP 목적지 = 최종 목적지. `ping -t 1` 의 TTL exceeded 응답 출발지가 `ip route get` 의 next-hop IP 와 같았다 |

## 관련

- [[tx-capture-is-not-proof-of-transmission]]
- [[ibgp-does-not-prepend-its-own-as-so-it-does-not-readvertise-ibgp-routes]]
- [[vpn-encapsulates-ip-packets-vxlan-encapsulates-ethernet-frames]]
