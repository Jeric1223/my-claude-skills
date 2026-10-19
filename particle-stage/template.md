# particle-stage 템플릿

엔진은 프레임워크 독립 바닐라 JS 클래스다. 장면(`scenes`)의 `build(api)`만 채우면 된다.
TS 프로젝트라면 `ParticleStage.ts`로 옮기고 필드에 타입만 붙인다.

## 1. 마크업 + CSS 골격

```html
<section class="stage" id="stage" tabindex="0" aria-label="소개 장면. 위아래 화살표로 이동">
  <canvas aria-hidden="true"></canvas>
  <div class="caption" aria-live="polite">
    <div class="step"></div>
    <h2 class="title"></h2>
    <p class="body"></p>
  </div>
  <ul class="dots"></ul>
  <div class="hint">스크롤하면 한 장씩 넘어가요</div>
</section>
```

```css
:root { --paper: #f3f5f4; --ink: #1d2422; --ink-soft: #5b6662; --accent: #2b6c8a; }
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) { --paper: #111514; --ink: #e3e9e6; --ink-soft: #95a19c; --accent: #6db3d3; color-scheme: dark; }
}
:root[data-theme="dark"] { --paper: #111514; --ink: #e3e9e6; --ink-soft: #95a19c; --accent: #6db3d3; color-scheme: dark; }
body { background: var(--paper); color: var(--ink); }

.stage { position: relative; height: clamp(440px, 70vh, 640px); overflow: hidden; outline: none; }
.stage:focus-visible { box-shadow: 0 0 0 3px var(--accent); }
.stage canvas { position: absolute; inset: 0; width: 100%; height: 100%; pointer-events: none; }
.caption { position: absolute; left: 32px; top: 50%; transform: translateY(-50%); width: min(34%, 340px); pointer-events: none; }
.caption > * { transition: opacity .45s, transform .45s; }
.caption.out > * { opacity: 0; transform: translateY(8px); }
.dots { position: absolute; right: 18px; top: 50%; transform: translateY(-50%); list-style: none; margin: 0; padding: 0; }
.dots button[aria-current="step"] { color: var(--ink); }
@media (max-width: 720px) {
  .caption { left: 18px; right: 18px; width: auto; top: auto; bottom: 44px; transform: none; }
}
@media (prefers-reduced-motion: reduce) { .caption > * { transition: none; } }
```

## 2. 엔진: `ParticleStage`

