# 출력 골격

README에 삽입할 스크린샷 섹션. `<...>`는 실제 값으로 치환한다. 이미지 경로는 **상대경로**를 쓰고, 이미지를 같은 경로의 링크로 감싼다.

## 라이트/다크 쌍 (기본형)

프로젝트 한 줄 소개 바로 다음, 설치 안내 앞에 놓는다.

```markdown
---

[![<화면 이름> 라이트 모드](./docs/<screen>-light.png)](./docs/<screen>-light.png)
[![<화면 이름> 다크 모드](./docs/<screen>-dark.png)](./docs/<screen>-dark.png)

---
```

두 이미지를 줄바꿈 없이 붙여 쓰면 나란히 렌더된다. 사이에 빈 줄을 넣으면 세로로 쌓인다.

## 단일 테마

```markdown
[![<화면 이름>](./docs/<screen>.png)](./docs/<screen>.png)
```

## 화면이 3장 이상일 때 — 캡션을 붙인다

화면마다 뭘 보여주는지 한 줄로 적는다. 이미지만 나열하면 뭘 보라는 건지 알 수 없다.

```markdown
## 화면

#### <화면 1 이름>
<이 화면이 무엇을 하는지 한 줄>

[![<화면 1 이름>](./docs/<screen-1>.png)](./docs/<screen-1>.png)

#### <화면 2 이름>
<한 줄>

[![<화면 2 이름>](./docs/<screen-2>.png)](./docs/<screen-2>.png)
```

## 데스크톱 + 모바일

반응형이 셀링 포인트일 때만 쓴다. 폭 차이가 커서 나란히 두면 모바일이 뭉개지므로 표로 정렬한다.

```markdown
| 데스크톱 | 모바일 |
| :---: | :---: |
| [![<화면> 데스크톱](./docs/<screen>-desktop.png)](./docs/<screen>-desktop.png) | [![<화면> 모바일](./docs/<screen>-mobile.png)](./docs/<screen>-mobile.png) |
```

## 가운데 정렬이 필요할 때

README 상단이 `<div align="center">`로 감싸여 있는 레포라면 그 안에 넣고, `<br/>` 로 줄을 나눈다. 마크다운 이미지 문법은 HTML 블록 안에서도 동작한다.

## 금지

- `/owner/repo/raw/main/...` 형태의 절대경로 — 포크·리네임에서 깨진다
- `![스크린샷](...)`, `![이미지 1](...)` 같은 alt text — 화면 이름을 쓴다
- 링크로 감싸지 않은 맨 이미지 — 축소된 상태로만 볼 수 있다
- `<screen>-v2.png`, `<screen>-new.png` — 갱신은 같은 파일명 덮어쓰기
