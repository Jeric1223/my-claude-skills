---
name: particle-stage
description: Use when designing or building a knot.kr-style interactive landing — step-by-step scroll where thousands of Canvas 2D particles morph into text, shapes, waveforms or graphs per scene — "파티클 랜딩", "스크롤하면 점이 모이는", "knot 같은 인터랙티브 사이트", "인터랙티브 랜딩 만들어줘"
---

# particle-stage

## Overview

knot.kr 랜딩을 실제로 뜯어보면 GSAP·three.js·Framer Motion 같은 애니메이션 라이브러리가 **하나도 없다**. React SPA 위에 **페이지를 덮는 canvas 하나**가 깔려 있고, 수천 개의 점이 장면마다 손글씨 로고 → 말풍선 → 음성 파형 → 카드·연결선 → 질문 그래프로 재배치된다. 글자·버튼·카드는 일반 DOM이다.

핵심 원칙: **점의 "목적지 좌표"만 바꾸고, 이동은 스프링 물리에 맡긴다.** 장면 전환 = 목적지 배열 교체. 트윈 라이브러리가 필요 없는 이유가 이거다.

## When to Use

- 제품 랜딩/소개 페이지를 "한 장씩 넘기는" 스토리텔링 구조로 만들 때
- 로고·문장·도형·데이터 모양(파형, 그래프)이 점으로 모였다 흩어지는 연출이 필요할 때
- 라이브러리 없이 가볍게(Canvas 2D) 끝내고 싶을 때

**Not for:** 3D 연출이나 1만 개 이상 파티클 → three.js `Points` + 셰이더로 간다. DOM 요소를 스크롤에 맞춰 고정·이동만 하면 되는 경우 → GSAP ScrollTrigger가 더 빠르다.

## 구조 (먼저 결정할 것)

```
[section.stage]  ← 휠/터치/키 입력을 받는 무대 (position: relative)
 ├─ canvas       ← position:absolute; inset:0; pointer-events:none  (점)
 ├─ .caption     ← 장면별 제목·설명 (DOM, aria-live="polite")
 ├─ .dots        ← 장면 인디케이터 (button, aria-current="step")
 └─ .hint        ← "스크롤하면 한 장씩 넘어가요"
```

- **텍스트는 DOM, 연출은 canvas.** SEO·접근성·텍스트 선택이 살아남는다. 캔버스 위에 글자를 그리지 않는다.
- 엔진은 **React 바깥의 순수 TS 클래스**(`ParticleStage`)로 만들고 컴포넌트는 `useEffect`에서 mount/destroy만 한다. 좌표를 React state로 들고 있으면 매 프레임 리렌더로 즉시 죽는다.
- 장면은 설정 배열: `{ label, title, body, build(), live? }`. 새 장면 추가 = 배열에 한 줄.

## Process

1. **장면 시나리오부터 쓴다.** 장면당 "점이 무엇이 되는가 + 캡션 한 문장". 점 모양이 캡션의 의미를 시각화해야 한다(녹음 → 파형, 정리 → 카드, 탐색 → 그래프). 4~7장이 적당.
2. **각 장면의 목적지 생성 방식을 고른다** (아래 표).
3. `template.md`의 엔진 골격을 복사해 `build()`만 장면별로 채운다.
4. 입력(휠·터치·키·포인터)과 품질 처리(dpr, 화면 밖 정지, reduced-motion)를 **빠짐없이** 넣는다 — 아래 체크리스트.
5. 디자인: 배경은 종이 질감의 옅은 뉴트럴, 점은 잉크색 2단계 알파 + 강조색 소수(약 3%). 장식은 점이 전부이고 나머지는 조용하게.
6. 데스크톱·모바일·다크모드·reduced-motion 각각 한 번씩 열어 확인한다.

### 목적지 생성 방식