```js
export class ParticleStage {
  /**
   * @param {HTMLElement} root  .stage 요소
   * @param {{label:string,title:string,body:string,build:(api)=>void,live?:boolean}[]} scenes
   * @param {{spring?:number,damping?:number,count?:{desktop:number,mobile:number}}} [opts]
   */
  constructor(root, scenes, opts = {}) {
    this.root = root;
    this.scenes = scenes;
    this.canvas = root.querySelector('canvas');
    this.ctx = this.canvas.getContext('2d');
    this.ui = {
      caption: root.querySelector('.caption'),
      step: root.querySelector('.step'),
      title: root.querySelector('.title'),
      body: root.querySelector('.body'),
      dots: root.querySelector('.dots'),
    };
    this.spring = opts.spring ?? 0.06;
    this.damping = opts.damping ?? 0.82;
    this.countCfg = opts.count ?? { desktop: 2600, mobile: 1400 };
    this.reduceMQ = matchMedia('(prefers-reduced-motion: reduce)');
    this.narrowMQ = matchMedia('(max-width: 720px)');

    this.W = 0; this.H = 0; this.dpr = 1; this.N = 0;
    this.current = 0; this.stageStart = 0; this.locked = false; this.wheelAcc = 0;
    this.mouse = { x: -9999, y: -9999 };
    this.running = false; this.visible = true;
    this.colors = { ink: '#000', accent: '#06c' };
    this.cleanups = [];
    this.frame = this.frame.bind(this);
  }

  async start() {
    this.renderDots();
    this.bindInput();
    this.readColors();
    if (this.fontsToWait) { try { await Promise.all(this.fontsToWait.map(f => document.fonts.load(f))); } catch {} }
    this.resize();
    this.go(0);
  }

  destroy() {
    this.visible = false;
    this.cleanups.forEach(fn => fn());
    this.cleanups = [];
  }

  // ---------- 상태 ----------
  alloc(n) {
    this.N = n;
    const f = () => new Float32Array(n);
    Object.assign(this, { x: f(), y: f(), vx: f(), vy: f(), tx: f(), ty: f(), delay: f(), seed: f() });
    this.grp = new Uint8Array(n); // 0 진한 점, 1 옅은 점, 2 강조색
    for (let i = 0; i < n; i++) {
      this.x[i] = Math.random() * this.W;
      this.y[i] = Math.random() * this.H;
      this.seed[i] = Math.random();
      this.grp[i] = i % 37 === 0 ? 2 : i % 3 === 0 ? 1 : 0;
    }
  }

  /** 점들이 모일 영역. 데스크톱은 캡션 오른쪽, 모바일은 캡션 위 */
  region() {
    const { W, H } = this;
    if (this.narrowMQ.matches) return { x: 16, y: 54, w: W - 32, h: H * 0.56 };
    const left = Math.min(W * 0.4, 400);
    return { x: left, y: 40, w: W - left - 110, h: H - 80 };
  }

  // ---------- 장면 build()에 넘기는 도구 ----------
  get api() {
    return {
      N: this.N, tx: this.tx, ty: this.ty, grp: this.grp, seed: this.seed,
      region: this.region(),
      sampleDrawing: (draw) => this.sampleDrawing(draw),
    };
  }

  /** 오프스크린 캔버스에 draw(g, w, h)로 그린 모양의 픽셀을 목적지로 */
  sampleDrawing(draw, step = 2) {
    const r = this.region();
    const ow = Math.max(1, Math.ceil(r.w)), oh = Math.max(1, Math.ceil(r.h));
    const oc = document.createElement('canvas');
    oc.width = ow; oc.height = oh;
    const g = oc.getContext('2d', { willReadFrequently: true });
    g.fillStyle = '#000'; g.strokeStyle = '#000';
    draw(g, ow, oh);
    const d = g.getImageData(0, 0, ow, oh).data;
    const pts = [];
    for (let yy = 0; yy < oh; yy += step)
      for (let xx = 0; xx < ow; xx += step)
        if (d[(yy * ow + xx) * 4 + 3] > 128) pts.push(xx, yy);
    const m = pts.length / 2;
    for (let i = 0; i < this.N; i++) {
      if (!m) { this.tx[i] = r.x + r.w / 2; this.ty[i] = r.y + r.h / 2; continue; }
      const k = (Math.random() * m) | 0;
      this.tx[i] = r.x + pts[k * 2] + Math.random() * step;
      this.ty[i] = r.y + pts[k * 2 + 1] + Math.random() * step;
    }
  }

  // ---------- 전환 ----------
  go(i) {
    if (i < 0 || i >= this.scenes.length) return;
    this.current = i;
    this.stageStart = performance.now();
    for (let k = 0; k < this.N; k++) this.delay[k] = Math.random() * 350;
    this.scenes[i].build(this.api);
    if (this.reduceMQ.matches) this.snap();
    this.updateCaption(i);
    if (!this.running) { this.running = true; requestAnimationFrame(this.frame); }
  }

  snap() { this.x.set(this.tx); this.y.set(this.ty); this.vx.fill(0); this.vy.fill(0); }

  updateCaption(i) {
    const s = this.scenes[i], ui = this.ui;
    ui.caption?.classList.add('out');
    setTimeout(() => {
      if (ui.step) ui.step.textContent = `${i + 1} / ${this.scenes.length}`;
      if (ui.title) ui.title.textContent = s.title;
      if (ui.body) ui.body.textContent = s.body;
      ui.caption?.classList.remove('out');
    }, this.reduceMQ.matches ? 0 : 220);
    ui.dots?.querySelectorAll('button').forEach((b, j) =>
      j === i ? b.setAttribute('aria-current', 'step') : b.removeAttribute('aria-current'));
  }

  renderDots() {
    if (!this.ui.dots) return;
    this.ui.dots.innerHTML = '';
    this.scenes.forEach((s, i) => {
      const li = document.createElement('li');
      const b = document.createElement('button');
      b.type = 'button';
      b.textContent = s.label;
      b.setAttribute('aria-label', `${i + 1}번째 장면: ${s.label}`);
      b.addEventListener('click', () => this.go(i));
      li.appendChild(b);
      this.ui.dots.appendChild(li);
    });
  }

  // ---------- 입력 ----------
  on(target, type, fn, opts) {
    target.addEventListener(type, fn, opts);
    this.cleanups.push(() => target.removeEventListener(type, fn, opts));
  }

  bindInput() {
    const root = this.root, last = () => this.scenes.length - 1;

    this.on(root, 'wheel', (e) => {
      const dir = Math.sign(e.deltaY), next = this.current + dir;
      if (!dir || next < 0 || next > last()) return; // 경계: 페이지 스크롤에 양보
      e.preventDefault();
      if (this.locked) return;
      this.wheelAcc += e.deltaY;
      if (Math.abs(this.wheelAcc) < 24) return;      // 미세 트랙패드 입력 무시
      this.wheelAcc = 0;
      this.go(next);
      this.locked = true;
      setTimeout(() => (this.locked = false), 800);
    }, { passive: false });

    let touchY = null;
    this.on(root, 'touchstart', (e) => { touchY = e.touches[0].clientY; }, { passive: true });
    this.on(root, 'touchmove', (e) => {
      if (touchY === null) return;
      const next = this.current + Math.sign(touchY - e.touches[0].clientY);
      if (next >= 0 && next <= last()) e.preventDefault();
    }, { passive: false });
    this.on(root, 'touchend', (e) => {
      if (touchY === null) return;
      const dy = touchY - e.changedTouches[0].clientY;
      touchY = null;
      if (Math.abs(dy) > 40) this.go(this.current + Math.sign(dy));
    });

    this.on(root, 'keydown', (e) => {
      if ((e.key === 'ArrowDown' || e.key === 'PageDown') && this.current < last()) { e.preventDefault(); this.go(this.current + 1); }
      if ((e.key === 'ArrowUp' || e.key === 'PageUp') && this.current > 0) { e.preventDefault(); this.go(this.current - 1); }
    });

    this.on(root, 'pointermove', (e) => {
      const r = root.getBoundingClientRect();
      this.mouse.x = e.clientX - r.left; this.mouse.y = e.clientY - r.top;
    });
    this.on(root, 'pointerleave', () => { this.mouse.x = this.mouse.y = -9999; });

    const io = new IntersectionObserver(([entry]) => {
      this.visible = entry.isIntersecting;
      if (this.visible && !this.running) { this.running = true; requestAnimationFrame(this.frame); }
    });
    io.observe(root);
    this.cleanups.push(() => io.disconnect());

    const ro = new ResizeObserver(() => this.resize());
    ro.observe(root);
    this.cleanups.push(() => ro.disconnect());

    const mo = new MutationObserver(() => this.readColors());
    mo.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme'] });
    this.cleanups.push(() => mo.disconnect());
    this.on(matchMedia('(prefers-color-scheme: dark)'), 'change', () => this.readColors());
    this.on(this.narrowMQ, 'change', () => this.resize());
  }

  readColors() {
    const cs = getComputedStyle(document.documentElement);
    this.colors.ink = cs.getPropertyValue('--ink').trim() || this.colors.ink;
    this.colors.accent = cs.getPropertyValue('--accent').trim() || this.colors.accent;
  }

  resize() {
    const rect = this.root.getBoundingClientRect();
    this.W = rect.width; this.H = rect.height;
    this.dpr = Math.min(window.devicePixelRatio || 1, 2);
    this.canvas.width = Math.round(this.W * this.dpr);
    this.canvas.height = Math.round(this.H * this.dpr);
    this.ctx.setTransform(this.dpr, 0, 0, this.dpr, 0, 0);
    const want = this.narrowMQ.matches ? this.countCfg.mobile : this.countCfg.desktop;
    if (want !== this.N) this.alloc(want);
    this.scenes[this.current].build(this.api);
    if (this.reduceMQ.matches) this.snap();
  }

  // ---------- 루프 ----------
  frame(now) {
    if (!this.visible) { this.running = false; return; }
    const { N, x, y, vx, vy, tx, ty, delay, spring, damping, mouse, ctx } = this;
    const scene = this.scenes[this.current];
    const reduce = this.reduceMQ.matches;

    if (scene.live && scene.update && !reduce) scene.update(this.api, now);

    const R = this.narrowMQ.matches ? 60 : 90, R2 = R * R;
    const elapsed = now - this.stageStart;
    for (let i = 0; i < N; i++) {
      let ax = 0, ay = 0;
      if (elapsed >= delay[i]) { ax = (tx[i] - x[i]) * spring; ay = (ty[i] - y[i]) * spring; }
      if (!reduce) {
        const dx = x[i] - mouse.x, dy = y[i] - mouse.y, d2 = dx * dx + dy * dy;
        if (d2 < R2) {
          const f = (1 - d2 / R2) * 2.2 / Math.sqrt(d2 || 1);
          ax += dx * f; ay += dy * f;
        }
      }
      vx[i] = (vx[i] + ax) * damping;
      vy[i] = (vy[i] + ay) * damping;
      x[i] += vx[i];
      y[i] += vy[i];
    }

    ctx.clearRect(0, 0, this.W, this.H);
    const passes = [[0, 0.82, 1.6, this.colors.ink], [1, 0.35, 1.3, this.colors.ink], [2, 0.95, 2, this.colors.accent]];
    for (const [g, a, s, c] of passes) {
      ctx.globalAlpha = a; ctx.fillStyle = c;
      for (let i = 0; i < N; i++) if (this.grp[i] === g) ctx.fillRect(x[i], y[i], s, s);
    }
    ctx.globalAlpha = 1;
    requestAnimationFrame(this.frame);
  }
}
```

