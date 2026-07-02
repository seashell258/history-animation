# 《商王的一天》Phase 1 垂直切片 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第一幕核心 90 秒體驗——開場字卡 → 宮殿夜景（牙痛的商王）→ 按住燒龜甲 → 裂紋竄開 → 「吉」判定 → 「齒」甲骨文演示——作為全案的品質樣品。

**Architecture:** 單一 `index.html`（HTML＋CSS＋JS 全部內嵌），內含：固定 1280×720 座標系的縮放舞台、場景狀態機、Web Audio 程式合成音效引擎、SVG 剪紙風場景、Canvas 火星粒子、以 `#selftest` hash 觸發的瀏覽器內自測框架。

**Tech Stack:** Vanilla JS、SVG、CSS animation、Canvas 2D、Web Audio API。零依賴、零建置。

## Global Constraints

- 交付物為**單一 HTML 檔** `index.html`，零外部依賴、零建置步驟、離線可用（spec §6）。
- **學生零開放式輸入**：僅按住、拖拉、點選、滑動（spec §3）。本切片只用到「按住」與「點選」。
- 同時支援滑鼠與觸控 → 一律使用 pointer events（spec §6）。
- Web Audio 需使用者手勢解鎖 → 「開始上課」按鈕為第一次點擊；**無音訊時所有互動仍可進行**（spec §6）。
- 舞台以 1280×720 橫向為基準，等比縮放至視窗（spec §6）。
- 色盤（spec §7）：墨黑 `#171310`、宣紙米白 `#efe3cc`、硃砂紅 `#b23a2a`、青銅綠 `#56705f`、火焰橘 `#e8823a`、高光金 `#d9a441`。
- 美術風格：剪紙／皮影＋甲骨拓片墨色；字體 `'Noto Serif TC','DFKai-SB','BiauKai','Microsoft JhengHei',serif`（spec §7）。
- 文案一律繁體中文，小學生可讀（短句、無艱澀詞）。
- 本切片**不做**：章節跳轉圓點、背景音樂、第二～四幕（spec §8 Phase 2）。
- 測試方式：瀏覽器開 `index.html#selftest` 跑內嵌自測（邏輯）；開 `index.html` 走視覺檢查清單（表現層）。無 Node、無測試框架。
- 驗證指令（PowerShell，於 `C:\Users\User\Desktop\TRY`）：
  - 自測：`start chrome "$((Resolve-Path .\index.html).Path)#selftest"`（無 Chrome 則用 `msedge`）
  - 視覺：`start .\index.html`

---

### Task 1: 骨架——舞台縮放、設計 tokens、自測框架、開始畫面

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces（後續 Task 依賴）：
  - `<div id="stage">` 內含 `<section data-scene="start|intro|palace|burn|verdict|glyph">` 六個空場景；`.active` class 控制顯示
  - 自測 API：`test(name, fn)`、`assertEq(actual, expected)`、`assertOk(cond, msg)`；`#selftest` 時執行並在畫面印出報告，全過時頁面含字串 `ALL PASS`
  - CSS custom properties：`--ink --paper --cinnabar --bronze --flame --gold --serif`
  - `#start-btn`（開始上課按鈕；Task 2 綁定行為、Task 3 綁定音訊解鎖）

- [ ] **Step 1: 建立 index.html 全部骨架（含一個注定失敗的自測）**

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>商王的一天・第一幕</title>
<style>
:root{
  --ink:#171310; --paper:#efe3cc; --cinnabar:#b23a2a;
  --bronze:#56705f; --flame:#e8823a; --gold:#d9a441;
  --serif:'Noto Serif TC','DFKai-SB','BiauKai','Microsoft JhengHei',serif;
}
html,body{margin:0;height:100%;background:#000;overflow:hidden;font-family:var(--serif);color:var(--paper);}
#stage{position:absolute;left:50%;top:50%;width:1280px;height:720px;
  transform-origin:center;background:var(--ink);overflow:hidden;}
section[data-scene]{position:absolute;inset:0;display:none;}
section[data-scene].active{display:block;}
/* --- start scene --- */
/* 注意：ID 選擇器一律不寫 display，否則特異性(1,0,0)會永遠壓過 .active 切換；
   顯示一律由「ID.active」層級控制 */
#scene-start{flex-direction:column;align-items:center;justify-content:center;gap:28px;}
#scene-start.active{display:flex;}
#scene-start h1{font-size:72px;letter-spacing:.18em;margin:0;color:var(--paper);}
#scene-start p{font-size:24px;letter-spacing:.3em;color:var(--gold);margin:0;}
#start-btn{font-family:var(--serif);font-size:28px;letter-spacing:.2em;padding:14px 56px;
  background:var(--cinnabar);color:var(--paper);border:2px solid var(--gold);
  border-radius:6px;cursor:pointer;}
#start-btn:hover{filter:brightness(1.15);}
</style>
</head>
<body>
<div id="stage">
  <section data-scene="start" id="scene-start" class="active">
    <h1>商王的一天</h1>
    <p>商朝互動歷史課・第一幕</p>
    <button id="start-btn">開始上課</button>
  </section>
  <section data-scene="intro"></section>
  <section data-scene="palace"></section>
  <section data-scene="burn"></section>
  <section data-scene="verdict"></section>
  <section data-scene="glyph"></section>
