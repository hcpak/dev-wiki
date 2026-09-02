---
status: 활성
verified: 2026-09-02
---
# pull --rebase 는 미푸시 로컬 커밋을 원격 위에 "재생성" 한다

> `git pull` 은 fetch + 합치기의 묶음이고, 합치는 방식이 merge(기본)냐 rebase 냐가 갈림길이다.
> `--rebase` 는 내 미푸시 커밋을 잠시 떼어내 원격 최신 커밋 위에 다시 만든다 — 이동이 아니라
> **새 커밋 생성**이라 해시가 바뀐다. 병합 커밋 없이 히스토리가 일렬로 유지된다.

## 왜 그렇게 동작하나

- `[문서]` `git pull` = `git fetch` + 통합. 통합 방식 기본값은 merge 이고, `--rebase` 를 주면
  `git rebase origin/<branch>` 로 대체된다. `git fetch && git rebase origin/main` 과 등가다.
- `[실측]` 원격과 로컬이 공통 조상에서 갈라진 상태(원격 5커밋 / 로컬 1커밋)에서 push 하면
  `! [rejected] ... (fetch first)` 로 거부된다. 원격이 가진 커밋을 로컬이 모르기 때문.
- `[실측]` rebase 는 로컬 커밋을 **재생성**한다. 같은 커밋 메시지의 해시가 rebase 전
  `394c88b` → 후 `7234146` 으로 바뀐 것이 reflog 에 남는다 (아래 출력).
- `[문서]` merge 로 합쳤다면 두 갈래가 남고 병합 커밋이 하나 생긴다. rebase 는
  "상대 것을 먼저 깔고 내 것을 위에 얹는" 방식이라 결과가 일직선이다.
- `[코드]` 다시 쓰이는 범위는 `origin/<branch>..HEAD`, 즉 **원격에 없는 로컬 커밋뿐**이다.
  이미 푸시된 커밋은 건드리지 않으므로 다른 클론과 히스토리가 어긋날 일이 없다.
  (반대로, 푸시된 커밋을 수동 rebase 하면 어긋난다 — 그래서 pull --rebase 는 안전한 방향으로만 동작)

## 확인하는 방법

```
$ git push origin main
 ! [rejected]        main -> main (fetch first)      # 갈라진 상태

$ git fetch && git rebase origin/main
Successfully rebased and updated refs/heads/main.

$ git reflog --date=short | head -4
7234146 HEAD@{2026-09-02}: rebase (finish): returning to refs/heads/main
7234146 HEAD@{2026-09-02}: rebase (pick): 프로젝트 메모리 이관 …
84a7972 HEAD@{2026-09-02}: rebase (start): checkout origin/main
394c88b HEAD@{2026-09-02}: commit: 프로젝트 메모리 이관 …   # 같은 커밋, 다른 해시
```

`git log --oneline` 에 병합 커밋 없이 로컬 커밋이 원격 커밋들 위에 한 줄로 얹혀 있으면 성공.

## 이 노트가 틀렸다면

- pull --rebase 후 `git log` 에 부모가 둘인 병합 커밋이 보인다 → rebase 로 동작하지 않은 것.
  `pull.rebase` / `branch.<name>.rebase` 설정과 실제 실행 명령을 다시 확인해야 한다
- 갈라진 상태에서 rebase 를 통과한 커밋의 해시가 그대로다 → "재생성" 주장이 틀린 것
  (단, 로컬 커밋이 0개면 fast-forward 라 재생성 자체가 없다 — 적용 범위 참고)
- 이미 푸시된 커밋의 해시가 pull --rebase 로 바뀌는 것이 관측된다 → "미푸시분만 다시 쓴다"
  주장이 틀린 것

## 적용 범위

- 원격과 로컬이 **갈라진(diverged)** 상태 기준. 로컬 신규 커밋이 없으면 그냥 fast-forward 라
  재생성이 일어나지 않는다
- git 2.x 일반 동작. 실측은 2026-09 macOS 로컬 저장소에서

## 함정

- ⚠️ rebase 중 충돌은 병합처럼 한 번에 나지 않고 **커밋을 하나씩 다시 얹는 시점마다** 난다.
  해결 후 `git rebase --continue`, 포기는 `git rebase --abort`
- ⚠️ 해시가 바뀌므로, 리베이스된 커밋 해시를 다른 곳(메모·이슈)에 이미 적어뒀다면 낡은 참조가 된다
- 💡 혼자 여러 기기에서 같은 브랜치에 커밋하는 동기화용 저장소라면 병합 커밋은 노이즈다 —
  push 전 `git pull --rebase` 를 습관으로 두면 히스토리가 일렬로 유지된다
- 💡 원래 해시는 사라지지 않는다 — reflog 에 남아 있어 `git reset --hard <옛 해시>` 로 되돌릴 수 있다

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-09-02 | 최초 작성. 두 기기 동기화 저장소에서 push 거부 → fetch+rebase → push 성공 전 과정 실측, reflog 로 해시 변경 확인 | 핵심 주장 `[실측]` |

## 관련

- [[squash-merge-produces-a-single-parent-commit]] — 히스토리를 일직선으로 만드는 또 다른 방식
- [[push-refspecs-succeed-independently]]
