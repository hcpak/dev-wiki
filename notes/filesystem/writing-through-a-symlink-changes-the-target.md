---
status: 활성
verified: 2026-09-09
---
# 심볼릭 링크에 쓰면 원본이 바뀌고, 이름을 바꾸면 링크만 옮겨진다

> 링크 경로를 열어 쓰는 동작은 전부 링크를 따라가 원본 파일의 내용을 바꾼다. 셸 리다이렉션, `cp`, Node 의 writeFile 에서 실측했다. 다른 런타임도 같은 `open(2)` 를 쓰므로 같을 것으로 본다. 반대로 `mv` 와 `ln -sfn` 은 링크 엔트리 자체를 다루므로 원본은 그대로다. 그래서 "설정 파일을 심볼릭으로 공유" 하면, 그 파일을 자기 것으로 여기고 다시 쓰는 도구가 정본을 덮어쓴다.

## 왜 그렇게 동작하나

- `[문서]` `open(2)` 는 기본적으로 심볼릭 링크를 따라간다. `O_NOFOLLOW` 를 주지 않는 한, 경로를 열어 쓰는 API 는 모두 원본 inode 에 쓴다. `rename(2)`, `unlink(2)`, `symlink(2)` 는 디렉터리 엔트리를 다루는 호출이라 링크를 따라가지 않는다. 출처: macOS·Linux `open(2)` man page 의 `O_NOFOLLOW` 절, POSIX `rename(2)` ("If the old argument points to a symbolic link, the link itself is renamed").
- `[실측]` 셸 `>`, `cp`, Node `fs.writeFileSync` 는 링크 경로에 쓰면 원본 내용이 바뀐다. `mv` 는 링크를 옮기고, 옮긴 자리에 새로 만든 파일은 일반 파일이 되어 링크가 조용히 끊긴다. `ln -sfn` 은 링크 대상만 바꾼다.
- `[코드]` oh-my-codex 0.21 의 setup 코드에는 자기 관리 파일을 다시 쓰는 경로가 둘 있다. 덮어쓰기 경로는 백업을 `copyFile` 로 만든 뒤 `writeFile` 하므로 원본을 바꾼다. 백업 경로는 기존 파일을 `rename` 하고 새 파일을 만들므로 링크를 끊는다. 어느 쪽이든 "링크로 공유한 정본" 은 살아남지 못한다. 다른 설치 도구도 이 둘 중 하나일 것으로 `[추정]` 한다 — 확인한 도구는 이것 하나다.

## 확인하는 방법

```
$ echo ORIGINAL > target.txt; ln -s target.txt link.txt

$ echo "WRITTEN VIA LINK" > link.txt; cat target.txt
WRITTEN VIA LINK

$ mv link.txt link.bak; cat target.txt; ls -l link.bak
WRITTEN VIA LINK
lrwxr-xr-x  link.bak -> target.txt

$ echo NEW > link.txt; ls -l link.txt        # 옛 링크 자리에 만든 파일은 일반 파일
-rw-r--r--  link.txt

$ ln -s target.txt link2.txt
$ node -e "require('fs').writeFileSync('link2.txt','NODE WROTE')"; cat target.txt
NODE WROTE

$ echo COPIED > src.txt
$ cp src.txt link2.txt; cat target.txt       # cp 도 따라간다
COPIED

$ ln -sfn other.txt link2.txt; cat target.txt   # 링크 대상만 바뀜, 원본 그대로
COPIED
```

## 이 노트가 틀렸다면

- 링크 경로에 `>` 로 썼는데 원본이 그대로고 링크 자리에 일반 파일이 생겼다 → 그 도구가 "임시 파일에 쓰고 rename" 하는 원자적 저장을 한 것이다. 주장은 "경로를 열어 쓰는 API" 에만 해당한다는 뜻이고, 그런 도구 목록을 함정에 추가한다.
- `cp` 로 링크 경로에 덮어썼는데 원본은 그대로고 링크가 일반 파일로 바뀌었다 → 그 `cp` 구현이 대상 링크를 지우고 새로 만드는 동작(GNU `--remove-destination` 류)을 기본으로 하는 것이다. 적용 범위를 "링크를 따라가는 cp" 로 좁힌다.
- Linux 나 네트워크 파일시스템에서 결과가 다르다 → 적용 범위에 조건을 추가한다.

## 적용 범위

macOS 24.6, APFS, zsh, Node 24 에서 실측했다. POSIX 의 open/rename 의미론에 기반하므로 Linux 에서도 같을 것으로 `[추정]` 한다. Linux 에서는 돌려 보지 않았다.

## 함정

- ⚠️ 설정 파일을 심볼릭으로 여러 도구에 공유할 때, 그중 하나가 그 파일을 자기 것으로 다시 쓰는 도구면 공유가 아니라 덮어쓰기 통로가 된다. 그 도구는 마커 없는 파일을 "다른 도구가 덮어썼다" 고 보고 재생성을 권하기까지 한다. 정본은 심볼릭이 아니라 복사와 동기화로 나눈다.
- ⚠️ 링크가 끊긴 뒤에는 같은 이름의 일반 파일이 그 자리에 있어 알아채기 어렵다. `ls -l` 에서 `->` 가 남아 있는지 본다.
- 💡 링크 경로의 쓰기 권한은 원본 inode 의 권한을 따른다. 원본을 읽기 전용으로 두면 링크 경로의 쓰기도 실패할 것이다. `[추정]` — 권한 조합을 실측하지 않았다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-09-09 | 최초 작성. macOS 에서 `>`, `mv`, 새 파일 생성, Node writeFile, `cp`, `ln -sfn` 여섯 조작 실측 | `[실측]` |

## 관련

- [birthtime 은 "언제 만들어졌나"를 알려주지 않는다](birthtime-is-not-creation-identity.md) — 원자적 교체가 새 inode 를 만든다는 이웃 주제. 위 반증 조건 첫 항목의 "임시 파일에 쓰고 rename" 이 그 경우다.