</div>
<script>
'use strict';
/* ===== 舞台縮放 ===== */
function fitStage(){
  const s = Math.min(innerWidth / 1280, innerHeight / 720);
  document.getElementById('stage').style.transform =
    `translate(-50%,-50%) scale(${s})`;
}
addEventListener('resize', fitStage);
fitStage();

/* ===== 自測框架 ===== */
const selfTests = [];
function test(name, fn){ selfTests.push({name, fn}); }
function assertEq(actual, expected){
  if (actual !== expected) throw new Error(`expected ${expected}, got ${actual}`);
}
function assertOk(cond, msg){ if (!cond) throw new Error(msg || 'assertion failed'); }
function runSelfTests(){
  const lines = [];
  let passed = 0;
  for (const t of selfTests){
    try { t.fn(); lines.push(`PASS  ${t.name}`); passed++; }
    catch(e){ lines.push(`FAIL  ${t.name} — ${e.message}`); }
  }
  lines.push('', passed === selfTests.length
    ? `ALL PASS (${passed}/${selfTests.length})`
    : `${passed}/${selfTests.length} passed`);
  const pre = document.createElement('pre');
  pre.style.cssText = 'position:fixed;inset:0;margin:0;background:#000;color:#4f4;' +
    'z-index:99;padding:20px;font:15px/1.6 Consolas,monospace;overflow:auto;';
  pre.textContent = lines.join('\n');
  document.body.appendChild(pre);
}
addEventListener('load', () => { if (location.hash === '#selftest') runSelfTests(); });

/* 骨架驗證用測試：六個場景 section 存在（先寫成錯的期望值，驗證框架會抓到 FAIL） */
test('stage 含 6 個場景 section', () => {
  assertEq(document.querySelectorAll('section[data-scene]').length, 5);
});
</script>
</body>
</html>
```

- [ ] **Step 2: 驗證自測框架會抓到失敗**

Run: `start chrome "$((Resolve-Path .\index.html).Path)#selftest"`
Expected: 報告顯示 `FAIL  stage 含 6 個場景 section — expected 5, got 6`，結尾為 `0/1 passed`。

- [ ] **Step 3: 把期望值改對**

```js
test('stage 含 6 個場景 section', () => {
  assertEq(document.querySelectorAll('section[data-scene]').length, 6);
});
```

- [ ] **Step 4: 驗證通過＋視覺檢查**

Run: 重新整理 `#selftest` 分頁 → Expected: `ALL PASS (1/1)`。
Run: `start .\index.html` → Expected: 黑底置中舞台、大字《商王的一天》、硃砂紅「開始上課」按鈕；縮放視窗時舞台等比縮放不變形。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: stage scaffold, design tokens, self-test harness"
```

---

### Task 2: 場景狀態機

**Files:**
- Modify: `index.html`（`<script>` 內、自測區塊之前）

**Interfaces:**
- Consumes: Task 1 的 `section[data-scene]`、`#start-btn`、自測 API
- Produces:
  - `const SCENES = ['start','intro','palace','burn','verdict','glyph']`
  - `game.goto(name) -> boolean`（切換 `.active`、dispatch `scene:enter` CustomEvent，`detail.scene` 為場景名；未知名稱回傳 `false` 且不動作）
  - `game.next()`、`game.current`（字串 getter）
  - 頁面載入後自動 `game.goto('start')`；`#start-btn` 點擊 → `game.goto('intro')`

- [ ] **Step 1: 先寫失敗的自測**

```js
test('game.goto 切換 active 與 current', () => {
  game.goto('palace');
  assertEq(game.current, 'palace');
  assertEq(document.querySelectorAll('section[data-scene].active').length, 1);
  assertEq(document.querySelector('section[data-scene].active').dataset.scene, 'palace');
});
test('game.goto 未知場景回傳 false 且不變', () => {
  game.goto('burn');
  assertEq(game.goto('nonsense'), false);
  assertEq(game.current, 'burn');
});
test('game.next 依序前進', () => {
  game.goto('start'); game.next();
  assertEq(game.current, 'intro');
});
test('scene:enter 事件會發出', () => {
  let got = null;
  const h = e => { got = e.detail.scene; };
  document.addEventListener('scene:enter', h);
  game.goto('glyph');
  document.removeEventListener('scene:enter', h);
  assertEq(got, 'glyph');
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 新增的測試全部 `FAIL`（`game is not defined`）。

- [ ] **Step 3: 實作狀態機（放在自測區塊之前）**

```js
/* ===== 場景狀態機 ===== */
const SCENES = ['start','intro','palace','burn','verdict','glyph'];
const game = {
  index: 0,
  get current(){ return SCENES[this.index]; },
  goto(name){
    const i = SCENES.indexOf(name);
    if (i === -1) return false;
    this.index = i;
    document.querySelectorAll('section[data-scene]').forEach(s =>
      s.classList.toggle('active', s.dataset.scene === name));
    document.dispatchEvent(new CustomEvent('scene:enter', {detail:{scene:name}}));
    return true;
  },
  next(){ if (this.index < SCENES.length - 1) this.goto(SCENES[this.index + 1]); }
};
addEventListener('load', () => { if (location.hash !== '#selftest') game.goto('start'); });
document.getElementById('start-btn').addEventListener('click', () => game.goto('intro'));
```

（注意：`#selftest` 模式不自動 goto，避免測試順序被干擾；測試自己控制場景。）

