# 비관리자 컨텍스트의 조회는 `tenant_id` 로 자동 필터된다 — 행이 있어도 NotFound 가 된다

> neutron 의 model_query 는 컨텍스트가 admin 이 아니고 모델에 `tenant_id` 속성이 있으면 `WHERE tenant_id = <요청자>` 를 **호출부에 안 보이게** 덧붙인다. 그래서 다른 테넌트가 소유한 행은 DB 에 멀쩡히 있어도 0 행으로 조회되고, `get_by_id` 계열은 그대로 NotFound 예외가 된다. 같은 코드가 admin 토큰이면 성공하고 프로젝트 토큰이면 실패한다.

**상태**: 활성
**마지막 검증**: 2026-08-28

## 왜 그렇게 동작하나

`[코드]` `neutron/db/_model_query.py` 의 쿼리 빌더가 기본 필터를 만든다.

```python
if ndb_utils.model_query_scope_is_project(context, model):
    if hasattr(model, 'rbac_entries'):
        query_filter = ((model.tenant_id == bindparam('tenant_id')) | <rbac 공유 조건>)
    elif hasattr(model, 'shared'):
        query_filter = ((model.tenant_id == bindparam('tenant_id')) | (model.shared == sql.true()))
    else:
        query_filter = (model.tenant_id == bindparam('tenant_id'))
    params['tenant_id'] = context.tenant_id
```

- `[코드]` 판정은 **모델에 `tenant_id` 속성이 있는가** 와 **컨텍스트가 admin 이 아닌가** 두 가지뿐이다. 모델이 `HasProject` 믹스인을 안 썼어도 컬럼만 있으면 걸린다.
- `[코드]` `shared` 나 `rbac_entries` 가 있는 모델은 OR 조건으로 완화된다. 둘 다 없는 모델은 **순수 테넌트 일치**만 통과한다.
- `[코드]` `get_by_id()` 는 이 쿼리에 `.one()` 을 붙인다. 0 행이면 `NoResultFound` 가 나고, 플러그인은 보통 이것을 자기 NotFound 예외로 변환한다. 호출부에는 "그런 리소스 없음" 으로만 보인다.
- `[코드]` `is_admin` 은 정책 파일의 `context_is_admin` 규칙(대개 `role:admin`)으로 정해진다. 프로젝트 단위 관리자 롤은 여기 해당하지 않아, 콘솔을 쓰는 일반 사용자와 똑같이 필터를 받는다.

핵심은 **비대칭이 생기는 지점**이다. 한 리소스가 두 테이블에 나뉘어 있고 한쪽은 소유자를 네트워크에서 물려받고 다른 쪽은 만든 주체를 기록하면, 두 값이 갈라질 수 있다.

| | 소유자(tenant) 가 정해지는 방식 |
| --- | --- |
| 포트 행 | 붙어 있는 네트워크의 소유자를 그대로 물려받음 |
| 그 포트가 참조하는 상위 리소스 행 | **만든 주체**의 컨텍스트 |

관리 주체가 사용자 네트워크에 붙는 리소스를 대신 만들면 두 값이 어긋나고, 사용자 토큰으로는 상위 리소스만 안 보인다.

## 확인하는 방법

`[실측]` 같은 리소스를 **토큰만 바꿔** 두 번 요청하면 갈린다.

```
$ <프로젝트 토큰> <상위 리소스 조회를 유발하는 API>
<NotFound: 그런 리소스 없음>

$ <admin 토큰> <같은 API>
(정상 응답)
```

`[실측]` DB 에서 두 소유자 컬럼을 대조하면 대상 행을 특정할 수 있다.

```sql
SELECT p.id, p.project_id AS port_tenant, t.tenant_id AS upper_tenant
  FROM ports p LEFT JOIN <상위 리소스 테이블> t ON t.id = p.device_id
 WHERE p.device_owner = '<vendor>:transit';
```

검증 환경에서 해당 device_owner 포트 15건 중 **1건**만 두 값이 달랐다. 나머지는 사용자가 직접 만든 것이라 일치했다.

## 이 노트가 틀렸다면

- **admin 토큰으로도 같은 NotFound 가 나면** → 테넌트 스코프가 아니라 행이 실제로 없는 것이다. 원인 분석을 다시 해야 한다.
- **모델에 `tenant_id` 컬럼이 없는데도 필터가 걸리면** → 판정 조건이 이 노트에 적힌 것과 다르다.
- **`is_admin` 인 컨텍스트에서도 필터가 걸리면** → 정책의 `context_is_admin` 규칙이 예상과 다르거나, 플러그인이 자체 필터를 더 붙이고 있다.
- **모델에 `shared` 컬럼이 있는데도 공유 행이 안 보이면** → OR 완화 경로가 그 버전에는 없다.

## 적용 범위

- neutron / neutron-lib 계열의 `model_query` 를 쓰는 플러그인. 자체 세션으로 직접 쿼리를 짜는 코드에는 해당하지 않는다.
- `.one()` 을 쓰는 단건 조회. 컬렉션 조회는 예외 없이 빈 목록이 되므로 증상이 "없음" 으로 조용히 나타난다 — 더 찾기 어렵다.

## 함정

- ⚠️ **개발·검증에서는 거의 안 밟힌다.** 보통 자기 프로젝트에서 만들고 자기 토큰으로 시험하므로 두 소유자가 항상 일치한다. 운영에서 관리 주체가 대신 만든 리소스가 섞였을 때 처음 드러난다.
- ⚠️ **에러 메시지가 "없다" 고 말해서 데이터 문제로 오진하기 쉽다.** 실제로는 권한 범위 문제다. DB 를 직접 보면 행이 있으므로, 메시지와 관측이 어긋날 때 이 경로를 의심한다.
- 💡 관리 주체가 대신 만드는 API 라면, 생성 시점에 소유 테넌트를 사용자 쪽으로 맞춰 두는 구현이 있는지 먼저 본다. 그렇게 해 두는 경로가 있으면 그 리소스는 이 문제를 안 겪는다.

## 처방

- 소유권 판정이 아니라 **구조 판정**이 목적인 조회라면 승격된 컨텍스트로 조회한다. "이 포트가 어떤 종류의 게이트웨이에 붙어 있나" 는 요청자가 누구든 같은 답이어야 한다.
- 그게 과하면, 조회 실패를 "해당 종류가 아님" 으로 정규화해 호출부가 예외를 안 보게 한다. 판정 결과는 어차피 같다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-28 | 쿼리 빌더 코드 확인 + 검증 환경에서 토큰만 바꿔 재현·대조 | 필터 조건 `[코드]`, 토큰별 결과 차이 `[실측]` |

## 관련

- [조회 API 의 "없음"이 None 인지 예외인지는 시그니처에 없다](../software-design/absence-contract-none-or-exception.md) — 이 필터가 만들어낸 NotFound 를 호출부가 어떻게 잘못 받는가
- [잘못 배포된 자격증명이 유효하면 실패가 아니라 남의 데이터가 된다](../api/valid-credential-wrong-tenant-succeeds.md) — 자격증명은 유효한데 대상 테넌트가 달라 생기는 반대 방향 사고