| 모양 | 방법 |
|---|---|
| 글자·로고·말풍선·아이콘 | 오프스크린 캔버스에 `fillText`/`stroke`로 그리고 `getImageData` 알파 > 128 픽셀을 2px 간격으로 수집 → N개 랜덤 샘플 |
| 파형·곡선 | 수식으로 직접 계산. `live: true`로 매 프레임 목적지를 갱신하면 살아 움직인다 |
| 그래프·네트워크 | 시드 고정 난수로 노드 배치 → 근접 이웃 2개로 간선 → 점을 노드 주변(가우시안)·간선 위에 분배. 강조색 점은 한 경로만 |
| 카드 테두리 | DOM 카드의 `getBoundingClientRect()`를 무대 좌표로 바꿔 둘레 위에 분배 (DOM과 점이 겹쳐 맞물리는 연출) |

## 반드시 넣을 것 (체크리스트)

- [ ] 스프링: `v = (v + (target - pos) * spring) * damping`. 기본값 spring 0.06, damping 0.82. 점마다 출발 지연 0~350ms
- [ ] 파티클 상태는 **`Float32Array` 필드별 배열** (x, y, vx, vy, tx, ty). 객체 배열 금지 — GC 스파이크
- [ ] 그리기: 색/알파 그룹별로 `fillStyle` 한 번만 바꾸고 `fillRect`로 점 찍기 (`arc`보다 훨씬 빠름)
- [ ] `devicePixelRatio`로 캔버스 해상도 확대, **상한 2**. `ctx.setTransform(dpr,0,0,dpr,0,0)`
- [ ] 점 개수: 데스크톱 ~2,600 / 모바일 ~1,400. Canvas 2D 현실적 한계는 5천~1만
- [ ] 휠: `{ passive: false }` + `preventDefault()`, **800ms 잠금**, 미세 트랙패드 델타 누적 임계값(≈24)
- [ ] **첫/마지막 장면 경계에서는 막지 않는다** → 페이지 스크롤로 양보 (사용자 가두기 금지)
- [ ] 터치: 40px 이상 스와이프로 전환, 경계 밖이면 `preventDefault` 안 함
- [ ] 키보드: 무대에 `tabindex="0"`, ↑↓/PageUp/PageDown, `:focus-visible` 링
- [ ] 포인터 반발: 반경 R(데스크톱 90 / 모바일 60) 안에서 `(1 - d²/R²) * 2.2 / d` 만큼 바깥으로
- [ ] `IntersectionObserver`로 화면 밖이면 rAF 루프 정지, 들어오면 재개 (`running` 플래그로 중복 루프 방지)
- [ ] `prefers-reduced-motion`: 전환 시 즉시 목적지로 스냅, 포인터 반발·live 갱신 끄기
- [ ] `ResizeObserver`로 리사이즈 시 현재 장면 `build()` 재실행
- [ ] 웹폰트로 글자를 샘플링한다면 `await document.fonts.load(...)` 후에 첫 `build()` — 안 그러면 fallback 폰트 모양으로 굳는다
- [ ] 테마 토큰(`--ink`, `--accent`)을 `getComputedStyle`로 읽어 캔버스 색에 반영, 테마 변경 시 다시 읽기
- [ ] 캡션은 처음부터 보이는 상태. `opacity:0`에서 시작해 observer를 기다리게 만들지 않는다

## Common Mistakes

- 장면 전환을 트윈(시작→끝 보간)으로 구현 → 전환 중 다시 넘기면 튐. 스프링은 어느 시점에 목적지가 바뀌어도 자연스럽다
- 휠 이벤트를 `passive` 기본값으로 등록 → `preventDefault`가 무시되고 페이지가 같이 스크롤됨
- 잠금 없이 휠마다 전환 → 트랙패드 한 번에 장면이 끝까지 넘어감
- 매 프레임 `getImageData` 호출 → 장면 전환 때 한 번만. 매 프레임 바뀌는 모양은 수식으로
- dpr 무시 → 레티나에서 점이 뭉개짐. dpr 상한 없음 → 3x 폰에서 픽셀 9배로 프레임 드랍
- 캔버스에 `pointer-events` 살려둠 → 그 위 DOM 버튼이 클릭 안 됨
- 모바일 레이아웃에서 캡션과 점 영역을 안 나눔 → 글자 위에 점이 겹쳐 안 읽힘. 좁은 화면은 점 영역 위, 캡션 아래로 분리

## 참고

- `template.md` — 엔진 골격(바닐라 JS, 그대로 동작) + React 래핑 예시