- [ ] **Step 4: 驗證通過**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (5/5)`。
Run: `start .\index.html` → 點「開始上課」→ Expected: start 畫面消失（intro 目前是空白黑畫面，正常）。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: scene state machine with scene:enter events"
```

---

### Task 3: Web Audio 音效引擎

**Files:**
- Modify: `index.html`（`<script>` 內、狀態機之後）

**Interfaces:**
- Consumes: Task 1 自測 API、`#start-btn`
- Produces:
  - `audio.unlock()`——惰性建立 `AudioContext` 並 `resume()`；綁在 `#start-btn` click 上
  - `audio.startFire() -> {stop()}`——持續的火焰劈啪聲；未解鎖時回傳 no-op handle **不拋錯**
  - `audio.pop()`——裂開「啪」（噪音爆點＋低頻墜落）；`audio.ding()`——答對／吉（雙正弦衰減）；`audio.thump()`——蓋印悶響。三者未解鎖時靜默返回
  - `audio.enabled`（布林，之後做靜音鈕用；本切片預設 `true` 即可）

- [ ] **Step 1: 先寫失敗的自測**

```js
test('audio 未解鎖時所有函式安全 no-op', () => {
  assertEq(audio.ctx, null);
  audio.pop(); audio.ding(); audio.thump();
  const h = audio.startFire();
  assertOk(h && typeof h.stop === 'function', 'startFire 應回傳含 stop 的 handle');
  h.stop();
});
test('audio.unlock 建立 AudioContext', () => {
  audio.unlock();
  assertOk(audio.ctx !== null, 'unlock 後 ctx 不應為 null');
  assertEq(typeof audio.ctx.createOscillator, 'function');
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 兩則新測試 `FAIL`（`audio is not defined`）。

- [ ] **Step 3: 實作音效引擎**

```js
/* ===== 音效引擎（全程式合成，零音檔） ===== */
const audio = {
  ctx: null,
  enabled: true,
  unlock(){
    if (!this.ctx) this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    if (this.ctx.state === 'suspended') this.ctx.resume();
  },
  _noise(dur){
    const n = Math.floor(this.ctx.sampleRate * dur);
    const buf = this.ctx.createBuffer(1, n, this.ctx.sampleRate);
    const d = buf.getChannelData(0);
    for (let i = 0; i < n; i++) d[i] = Math.random() * 2 - 1;
    return buf;
  },
  startFire(){
    if (!this.ctx || !this.enabled) return { stop(){} };
    const src = this.ctx.createBufferSource();
    src.buffer = this._noise(2); src.loop = true;
    const bp = this.ctx.createBiquadFilter();
    bp.type = 'bandpass'; bp.frequency.value = 900; bp.Q.value = 0.8;
    const g = this.ctx.createGain(); g.gain.value = 0;
    src.connect(bp); bp.connect(g); g.connect(this.ctx.destination);
    src.start();
    const jitter = setInterval(() => {
      g.gain.value = 0.04 + Math.random() * 0.14;   // 隨機抖動 = 劈啪感
    }, 60);
    return {
      stop: () => {
        clearInterval(jitter);
        g.gain.setTargetAtTime(0, this.ctx.currentTime, 0.1);
        src.stop(this.ctx.currentTime + 0.4);
      }
    };
  },
  pop(){
    if (!this.ctx || !this.enabled) return;
    const t = this.ctx.currentTime;
    const src = this.ctx.createBufferSource(); src.buffer = this._noise(0.15);
    const hp = this.ctx.createBiquadFilter(); hp.type = 'highpass'; hp.frequency.value = 1200;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(0.9, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.15);
    src.connect(hp); hp.connect(g); g.connect(this.ctx.destination); src.start(t);
    const o = this.ctx.createOscillator();
    o.frequency.setValueAtTime(90, t);
    o.frequency.exponentialRampToValueAtTime(45, t + 0.25);
    const og = this.ctx.createGain();
    og.gain.setValueAtTime(0.8, t);
    og.gain.exponentialRampToValueAtTime(0.001, t + 0.3);
    o.connect(og); og.connect(this.ctx.destination); o.start(t); o.stop(t + 0.3);
  },
  ding(){
    if (!this.ctx || !this.enabled) return;
    const t = this.ctx.currentTime;
    [660, 990].forEach((f, i) => {
      const o = this.ctx.createOscillator(); o.type = 'sine'; o.frequency.value = f;
      const g = this.ctx.createGain();
      g.gain.setValueAtTime(i ? 0.12 : 0.28, t);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.9);
      o.connect(g); g.connect(this.ctx.destination); o.start(t); o.stop(t + 0.9);
    });
  },
  thump(){
    if (!this.ctx || !this.enabled) return;
    const t = this.ctx.currentTime;
    const o = this.ctx.createOscillator();
    o.frequency.setValueAtTime(120, t);
    o.frequency.exponentialRampToValueAtTime(50, t + 0.2);
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(0.9, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.25);
    o.connect(g); g.connect(this.ctx.destination); o.start(t); o.stop(t + 0.25);
  }
};
document.getElementById('start-btn').addEventListener('click', () => audio.unlock());
```

- [ ] **Step 4: 驗證通過＋人耳檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (7/7)`。
Run: `start .\index.html` → 開 DevTools Console 依序執行：
`audio.unlock(); audio.pop();`（應聽到「啪」）、`audio.ding()`（清亮雙音）、`audio.thump()`（悶響）、`const f=audio.startFire()` 3 秒後 `f.stop()`（劈啪聲起、漸弱收掉）。
Expected: 四種聲音可辨識、無爆音、stop 後火聲確實停止。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: procedural Web Audio SFX engine (fire/pop/ding/thump)"
```

---

### Task 4: 開場字卡與宮殿夜景

**Files:**
- Modify: `index.html`（intro 與 palace 兩個 section 的 HTML＋CSS＋JS）

**Interfaces:**
- Consumes: `game.goto/next`、`scene:enter` 事件、CSS tokens
- Produces:
  - intro：三張字卡點擊逐張推進，最後一張點擊後 `game.goto('palace')`
  - palace：夜景＋商王＋對白條＋`#ask-btn`「問天」按鈕 → `game.goto('burn')`

