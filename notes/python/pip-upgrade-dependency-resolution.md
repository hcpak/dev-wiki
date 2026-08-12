# pip `--upgrade` 는 의존성까지 건드린다

> `pip install --upgrade <pkg>==<ver>` 는 지정한 패키지만 올리는 게 아니라 **의존성 트리를 다시 푼다.**
> 인덱스에 내용이 빈 버전이 올라와 있으면 그걸 집어와서, 설치는 성공했는데 런타임이 깨지는 상태가 된다.

## 왜 그렇게 동작하나

`pip install --upgrade A==1.2` 는 "A를 1.2로"만 뜻하지 않는다. A의 의존성(B, C…)도
**업그레이드 후보로 다시 계산**한다. 이미 설치된 B가 요구사항을 만족하더라도, 인덱스에 더 높은
버전이 있으면 그쪽으로 바꾸려 든다.

문제는 **사설 미러에 올라온 상위 버전이 정상 패키지라는 보장이 없다**는 점이다. 실제로 이렇게 됐다.

```
pip install --upgrade <pkg>==<ver>
→ Successfully installed Jinja2-2.11.3 MarkupSafe-0.0.0 <pkg>-<ver>
```

`MarkupSafe` 가 **1.1.1 → 0.0.0** 으로 바뀌었다. 버전 번호는 낮지만 pip 입장에서는 인덱스에 존재하는
후보였고, 설치는 성공으로 끝났다. 그런데 그 배포판에는 모듈 실체가 없다.

```
python -c "import flask"
ImportError: No module named markupsafe
  File ".../flask/__init__.py", line 19, in <module>
    from jinja2 import Markup, escape
  File ".../jinja2/__init__.py", line 6, in <module>
    from markupsafe import escape
```

💡 `0.0.0` 은 이름만 선점해 둔 플레이스홀더로 보인다(설치는 되지만 임포트할 모듈이 없음).
미러 구성 문제로 추정하며 확증하지는 못했다.

## 확인하는 방법

설치 직후 **재시작하기 전에** 임포트가 되는지 본다. 서비스가 쓰는 인터프리터로 확인해야 한다.

```bash
# 1) 설치
pip install --upgrade <pkg>==<ver>

# 2) 재시작 전 임포트 검증 — 여기서 걸러야 한다
python -c "import flask, markupsafe, <module>; print('ok')"

# 3) 깨졌으면 복구
pip install --force-reinstall MarkupSafe==1.1.1
python -c "import flask, markupsafe; print(flask.__version__, markupsafe.__version__)"
# ('0.10.1', '1.1.1')
```

python 2.7 + 시스템 pip 환경에서 확인했고, 같은 절차를 밟은 노드 3대 모두 동일하게 재현됐다.

## 함정

- ⚠️ **`Successfully installed` 만 보고 재시작하면 서비스가 안 뜬다.** 설치는 성공이고 깨지는 건 임포트 시점이라,
  로그만으로는 알 수 없다. 재시작 전에 임포트 한 번 돌려보는 습관이 안전장치다.
- ⚠️ 하필 지표 수집기(exporter)에서 이게 터지면 **고치려던 것과 같은 결과**(그 노드 지표 전부 유실)가 난다.
  재시작 전에 발견해서 사고로 이어지지는 않았다.
- **`--upgrade` 없이 `pkg==버전` 으로 설치하는 절차에서는 재현되지 않았다.** 이미 충족된 의존성을
  건드리지 않기 때문으로 이해하고 있다. 즉 **손으로 올릴 때만 밟는 지뢰**다.

## 관련

- [py2 유니코드 대칭](py2-unicode-symmetry.md)
