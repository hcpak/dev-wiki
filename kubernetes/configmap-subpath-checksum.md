# ConfigMap subPath 마운트와 checksum 애노테이션

> ConfigMap을 `subPath`로 얹은 파일은 내용이 바뀌어도 **파드 안에서 자동 갱신되지 않는다.**
> 그래서 차트는 파드 애노테이션에 설정 해시(`checksum/config`)를 박아 파드를 새로 띄우게 만든다.

## 왜 그렇게 동작하나

ConfigMap을 컨테이너에 얹는 방식은 두 가지다.

| 방식 | 동작 |
| --- | --- |
| 디렉터리로 통째 마운트 | ConfigMap 내용이 바뀌면 kubelet이 파드 안 파일을 **자동 갱신** |
| `subPath`로 파일 하나만 마운트 | **자동 갱신되지 않음** (쿠버네티스의 알려진 제약) |

설정 파일을 하나씩 얹는 차트는 후자다.

```yaml
volumeMounts:
  - name: app-config
    mountPath: /etc/<svc>/<svc>.conf
    subPath: <svc>.conf
  - name: app-config
    mountPath: /etc/<svc>/api-paste.ini
    subPath: api-paste.ini
```

파일을 하나씩 얹는 이유는 그 디렉터리에 이미지가 넣어둔 다른 항목을 디렉터리 마운트로
덮어버리지 않기 위해서다. 대신 자동 갱신을 포기하게 된다.

그래서 파드를 새로 띄워야 반영되고, 그것을 자동화한 것이 pod 템플릿의 애노테이션이다.

```yaml
template:
  metadata:
    annotations:
      checksum/config: '{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}'
```

configmap 템플릿을 렌더한 결과의 해시를 파드 스펙에 넣어둔다.

```
config 파일 내용 변경 → configmap 렌더 결과 변경 → 해시 변경 → 파드 스펙 변경
                    → Deployment 롤링 재생성 → 새 파드가 새 파일을 얹고 기동
```

즉 "subPath라서 자동 갱신은 안 되지만, checksum 애노테이션 덕에 파드가 새로 떠서 결과적으로 반영된다".

## 확인하는 방법

**① 어떤 방식으로 마운트됐는지** — 차트 템플릿에서 `subPath` 유무를 본다:

```bash
grep -A2 "mountPath" charts/<chart>/templates/deployment.yaml
```

**② checksum 애노테이션이 있는지**:

```bash
grep -n "checksum/config" charts/<chart>/templates/deployment.yaml
```

**③ 파드 안 파일이 실제로 무엇인지** (configmap과 다를 수 있다):

```bash
kubectl exec -n <ns> <pod> -- cat /etc/<svc>/<file>
kubectl get cm -n <ns> <cm-name> -o jsonpath='{.data.<file>}'
```

## 함정

- ⚠️ **checksum 애노테이션이 없는 차트에서 subPath를 쓰면 설정 변경이 조용히 무시된다.**
  ConfigMap은 갱신됐는데 파드 안 파일은 옛 내용이고, 아무 에러도 나지 않는다.
  설정을 고쳤는데 동작이 그대로일 때 여기를 먼저 의심할 것.
- 반대로 애노테이션이 있으면 **설정 한 줄만 바꿔도 파드가 롤링 재기동된다.** 무중단이긴 하지만
  "config만 고치는 PR"이 파드 재기동을 유발한다는 점은 리뷰·배포 협의에서 미리 밝혀야 한다.
- GitOps 자동 동기화(`selfHeal`, `prune`) 환경에서는 브랜치 머지 즉시 이 롤아웃이 일어난다.
  반영 시점을 고르고 싶으면 수동 sync 환경과 순서를 맞춰야 한다.
- 💡 GitOps가 관리하지 않는(수동 helm) 환경은 머지만으로는 아무 일도 일어나지 않는다.
  별도로 `helm upgrade`를 돌려야 새 설정이 반영된다.