- [ ] **Step 1: intro 字卡（HTML／CSS／JS）**

HTML——取代空的 intro section：

```html
<section data-scene="intro" id="scene-intro">
  <p id="intro-line"></p>
  <p id="intro-hint">點一下，繼續</p>
</section>
```

CSS——加入 `<style>`：

```css
/* --- intro --- */
#scene-intro{align-items:center;justify-content:center;cursor:pointer;}
#scene-intro.active{display:flex;}
#intro-line{font-size:44px;letter-spacing:.12em;line-height:1.8;text-align:center;
  max-width:900px;opacity:0;transition:opacity .8s;}
#intro-line.show{opacity:1;}
#intro-hint{position:absolute;bottom:36px;right:56px;font-size:18px;color:var(--gold);
  opacity:.7;animation:breathe 2s infinite;}
@keyframes breathe{0%,100%{opacity:.25}50%{opacity:.8}}
```

JS——加入 `<script>`（自測區塊之前）：

```js
/* ===== 開場字卡 ===== */
const INTRO_LINES = [
  '3000 年前的一個深夜。',
  '全天下最有權力的人——商王武丁，',
  '正在為一顆牙，痛到睡不著。'
];
let introIdx = -1;
function introAdvance(){
  const el = document.getElementById('intro-line');
  introIdx++;
  if (introIdx >= INTRO_LINES.length){ game.goto('palace'); return; }
  el.classList.remove('show');
  setTimeout(() => { el.textContent = INTRO_LINES[introIdx]; el.classList.add('show'); }, 250);
}
document.getElementById('scene-intro').addEventListener('click', introAdvance);
document.addEventListener('scene:enter', e => {
  if (e.detail.scene === 'intro'){ introIdx = -1; introAdvance(); }
});
```

- [ ] **Step 2: palace 夜景（HTML／CSS／JS）**

HTML——取代空的 palace section：

```html
<section data-scene="palace" id="scene-palace">
  <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
    <!-- 月亮 -->
    <circle cx="1040" cy="130" r="64" fill="var(--paper)" opacity=".92"/>
    <!-- 遠山 -->
    <path d="M0 520 L180 420 L340 500 L520 400 L700 490 L900 410 L1100 500 L1280 440 L1280 720 L0 720 Z"
          fill="#241e18"/>
    <!-- 宮殿剪影：雙層屋簷 -->
    <g fill="#0d0b09">
      <path d="M340 520 L400 440 L880 440 L940 520 Z"/>
      <path d="M300 560 L380 500 L900 500 L980 560 Z"/>
      <rect x="380" y="560" width="520" height="160"/>
      <path d="M330 442 Q340 420 360 424 L400 440 Z"/>
      <path d="M950 442 Q940 420 920 424 L880 440 Z"/>
    </g>
    <!-- 窗內暖光 -->
    <rect x="590" y="600" width="100" height="120" fill="var(--flame)" opacity=".28"/>
    <!-- 商王剪影：抱著臉頰 -->
    <g id="king" transform="translate(590,470)">
      <ellipse cx="50" cy="118" rx="46" ry="30" fill="#0d0b09"/>
      <rect x="26" y="46" width="48" height="66" rx="18" fill="#0d0b09"/>
      <circle cx="50" cy="34" r="22" fill="#0d0b09"/>
      <rect x="18" y="6" width="64" height="10" rx="2" fill="#0d0b09"/>
      <line x1="22" y1="16" x2="20" y2="30" stroke="var(--gold)" stroke-width="2"/>
      <line x1="78" y1="16" x2="80" y2="30" stroke="var(--gold)" stroke-width="2"/>
      <ellipse cx="74" cy="42" rx="10" ry="14" fill="#0d0b09" transform="rotate(24 74 42)"/>
      <!-- 痛的閃線 -->
      <g id="pain" stroke="var(--cinnabar)" stroke-width="4" stroke-linecap="round" fill="none">
        <path d="M92 20 l14 -12"/><path d="M98 34 l18 -4"/><path d="M94 8 l6 -16"/>
      </g>
    </g>
  </svg>
  <div id="palace-dialog">
    <p>「唉唷……牙好痛。這時代<strong>沒有牙醫</strong>，怎麼辦？」</p>
    <p class="speaker">只剩一個辦法——</p>
    <button id="ask-btn">問天 →</button>
  </div>
</section>
```