## 3. 장면 정의 예시

```js
const HAND_FONT = "'Nanum Pen Script', cursive";

export const scenes = [
  {
    label: '소개', title: '팀의 결정을 모두의 기억으로', body: '…',
    build: ({ sampleDrawing }) => sampleDrawing((g, w, h) => {
      g.font = `${Math.min(h * 0.62, w * 0.42)}px ${HAND_FONT}`;
      g.textAlign = 'center'; g.textBaseline = 'middle';
      g.fillText('knot', w / 2, h / 2);
    }),
  },
  {
    label: '문제', title: '정한 건 많은데, 남은 건 없던 팀에게.', body: '…',
    build: ({ sampleDrawing }) => sampleDrawing((g, w, h) => {
      g.font = `36px ${HAND_FONT}`; g.textAlign = 'center'; g.textBaseline = 'middle'; g.lineWidth = 3;
      [['방금 뭐라고 하셨죠?', .3, .2], ['그거 지난번에 정했나?', .65, .5], ['기록은 누가 해요?', .4, .78]]
        .forEach(([t, fx, fy]) => {
          const bw = g.measureText(t).width + 40, bh = 54, cx = fx * w, cy = fy * h;
          g.beginPath(); g.roundRect(cx - bw / 2, cy - bh / 2, bw, bh, bh / 2); g.stroke();
          g.fillText(t, cx, cy + 2);
        });
    }),
  },
  {
    label: '녹음', title: '마이크를 누르면 기록이 시작돼요.', body: '…',
    live: true,                                   // 매 프레임 update()로 목적지 갱신
    build: (api) => waveTargets(api, performance.now()),
    update: waveTargets,
  },
];

// 점마다 가로 위치(seed)와 세로 비율(u)은 고정, 진폭만 시간에 따라 바뀐다
function waveTargets({ N, tx, ty, seed, region: r }, now) {
  const s = now / 1000, cy = r.y + r.h / 2;
  for (let i = 0; i < N; i++) {
    const p = seed[i];                                   // 0~1
    const u = ((i * 7919) % 1000) / 500 - 1;             // -1~1
    const env = Math.exp(-(((p - 0.5) / 0.26) ** 2));    // 가운데가 높은 포락선
    const amp = r.h * 0.42 * env * (0.35 + 0.65 * Math.abs(Math.sin(p * 23 + s * 2.1) * Math.cos(p * 7 - s * 1.3)));
    tx[i] = r.x + p * r.w;
    ty[i] = cy + Math.sign(u) * Math.abs(u) ** 1.6 * amp;
  }
}
```

## 4. 바닐라 사용

```js
const stage = new ParticleStage(document.getElementById('stage'), scenes);
stage.fontsToWait = ["120px 'Nanum Pen Script'"];
stage.start();
```

## 5. React 래핑

```tsx
import { useEffect, useRef } from 'react';
import { ParticleStage } from './ParticleStage';
import { scenes } from './scenes';

export function ParticleHero() {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    const stage = new ParticleStage(ref.current, scenes);
    stage.fontsToWait = ["120px 'Nanum Pen Script'"];
    stage.start();
    return () => stage.destroy();   // StrictMode 이중 마운트에서도 리스너가 남지 않게
  }, []);

  return (
    <section ref={ref} className="stage" tabIndex={0} aria-label="소개 장면. 위아래 화살표로 이동">
      <canvas aria-hidden="true" />
      <div className="caption" aria-live="polite">
        <div className="step" /><h2 className="title" /><p className="body" />
      </div>
      <ul className="dots" />
      <div className="hint">스크롤하면 한 장씩 넘어가요</div>
    </section>
  );
}
```

> 좌표·속도는 절대 `useState`에 넣지 않는다. React는 마운트/언마운트만 담당한다.
