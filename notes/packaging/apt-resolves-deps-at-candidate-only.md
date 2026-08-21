# apt 는 의존성을 후보 버전으로만 자동 해석한다 — 핀 세트는 저장소와 함께 늙는다

> `pkg=버전` 명시는 **그 패키지에만** 적용된다. 의존성은 언제나 후보(candidate, 보통 저장소의
> 최신) 버전으로만 자동 선택된다. 그래서 저장소에 더 높은 버전이 올라오는 순간, 어제까지
> 성공하던 핀 세트 설치가 깨진다 — versioned Depends 는 해석 실패로, Recommends 는
> **에러 없이 조용히 탈락**하는 식으로.

**상태**: 활성
**마지막 검증**: 2026-08-20

## 왜 그렇게 동작하나

`[문서]` apt 의 자동 의존성 해석은 각 패키지의 policy candidate 하나만 후보로 삼는다.
명령행에 `pkg=버전` 으로 적은 패키지만 그 선택이 candidate 를 덮어쓴다. 어떤 패키지의
Depends 가 `(= 구버전)` 을 요구하는데 그 구버전이 candidate 가 아니면, apt 는 자동으로
구버전을 골라주지 않고 "not going to be installed" 로 포기한다.

`[실측]` 결과적으로 세 단계의 함정이 연쇄된다 (이미지 빌드 3연속 실패로 확인):

1. **버전 없이 적은 패키지**가 저장소에 새로 올라온 상위 버전으로 풀리며, 명시 핀들과 충돌
2. 그 패키지를 핀하면, 이번엔 그것의 **엄격 버전 Depends**(`= 같은 버전`)가 비후보 버전을
   요구해 또 실패 — 자식도 명시 핀해야 풀린다
3. **Recommends** 는 만족 불가면 에러 없이 빠진다 — 빌드는 성공하는데 이미지 구성이
   이전 빌드와 조용히 달라진다

같은 소스에서 나온 형제 패키지들(서버·공용 라이브러리·플러그인)이 `(= 동일버전)` 으로 서로를
묶는 배포 구조(OpenStack 계열 deb 등)에서 특히 잘 터진다.

## 확인하는 방법

```
$ apt-get install -y <server-pkg> <lib-pkg>=1.0~rc1-1
The following packages have unmet dependencies:
 <server-pkg> : Depends: <common-pkg> (= 1.0~rc1-1) but it is not going to be installed
E: Unable to correct problems, you have held broken packages.

$ apt-cache policy <common-pkg>     # candidate 가 1.0-1(신버전)로 바뀌어 있음
```

candidate 가 요구 버전보다 높아져 있으면 이 노트의 상황이다. 해결은 요구되는 형제 패키지
전부를 같은 버전으로 명시 핀하거나, 세트 전체를 후보 버전으로 올리는 것.

## 이 노트가 틀렸다면

- 같은 상황(비후보 구버전을 요구하는 Depends)에서 apt 가 **자동으로 구버전을 선택해 설치
  성공**하는 관측 → resolver 동작 이해가 틀림 — 재조사
- `APT::Install-Recommends "false"` 환경이라면 3번(Recommends 조용한 탈락)은 해당 없음 —
  탈락이 아니라 애초에 설치 대상이 아니다

## 적용 범위

Debian/Ubuntu `apt-get install` (bionic·jammy 컨테이너 빌드에서 실측). `aptitude` 나 대화형
해석기는 후보 외 버전을 제안하기도 하므로 별개 `[추정 — aptitude 는 이번에 안 써봤다]`.

## 함정

- ⚠️ **어제 성공한 빌드가 오늘 깨지면 내 변경보다 저장소를 먼저 의심하라.** 핀 세트는 그대로인데
  저장소에 새 버전이 올라온 것만으로 깨진다 — Dockerfile diff 에는 아무것도 없다.
- ⚠️ Recommends 탈락은 성공 로그에 묻힌다. 빌드 성공 ≠ 구성 동일 — 이전 빌드와 설치 목록을
  비교하거나 Recommends 대상도 명시 핀해서 실패가 드러나게 만든다.
- 💡 한 패키지를 핀할 때는 같은 소스의 형제 패키지(엄격 버전 Depends 로 묶인 것들)를 통째로
  핀한다. 하나씩 추가하며 빌드로 검증하면 실패가 한 단계씩만 드러나 시간을 배로 쓴다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-08-20 | 최초 작성 — 저장소에 신버전 업로드 직후 rc 핀 이미지 빌드 3연속 실패를 단계별로 해소하며 확인 | `[실측]` |

## 관련

- `version-skew-in-image-pins.md` — 핀을 "덜 올려서" 생기는 스큐. 이 노트는 반대로
  핀을 "안 건 이웃" 이 저장소 변화로 어긋나는 경우다.
- constraints 핀은 원본 잠금과 함께 늙는다 — 같은 계열의 부패 패턴.