CSS：

```css
/* --- palace --- */
#palace-dialog{position:absolute;left:50%;bottom:28px;transform:translateX(-50%);
  width:860px;background:var(--paper);color:var(--ink);border-radius:8px;
  padding:20px 32px;box-shadow:0 6px 24px rgba(0,0,0,.6);}
#palace-dialog p{margin:6px 0;font-size:26px;line-height:1.6;}
#palace-dialog .speaker{color:var(--cinnabar);font-size:22px;}
#palace-dialog strong{color:var(--cinnabar);}
#ask-btn{position:absolute;right:24px;bottom:18px;font-family:var(--serif);
  font-size:24px;letter-spacing:.15em;padding:8px 28px;background:var(--bronze);
  color:var(--paper);border:none;border-radius:6px;cursor:pointer;}
#ask-btn:hover{filter:brightness(1.15);}
#pain{animation:painFlash 1.1s infinite;}
@keyframes painFlash{0%,100%{opacity:0}30%,70%{opacity:1}}
```

JS：

```js
document.getElementById('ask-btn').addEventListener('click', () => game.goto('burn'));
```

- [ ] **Step 3: 自測（回歸）＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (7/7)`（本 Task 純表現層，不加邏輯測試，但不得弄壞既有測試）。
Run: `start .\index.html` → 開始上課 → 點三下看完字卡 → Expected:
1. 三句字卡淡入淡出、右下角「點一下，繼續」呼吸提示；
2. 宮殿夜景：月亮、雙簷剪影、窗內暖光、商王抱臉剪影、硃砂紅痛感閃線一閃一閃；
3. 對白條可讀、「問天 →」按鈕點擊後切到 burn（目前空黑，正常）。

- [ ] **Step 4: Commit**

```powershell
git add index.html; git commit -m "feat: intro title cards and palace night scene with aching king"
```

---

### Task 5: 燒龜甲互動（充能模型＋火光＋粒子）

**Files:**
- Modify: `index.html`（burn section 的 HTML＋CSS；JS：純函式 `burnStep`、粒子模組、互動綁定）

**Interfaces:**
- Consumes: `game`、`audio.startFire/unlock`、`scene:enter`
- Produces:
  - 純函式 `burnStep(charge, dt, holding) -> number`（0..1 夾住；按住 3 秒充滿，放開 1.2 秒衰減完）
  - `embers.spawnAt(x, y, rate)`（Canvas 火星粒子，`#embers` 畫布 1280×720）
  - `crackSequence()` 的呼叫點：充能滿後放開觸發（函式本體在 Task 6 實作；本 Task 先放 stub `function crackSequence(){ console.log('crack!'); }`，Task 6 取代之）

- [ ] **Step 1: 先寫 burnStep 的失敗自測**

```js
test('burnStep 按住 3 秒充滿', () => {
  let c = 0;
  for (let i = 0; i < 30; i++) c = burnStep(c, 0.1, true);
  assertEq(c, 1);
});
test('burnStep 放開會衰減且不低於 0', () => {
  let c = burnStep(1, 0.5, false);
  assertOk(c < 1 && c > 0, `0.5s 後應介於 0~1，got ${c}`);
  assertEq(burnStep(0, 5, false), 0);
});
test('burnStep 上限夾在 1', () => {
  assertEq(burnStep(0.99, 9, true), 1);
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 三則新測試 `FAIL`（`burnStep is not defined`）。

- [ ] **Step 3: 實作充能模型（純函式，放在自測區塊之前）**

```js
/* ===== 灼燒充能模型 ===== */
const BURN = { chargeTime: 3.0, decayTime: 1.2 };
function burnStep(charge, dt, holding){
  const next = charge + dt * (holding ? 1 / BURN.chargeTime : -1 / BURN.decayTime);
  return Math.max(0, Math.min(1, next));
}
```

- [ ] **Step 4: 驗證通過**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (10/10)`。

- [ ] **Step 5: burn 場景（HTML／CSS）**

HTML——取代空的 burn section：

```html
<section data-scene="burn" id="scene-burn">
  <svg viewBox="0 0 1280 720" width="1280" height="720">
    <defs>
      <radialGradient id="glowGrad" cx="50%" cy="50%" r="50%">
        <stop offset="0%" stop-color="var(--flame)" stop-opacity=".95"/>
        <stop offset="60%" stop-color="var(--flame)" stop-opacity=".35"/>
        <stop offset="100%" stop-color="var(--flame)" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <!-- 火光（充能時淡入） -->
    <circle id="shell-glow" cx="640" cy="380" r="240" fill="url(#glowGrad)" opacity="0"/>
    <!-- 龜腹甲 -->
    <g id="shell" style="cursor:pointer">
      <path d="M640 200
               C 520 200 470 280 470 380 C 470 490 540 560 640 560
               C 740 560 810 490 810 380 C 810 280 760 200 640 200 Z"
            fill="var(--paper)" stroke="#c9b992" stroke-width="4"/>
      <!-- 腹甲天然紋路 -->
      <g stroke="#c9b992" stroke-width="3" fill="none" opacity=".8">
        <line x1="640" y1="205" x2="640" y2="555"/>
        <path d="M480 320 L800 320"/><path d="M475 430 L805 430"/>
        <path d="M520 250 L760 250"/><path d="M520 510 L760 510"/>
      </g>
      <!-- 裂紋（Task 6 啟用；先隱藏備好） -->
      <g id="cracks" stroke="var(--ink)" stroke-width="5" fill="none"
         stroke-linecap="round" opacity="0">
        <path class="crack" d="M640 380 L610 320 L620 260"/>
        <path class="crack" d="M640 380 L700 350 L750 360"/>
        <path class="crack" d="M640 380 L630 450 L580 500"/>
      </g>
    </g>
  </svg>
  <canvas id="embers" width="1280" height="720"></canvas>
  <p id="burn-hint">按住龜甲，開始灼燒</p>
</section>
```

CSS：

```css
/* --- burn --- */
#embers{position:absolute;inset:0;pointer-events:none;}
#burn-hint{position:absolute;left:50%;bottom:48px;transform:translateX(-50%);
  font-size:30px;letter-spacing:.2em;color:var(--gold);}
#burn-hint.ready{color:var(--cinnabar);font-size:40px;font-weight:bold;
  animation:breathe .5s infinite;}
#scene-burn svg{position:absolute;inset:0;touch-action:none;}
```

- [ ] **Step 6: 粒子模組＋互動綁定（JS，自測區塊之前）**

```js
/* ===== 火星粒子 ===== */
const embers = (() => {
  const cv = document.getElementById('embers');
  const c = cv.getContext('2d');
  let ps = [], last = 0;
  function spawnAt(x, y, rate){
    if (Math.random() > rate * 0.6) return;
    ps.push({ x: x + (Math.random() - 0.5) * 200, y: y + 80,
              vx: (Math.random() - 0.5) * 50, vy: -(60 + Math.random() * 140),
              life: 0.7 + Math.random() * 0.6 });
  }
  function loop(ts){
    const dt = last ? (ts - last) / 1000 : 0; last = ts;
    c.clearRect(0, 0, cv.width, cv.height);
    ps = ps.filter(p => p.life > 0);
    for (const p of ps){
      p.x += p.vx * dt; p.y += p.vy * dt; p.life -= dt;
      c.globalAlpha = Math.max(p.life, 0);
      c.fillStyle = Math.random() < 0.2 ? '#d9a441' : '#e8823a';
      c.beginPath(); c.arc(p.x, p.y, 2.5, 0, 7); c.fill();
    }
    c.globalAlpha = 1;
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
  return { spawnAt };
})();

/* ===== 燒龜甲互動 ===== */
function crackSequence(){ console.log('crack!'); }  /* Task 6 取代 */

(() => {
  const shell = document.getElementById('shell');
  const glow  = document.getElementById('shell-glow');
  const hint  = document.getElementById('burn-hint');
  let holding = false, charge = 0, last = 0, fire = null, done = false, running = false;
  function frame(ts){
    if (!running) return;
    const dt = last ? Math.min((ts - last) / 1000, 0.05) : 0; last = ts;
    charge = burnStep(charge, dt, holding);
    glow.setAttribute('opacity', charge.toFixed(3));
    shell.setAttribute('transform',
      `translate(${(Math.random() - 0.5) * charge * 5},${(Math.random() - 0.5) * charge * 5})`);
    if (holding) embers.spawnAt(640, 380, charge);
    const ready = charge >= 1;
    hint.textContent = ready ? '放開！' : '按住龜甲，開始灼燒';
    hint.classList.toggle('ready', ready);
    requestAnimationFrame(frame);
  }
  shell.addEventListener('pointerdown', e => {
    e.preventDefault();
    if (done) return;
    holding = true;
    fire = audio.startFire();
  });
  addEventListener('pointerup', () => {
    if (!holding) return;
    holding = false;
    if (fire){ fire.stop(); fire = null; }
    if (charge >= 1 && !done){ done = true; running = false; crackSequence(); }
  });
  document.addEventListener('scene:enter', e => {
    if (e.detail.scene === 'burn'){
      charge = 0; last = 0; done = false; running = true;
      requestAnimationFrame(frame);
    } else { running = false; }
  });
})();
```

- [ ] **Step 7: 自測回歸＋視覺／手感檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (10/10)`。
Run: `start .\index.html` → 走到 burn 場景 → Expected:
1. 米白龜腹甲置中、天然紋路可見；
2. 按住：火光漸亮、龜甲微顫、火星往上飄、劈啪聲響起；
3. 中途放開：火光回落、聲音停、可以重按（沒有卡死）；
4. 按滿 3 秒：「放開！」硃砂大字閃動；放開後 Console 出現 `crack!`。

- [ ] **Step 8: Commit**

```powershell
git add index.html; git commit -m "feat: hold-to-burn interaction with charge model, embers, fire SFX"
```

---

### Task 6: 裂紋動畫與「吉」判定

**Files:**
- Modify: `index.html`（取代 `crackSequence` stub；verdict section 的 HTML＋CSS＋JS；stage 震動 keyframes）

**Interfaces:**
- Consumes: Task 5 的 `#cracks .crack` 路徑、`audio.pop/ding/thump`、`game.goto`
- Produces:
  - `crackSequence()`——啪聲＋畫面震動＋三條裂紋依序竄開，1.8 秒後 `game.goto('verdict')`
  - verdict 場景：貞人判詞＋「吉」硃砂印章砸下（thump）＋`#carve-btn`「把它刻下來 →」→ `game.goto('glyph')`

- [ ] **Step 1: 震動與印章 CSS**

```css
/* --- crack & verdict --- */
#stage.shake{animation:shake .4s;}
@keyframes shake{
  0%,100%{transform:translate(-50%,-50%) scale(var(--fit,1));}
  20%{transform:translate(calc(-50% + 10px),calc(-50% - 8px)) scale(var(--fit,1));}
  40%{transform:translate(calc(-50% - 8px),calc(-50% + 6px)) scale(var(--fit,1));}
  60%{transform:translate(calc(-50% + 6px),calc(-50% + 8px)) scale(var(--fit,1));}
  80%{transform:translate(calc(-50% - 4px),calc(-50% - 4px)) scale(var(--fit,1));}
}
#scene-verdict{flex-direction:column;align-items:center;justify-content:center;gap:30px;}
#scene-verdict.active{display:flex;}
#verdict-text{font-size:34px;letter-spacing:.1em;line-height:1.9;text-align:center;max-width:900px;}
#seal{width:200px;height:200px;background:var(--cinnabar);color:var(--paper);
  display:flex;align-items:center;justify-content:center;font-size:130px;font-weight:bold;
  border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.7);
  transform:scale(6) rotate(18deg);opacity:0;}
#seal.stamped{transition:transform .28s cubic-bezier(.2,2,.4,1),opacity .18s;
  transform:scale(1) rotate(-4deg);opacity:1;}
#carve-btn{font-family:var(--serif);font-size:26px;letter-spacing:.18em;padding:10px 40px;
  background:var(--bronze);color:var(--paper);border:none;border-radius:6px;cursor:pointer;
  opacity:0;transition:opacity .5s;}
#carve-btn.show{opacity:1;}
#carve-btn:hover{filter:brightness(1.15);}
```

**注意 `--fit`：** Task 1 的 `fitStage()` 要同步改成寫入 CSS 變數，否則 shake 動畫會覆蓋縮放——

```js
function fitStage(){
  const s = Math.min(innerWidth / 1280, innerHeight / 720);
  const st = document.getElementById('stage');
  st.style.setProperty('--fit', s);
  st.style.transform = `translate(-50%,-50%) scale(${s})`;
}
```

- [ ] **Step 2: verdict 場景 HTML**

```html
<section data-scene="verdict" id="scene-verdict">
  <p id="verdict-text">貞人湊近細看裂紋——<br>「裂紋向上而直，是好兆頭。」</p>
  <div id="seal">吉</div>
  <p style="font-size:26px;color:var(--gold);margin:0;">牙痛，會好。</p>
  <button id="carve-btn">把它刻下來 →</button>
</section>
```

- [ ] **Step 3: 取代 crackSequence stub＋verdict JS**

刪掉 `function crackSequence(){ console.log('crack!'); }`，改為：

```js
function crackSequence(){
  audio.pop();
  document.getElementById('stage').classList.add('shake');
  setTimeout(() => document.getElementById('stage').classList.remove('shake'), 450);
  document.getElementById('cracks').setAttribute('opacity', '1');
  document.querySelectorAll('.crack').forEach((p, i) => {
    const len = p.getTotalLength();
    p.style.strokeDasharray = len;
    p.style.strokeDashoffset = len;
    setTimeout(() => {
      p.style.transition = 'stroke-dashoffset .5s ease-out';
      p.style.strokeDashoffset = 0;
    }, 120 + 160 * i);
  });
  document.getElementById('burn-hint').textContent = '裂開了！';
  setTimeout(() => game.goto('verdict'), 1800);
}
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'verdict') return;
  const seal = document.getElementById('seal');
  seal.classList.remove('stamped');
  document.getElementById('carve-btn').classList.remove('show');
  setTimeout(() => { seal.classList.add('stamped'); audio.thump(); }, 900);
  setTimeout(() => { audio.ding(); document.getElementById('carve-btn').classList.add('show'); }, 1600);
});
document.getElementById('carve-btn').addEventListener('click', () => game.goto('glyph'));
```

- [ ] **Step 4: 自測回歸＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (10/10)`。
Run: `start .\index.html` → 走完燒龜甲 → Expected:
1. 放開瞬間：「啪」聲＋畫面震動＋三條墨黑裂紋從中心依序竄開（描邊生長，不是瞬間出現）；
2. 1.8 秒後切到判定：貞人判詞先出，接著「吉」印從天砸下（thump 悶響、微歪角度、彈性落地），再響 ding、浮出「把它刻下來 →」；
3. 縮放瀏覽器視窗再觸發一次震動，舞台縮放不跑版（`--fit` 生效）。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: crack animation, screen shake, auspicious seal verdict"
```

---

### Task 7: 「齒」字演示、收尾卡與整體 90 秒 QA

**Files:**
- Modify: `index.html`（glyph section 的 HTML＋CSS＋JS）

**Interfaces:**
- Consumes: `scene:enter`、`audio.ding`、CSS tokens
- Produces: glyph 場景——甲骨文「齒」描邊生長 → 交叉淡化成楷書「齒」 → 說明字卡 → 「第一幕・完」收尾卡。切片終點，無後續 goto。

- [ ] **Step 1: glyph 場景 HTML**

```html
<section data-scene="glyph" id="scene-glyph">
  <p class="glyph-caption" id="glyph-cap1">商王把這次占卜，刻在了甲骨上。</p>
  <div id="glyph-box">
    <svg id="oracle-tooth" viewBox="0 0 200 200" width="300" height="300">
      <!-- 甲骨文「齒」：一張開口，裡面畫著牙 -->
      <g fill="none" stroke="var(--paper)" stroke-width="9" stroke-linecap="round">
        <path class="stroke" d="M55 45 Q40 130 100 165 Q160 130 145 45"/>
        <path class="stroke" d="M62 62 L80 112 L100 62 L120 112 L138 62"/>
      </g>
    </svg>
    <div id="kai-tooth">齒</div>
  </div>
  <p class="glyph-caption" id="glyph-cap2">
    一張嘴，裡面畫著牙——<br>這就是 3000 年前的「<strong>齒</strong>」。
  </p>
  <div id="end-card">
    <p>第一幕・完</p>
    <p class="small">（垂直切片到此為止）</p>
  </div>
</section>
```

- [ ] **Step 2: glyph CSS**

```css
/* --- glyph --- */
#scene-glyph{flex-direction:column;align-items:center;justify-content:center;gap:26px;}
#scene-glyph.active{display:flex;}
.glyph-caption{font-size:32px;letter-spacing:.1em;line-height:1.8;text-align:center;
  margin:0;opacity:0;transition:opacity .8s;}
.glyph-caption.show{opacity:1;}
.glyph-caption strong{color:var(--cinnabar);font-size:40px;}
#glyph-box{position:relative;width:300px;height:300px;}
#oracle-tooth{position:absolute;inset:0;transition:opacity 1.2s;}
#kai-tooth{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;
  font-size:210px;color:var(--paper);opacity:0;transition:opacity 1.2s;}
#glyph-box.morph #oracle-tooth{opacity:0;}
#glyph-box.morph #kai-tooth{opacity:1;}
#end-card{position:absolute;inset:0;background:var(--ink);display:flex;flex-direction:column;
  align-items:center;justify-content:center;opacity:0;pointer-events:none;transition:opacity 1.2s;}
#end-card.show{opacity:1;pointer-events:auto;}
#end-card p{font-size:52px;letter-spacing:.3em;margin:8px;}
#end-card .small{font-size:20px;color:var(--gold);letter-spacing:.15em;}
```

- [ ] **Step 3: glyph JS（描邊生長＋morph 時序）**

```js
/* ===== 「齒」字演示 ===== */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'glyph') return;
  const cap1 = document.getElementById('glyph-cap1');
  const cap2 = document.getElementById('glyph-cap2');
  const box  = document.getElementById('glyph-box');
  const end  = document.getElementById('end-card');
  cap1.classList.remove('show'); cap2.classList.remove('show');
  box.classList.remove('morph'); end.classList.remove('show');
  document.querySelectorAll('#oracle-tooth .stroke').forEach(p => {
    const len = p.getTotalLength();
    p.style.transition = 'none';
    p.style.strokeDasharray = len;
    p.style.strokeDashoffset = len;
  });
  setTimeout(() => cap1.classList.add('show'), 400);
  document.querySelectorAll('#oracle-tooth .stroke').forEach((p, i) => {
    setTimeout(() => {
      p.style.transition = 'stroke-dashoffset 1.1s ease-in-out';
      p.style.strokeDashoffset = 0;
    }, 1400 + i * 1100);
  });
  setTimeout(() => { box.classList.add('morph'); audio.ding(); }, 4200);
  setTimeout(() => cap2.classList.add('show'), 5000);
  setTimeout(() => end.classList.add('show'), 9500);
});
```

- [ ] **Step 4: 自測回歸**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (10/10)`。

- [ ] **Step 5: 完整 90 秒 QA（驗收清單）**

Run: `start .\index.html`，從頭完整走一遍並計時。Expected 全部成立：

1. 開始上課 → 三張字卡 → 宮殿 → 問天 → 燒龜甲 → 裂紋 → 吉 → 刻字 → 齒字演示 → 「第一幕・完」，全程無需鍵盤、無 Console 錯誤；
2. 正常節奏走完約 80–110 秒；
3. 甲骨文「齒」兩筆依序描邊生長，morph 成楷書「齒」時 ding 一聲，字卡點題可讀；
4. 全程音效齊備：火劈啪、裂啪、印章 thump、吉 ding、齒 ding；
5. 把視窗縮小到約 800×500 再走一遍，等比縮放、無跑版；
6. 觸控檢查（如有觸控螢幕）：按住龜甲用手指同樣可燒；沒有觸控裝置則用 DevTools device mode 模擬確認 pointer events 有效。

如有任何一項不成立：修復後重跑本清單，再進 Step 6。

- [ ] **Step 6: Commit**

```powershell
git add index.html; git commit -m "feat: oracle-bone tooth glyph reveal and act-one ending card"
```

---

## 完成定義（Phase 1 驗收，對照 spec §8）

- 使用者雙擊 `index.html` 即玩，離線、單檔、零依賴。
- 燒／裂／音效流暢無卡頓。
- 美術風格達到「有設計感」：統一色盤、剪紙剪影、拓片質感。
- **品質不過關即止損，不進 Phase 2。**
