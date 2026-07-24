# 《越王的十年》第二幕垂直切片 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 實作第二幕「臥薪嘗膽」完整體驗——回家 → 逍遙 → 轉折 → 決定 → 雙點互動 → 收尾——作為周朝動畫系列的品質樣品。

**Architecture:** `zhou/shared/style.css` 存周朝色盤 tokens；`zhou/index.html` 內嵌全部 HTML+CSS+JS，場景狀態機＋Web Audio 合成音效＋SVG 剪紙風場景，以 `#selftest` hash 觸發瀏覽器內自測框架。

**Tech Stack:** Vanilla JS、SVG、CSS animation、Web Audio API。零依賴、零建置步驟。

## Global Constraints

- 舞台 1280×720，transform-origin center，等比縮放（`--fit` CSS 變數同步）
- Pointer events 支援觸控＋滑鼠；主要互動目標為平板觸控
- Web Audio 需手勢解鎖；**無音效時所有互動仍可進行**
- 零外部依賴，離線可用，雙擊即開
- 色盤 tokens from `zhou/shared/style.css`（不自行發明顏色）
- 字型：`'Noto Serif TC','DFKai-SB','BiauKai','Microsoft JhengHei',serif`
- 文案繁體中文，短句，不說教
- 驗證指令（於 `C:\Users\user\Desktop\history-animation`）：
  - 自測：`start chrome "$((Resolve-Path .\zhou\index.html).Path)#selftest"`
  - 視覺：`start .\zhou\index.html`

---

### Task 1: 目錄結構＋共用 CSS＋舞台骨架

**Files:**
- Create: `zhou/shared/style.css`
- Create: `zhou/index.html`

- [ ] **Step 1: 建立 `zhou/shared/style.css`**

```css
:root{
  --night:#0e1a1c;
  --slip:#d9ceae;
  --jade:#3d6b5a;
  --gall:#7a8c30;
  --ember:#c4642a;
  --gold:#c9982a;
  --warm:#d4893a;
  --serif:'Noto Serif TC','DFKai-SB','BiauKai','Microsoft JhengHei',serif;
}
```

- [ ] **Step 2: 建立 `zhou/index.html` 骨架**

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>越王的十年・臥薪嘗膽</title>
<link rel="stylesheet" href="shared/style.css">
<style>
html,body{margin:0;height:100%;background:#000;overflow:hidden;font-family:var(--serif);color:var(--slip);}
#stage{position:absolute;left:50%;top:50%;width:1280px;height:720px;
  transform-origin:center;background:var(--night);overflow:hidden;}
section[data-scene]{position:absolute;inset:0;display:none;}
section[data-scene].active{display:block;}

/* start */
#scene-start{flex-direction:column;align-items:center;justify-content:center;gap:32px;}
#scene-start.active{display:flex;}
#scene-start h1{font-size:64px;letter-spacing:.18em;margin:0;color:var(--slip);}
#scene-start p{font-size:22px;letter-spacing:.3em;color:var(--gold);margin:0;}
#start-btn{font-family:var(--serif);font-size:26px;letter-spacing:.2em;
  padding:12px 52px;background:var(--jade);color:var(--slip);
  border:2px solid var(--gold);border-radius:6px;cursor:pointer;}
#start-btn:hover{filter:brightness(1.15);}

@keyframes breathe{0%,100%{opacity:.2}50%{opacity:.9}}
@keyframes fadeIn{from{opacity:0}to{opacity:1}}
</style>
</head>
<body>
<div id="stage">
  <section data-scene="start" id="scene-start" class="active">
    <h1>越王的十年</h1>
    <p>臥薪嘗膽・春秋動畫</p>
    <button id="start-btn">開始上課</button>
  </section>
  <section data-scene="homecoming" id="scene-homecoming"></section>
  <section data-scene="comfort"    id="scene-comfort"></section>
  <section data-scene="turning"    id="scene-turning"></section>
  <section data-scene="decision"   id="scene-decision"></section>
  <section data-scene="ritual"     id="scene-ritual"></section>
  <section data-scene="ending"     id="scene-ending"></section>
</div>
<script>
'use strict';

/* ===== 舞台縮放 ===== */
function fitStage(){
  const s = Math.min(innerWidth/1280, innerHeight/720);
  const st = document.getElementById('stage');
  st.style.setProperty('--fit', s);
  st.style.transform = `translate(-50%,-50%) scale(${s})`;
}
addEventListener('resize', fitStage);
fitStage();

/* ===== 自測框架 ===== */
const selfTests = [];
function test(name, fn){ selfTests.push({name, fn}); }
function assertEq(a, b){ if(a!==b) throw new Error(`expected ${b}, got ${a}`); }
function assertOk(c, m){ if(!c) throw new Error(m||'assertion failed'); }
function runSelfTests(){
  const lines=[]; let passed=0;
  for(const t of selfTests){
    try{ t.fn(); lines.push(`PASS  ${t.name}`); passed++; }
    catch(e){ lines.push(`FAIL  ${t.name} — ${e.message}`); }
  }
  lines.push('', passed===selfTests.length
    ? `ALL PASS (${passed}/${selfTests.length})`
    : `${passed}/${selfTests.length} passed`);
  const pre=document.createElement('pre');
  pre.style.cssText='position:fixed;inset:0;margin:0;background:#000;color:#4f4;z-index:99;padding:20px;font:15px/1.6 Consolas,monospace;overflow:auto;';
  pre.textContent=lines.join('\n');
  document.body.appendChild(pre);
}
addEventListener('load',()=>{ if(location.hash==='#selftest') runSelfTests(); });

/* 骨架驗證 */
test('stage 含 7 個場景 section',()=>{
  assertEq(document.querySelectorAll('section[data-scene]').length, 7);
});

</script>
</body>
</html>
```

- [ ] **Step 3: 驗證**

Run: `start chrome "$((Resolve-Path .\zhou\index.html).Path)#selftest"`
Expected: `ALL PASS (1/1)`

Run: `start .\zhou\index.html`
Expected: 黑底、「越王的十年」大字、jade 色「開始上課」按鈕、縮放視窗等比不跑版

- [ ] **Step 4: Commit**

```powershell
git add zhou/; git commit -m "feat: zhou dynasty folder, CSS tokens, stage scaffold"
```

---

### Task 2: 場景狀態機

**Files:**
- Modify: `zhou/index.html`（`<script>` 內，自測區塊之前）

- [ ] **Step 1: 先寫失敗的自測（加在 `<script>` 尾部）**

```js
test('game.goto 切換 active 與 current',()=>{
  game.goto('ritual');
  assertEq(game.current,'ritual');
  assertEq(document.querySelectorAll('section[data-scene].active').length,1);
  assertEq(document.querySelector('section[data-scene].active').dataset.scene,'ritual');
});
test('game.goto 未知場景回傳 false 且不變',()=>{
  game.goto('comfort');
  assertEq(game.goto('nonsense'),false);
  assertEq(game.current,'comfort');
});
test('game.next 依序前進',()=>{
  game.goto('start'); game.next();
  assertEq(game.current,'homecoming');
});
test('scene:enter 事件帶 detail.scene',()=>{
  let got=null;
  const h=e=>{got=e.detail.scene;};
  document.addEventListener('scene:enter',h);
  game.goto('ending');
  document.removeEventListener('scene:enter',h);
  assertEq(got,'ending');
});
```

- [ ] **Step 2: 驗證失敗**

Run: refresh `#selftest` → Expected: 4 new FAIL（`game is not defined`）

- [ ] **Step 3: 實作狀態機（加在自測區塊之前）**

```js
/* ===== 場景狀態機 ===== */
const SCENES=['start','homecoming','comfort','turning','decision','ritual','ending'];
const game={
  index:0,
  get current(){ return SCENES[this.index]; },
  goto(name){
    const i=SCENES.indexOf(name);
    if(i===-1) return false;
    this.index=i;
    document.querySelectorAll('section[data-scene]').forEach(s=>
      s.classList.toggle('active', s.dataset.scene===name));
    document.dispatchEvent(new CustomEvent('scene:enter',{detail:{scene:name}}));
    return true;
  },
  next(){ if(this.index<SCENES.length-1) this.goto(SCENES[this.index+1]); }
};
addEventListener('load',()=>{ if(location.hash!=='#selftest') game.goto('start'); });
document.getElementById('start-btn').addEventListener('click',()=>game.goto('homecoming'));
```

- [ ] **Step 4: 驗證通過**

Run: refresh `#selftest` → Expected: `ALL PASS (5/5)`
Run: `start .\zhou\index.html` → 點「開始上課」→ start 畫面消失（homecoming 空黑，正常）

- [ ] **Step 5: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: zhou scene state machine"
```

---

### Task 3: Web Audio 引擎

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: 先寫失敗的自測**

```js
test('audio 未解鎖時所有方法安全 no-op',()=>{
  assertEq(audio.ctx,null);
  audio.resolve();
  const h1=audio.startWind();
  assertOk(typeof h1.stop==='function','startWind 應回傳 {stop}');
  h1.stop();
  const h2=audio.startDrum();
  assertOk(typeof h2.setIntensity==='function','startDrum 應回傳 {setIntensity,stop}');
  h2.stop();
  const h3=audio.startWarm();
  assertOk(typeof h3.stop==='function','startWarm 應回傳 {stop}');
  h3.stop();
});
test('audio.unlock 建立 AudioContext',()=>{
  audio.unlock();
  assertOk(audio.ctx!==null,'unlock 後 ctx 不應為 null');
  assertEq(typeof audio.ctx.createOscillator,'function');
});
```

- [ ] **Step 2: 驗證失敗**

Run: refresh `#selftest` → Expected: 2 new FAIL（`audio is not defined`）

- [ ] **Step 3: 實作音效引擎（加在自測區塊之前）**

```js
/* ===== Web Audio 音效引擎 ===== */
const audio={
  ctx:null, enabled:true,
  unlock(){
    if(!this.ctx) this.ctx=new(window.AudioContext||window.webkitAudioContext)();
    if(this.ctx.state==='suspended') this.ctx.resume();
  },
  _noise(dur){
    const n=Math.floor(this.ctx.sampleRate*dur);
    const buf=this.ctx.createBuffer(1,n,this.ctx.sampleRate);
    const d=buf.getChannelData(0);
    for(let i=0;i<n;i++) d[i]=Math.random()*2-1;
    return buf;
  },
  /* 竹篁風聲：低頻帶通噪音，持續 */
  startWind(){
    if(!this.ctx||!this.enabled) return {stop(){}};
    const src=this.ctx.createBufferSource();
    src.buffer=this._noise(3); src.loop=true;
    const bp=this.ctx.createBiquadFilter();
    bp.type='bandpass'; bp.frequency.value=320; bp.Q.value=0.5;
    const g=this.ctx.createGain(); g.gain.value=0.07;
    src.connect(bp); bp.connect(g); g.connect(this.ctx.destination);
    src.start();
    return {stop:()=>{
      g.gain.setTargetAtTime(0,this.ctx.currentTime,0.3);
      src.stop(this.ctx.currentTime+0.6);
    }};
  },
  /* 遠方鼓聲：單擊脈衝，intensity(0~1)控制觸發頻率 */
  startDrum(){
    if(!this.ctx||!this.enabled) return {setIntensity(){},stop(){}};
    let intensity=0, timer=null;
    const ctx=this.ctx;
    const beat=()=>{
      if(intensity<=0||!ctx) return;
      const o=ctx.createOscillator();
      o.frequency.setValueAtTime(80,ctx.currentTime);
      o.frequency.exponentialRampToValueAtTime(38,ctx.currentTime+0.18);
      const g=ctx.createGain();
      g.gain.setValueAtTime(0.25+intensity*0.35,ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001,ctx.currentTime+0.22);
      o.connect(g); g.connect(ctx.destination);
      o.start(ctx.currentTime); o.stop(ctx.currentTime+0.22);
      const nextMs=1800-intensity*1400+Math.random()*180;
      timer=setTimeout(beat,nextMs);
    };
    return {
      setIntensity(v){
        intensity=Math.max(0,Math.min(1,v));
        if(!timer&&intensity>0) beat();
      },
      stop(){ clearTimeout(timer); timer=null; intensity=0; }
    };
  },
  /* 溫暖室內環境音：低頻噪音 */
  startWarm(){
    if(!this.ctx||!this.enabled) return {stop(){}};
    const src=this.ctx.createBufferSource();
    src.buffer=this._noise(4); src.loop=true;
    const lp=this.ctx.createBiquadFilter();
    lp.type='lowpass'; lp.frequency.value=180;
    const g=this.ctx.createGain(); g.gain.value=0.04;
    src.connect(lp); lp.connect(g); g.connect(this.ctx.destination);
    src.start();
    return {stop:()=>{
      g.gain.setTargetAtTime(0,this.ctx.currentTime,0.5);
      src.stop(this.ctx.currentTime+1);
    }};
  },
  /* 決心完成：上升和弦 */
  resolve(){
    if(!this.ctx||!this.enabled) return;
    const t=this.ctx.currentTime;
    [330,440,550,660].forEach((f,i)=>{
      const o=this.ctx.createOscillator(); o.type='sine'; o.frequency.value=f;
      const g=this.ctx.createGain();
      g.gain.setValueAtTime(0,t+i*0.07);
      g.gain.linearRampToValueAtTime(0.14,t+i*0.07+0.06);
      g.gain.exponentialRampToValueAtTime(0.001,t+i*0.07+0.9);
      o.connect(g); g.connect(this.ctx.destination);
      o.start(t+i*0.07); o.stop(t+i*0.07+0.9);
    });
  }
};
document.getElementById('start-btn').addEventListener('click',()=>audio.unlock());
```

- [ ] **Step 4: 驗證通過**

Run: refresh `#selftest` → Expected: `ALL PASS (7/7)`
Run: `start .\zhou\index.html` → DevTools console:
```js
audio.unlock(); audio.resolve();
```
Expected: 聽到上升四音和弦。
```js
const w=audio.startWind(); setTimeout(()=>w.stop(),3000);
```
Expected: 竹篁風聲起、3 秒後漸弱收掉。

- [ ] **Step 5: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: zhou web audio engine (wind/drum/warm/resolve)"
```

---

### Task 4: 場景 A — 回家

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: 加入 CSS（`<style>` 內）**

```css
/* --- homecoming --- */
#scene-homecoming{cursor:pointer;}
#scene-homecoming svg{position:absolute;inset:0;}
#homecoming-card{
  position:absolute;left:50%;bottom:64px;transform:translateX(-50%);
  font-size:36px;letter-spacing:.18em;color:var(--slip);
  text-shadow:0 2px 16px rgba(0,0,0,.9);
  animation:fadeIn .9s ease both;
}
#homecoming-hint{
  position:absolute;right:60px;bottom:36px;font-size:18px;
  color:var(--gold);animation:breathe 2s infinite;
}
```

- [ ] **Step 2: 取代空的 homecoming section**

```html
<section data-scene="homecoming" id="scene-homecoming">
  <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
    <rect width="1280" height="720" fill="var(--night)"/>
    <defs>
      <radialGradient id="homeGlow" cx="50%" cy="35%" r="58%">
        <stop offset="0%" stop-color="#d4893a" stop-opacity=".38"/>
        <stop offset="100%" stop-color="#d4893a" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <rect width="1280" height="720" fill="url(#homeGlow)"/>
    <!-- 地板 -->
    <rect x="0" y="570" width="1280" height="150" fill="#111008"/>
    <!-- 燈籠左 -->
    <g transform="translate(280,70)">
      <line x1="0" y1="0" x2="0" y2="28" stroke="var(--slip)" stroke-width="2"/>
      <ellipse cx="0" cy="62" rx="22" ry="36" fill="var(--warm)" opacity=".88"/>
      <line x1="0" y1="98" x2="0" y2="114" stroke="var(--slip)" stroke-width="2"/>
    </g>
    <!-- 燈籠右 -->
    <g transform="translate(1000,70)">
      <line x1="0" y1="0" x2="0" y2="28" stroke="var(--slip)" stroke-width="2"/>
      <ellipse cx="0" cy="62" rx="22" ry="36" fill="var(--warm)" opacity=".88"/>
      <line x1="0" y1="98" x2="0" y2="114" stroke="var(--slip)" stroke-width="2"/>
    </g>
    <!-- 跪迎臣子 A -->
    <g transform="translate(220,490)" fill="#0c1808">
      <circle cx="0" cy="-58" r="18"/>
      <rect x="-15" y="-38" width="30" height="48" rx="6"/>
      <rect x="-13" y="10" width="26" height="16" rx="4"
            transform="rotate(-32 0 10)"/>
      <line x1="-14" y1="2" x2="-48" y2="22"
            stroke="#0c1808" stroke-width="8" stroke-linecap="round"/>
    </g>
    <!-- 跪迎臣子 B -->
    <g transform="translate(440,508)" fill="#0c1808">
      <circle cx="0" cy="-58" r="18"/>
      <rect x="-15" y="-38" width="30" height="48" rx="6"/>
      <rect x="-13" y="10" width="26" height="16" rx="4"
            transform="rotate(-32 0 10)"/>
      <line x1="-14" y1="2" x2="-48" y2="22"
            stroke="#0c1808" stroke-width="8" stroke-linecap="round"/>
    </g>
    <!-- 勾踐走入（右側） -->
    <g transform="translate(860,290)" fill="#0c1808">
      <rect x="-18" y="-116" width="36" height="10" rx="2"/>
      <path d="M-14,-116 L-10,-132 L0,-126 L10,-132 L14,-116 Z"/>
      <circle cx="0" cy="-84" r="26"/>
      <path d="M-30,-58 C-32,10 -30,100 -22,150 L22,150 C30,100 32,10 30,-58 Z"/>
      <ellipse cx="-14" cy="158" rx="16" ry="8"/>
      <ellipse cx="18" cy="154" rx="16" ry="8"/>
    </g>
  </svg>
  <div id="homecoming-card">終於，夫差放他回家了。</div>
  <div id="homecoming-hint">點一下，繼續</div>
</section>
```

- [ ] **Step 3: JS（加在自測區塊之前）**

```js
/* ===== Scene A: 回家 ===== */
let warmHandle = null;
document.addEventListener('scene:enter', e=>{
  if(e.detail.scene==='homecoming') warmHandle=audio.startWarm();
  if(e.detail.scene==='turning'){
    if(warmHandle){ warmHandle.stop(); warmHandle=null; }
  }
});
document.getElementById('scene-homecoming').addEventListener('click',()=>{
  game.goto('comfort');
});
```

- [ ] **Step 4: 視覺檢查**

Run: `start .\zhou\index.html` → 開始上課 → Expected:
1. 暖光宮殿、左右燈籠、兩名臣子跪伏
2. 勾踐剪影站右側，有王冠
3. 字卡「終於，夫差放他回家了。」淡入
4. 右下呼吸閃爍提示
5. 點一下切到 comfort（空黑，正常）

- [ ] **Step 5: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: scene-homecoming warm palace welcome"
```

---

### Task 5: 場景 B（逍遙）＋ 場景 C（轉折）

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: CSS**

```css
/* --- comfort --- */
#scene-comfort svg{position:absolute;inset:0;}

/* --- turning --- */
#scene-turning{cursor:pointer;}
#scene-turning svg{position:absolute;inset:0;}
#flashback-overlay{
  position:absolute;inset:0;background:#080f14;
  opacity:0;pointer-events:none;transition:opacity .07s;
}
#flashback-overlay.show{opacity:.88;}
#turning-hint{
  position:absolute;right:60px;bottom:36px;font-size:18px;
  color:var(--gold);animation:breathe 2s infinite;
}
```

- [ ] **Step 2: 取代空的 comfort section**

```html
<section data-scene="comfort" id="scene-comfort">
  <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
    <rect width="1280" height="720" fill="var(--night)"/>
    <defs>
      <radialGradient id="comfortGlow" cx="48%" cy="38%" r="52%">
        <stop offset="0%" stop-color="#d4893a" stop-opacity=".42"/>
        <stop offset="100%" stop-color="#d4893a" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <rect width="1280" height="720" fill="url(#comfortGlow)"/>
    <!-- 地板 -->
    <rect x="0" y="580" width="1280" height="140" fill="#111008"/>
    <!-- 榻 -->
    <rect x="340" y="460" width="560" height="108" rx="20" fill="#3a2810"/>
    <rect x="328" y="442" width="584" height="28" rx="8" fill="#4a3218"/>
    <!-- 靠背 -->
    <rect x="790" y="392" width="96" height="108" rx="14" fill="#4e3616"/>
    <!-- 勾踐坐榻：放鬆後靠 -->
    <g transform="translate(560,340)" fill="#0c1808">
      <rect x="-19" y="-32" width="38" height="10" rx="2"/>
      <path d="M-15,-32 L-11,-48 L0,-42 L11,-48 L15,-32 Z"/>
      <circle cx="0" cy="0" r="27"/>
      <!-- 後靠身體 -->
      <path d="M-30,26 C-34,88 -28,158 -18,186 L18,186 C28,158 34,88 30,26 Z"
            transform="rotate(14 0 100)"/>
      <!-- 手拿碗 -->
      <ellipse cx="58" cy="128" rx="26" ry="10"/>
      <line x1="26" y1="100" x2="58" y2="128"
            stroke="#0c1808" stroke-width="9" stroke-linecap="round"/>
    </g>
    <!-- 侍者（右） -->
    <g transform="translate(940,278)" fill="#0c1808">
      <circle cx="0" cy="0" r="20"/>
      <rect x="-17" y="20" width="34" height="96" rx="8"/>
      <rect x="-28" y="-8" width="56" height="7" rx="3"
            transform="rotate(-8 0 0)"/>
    </g>
  </svg>
</section>
```

- [ ] **Step 3: 取代空的 turning section**

```html
<section data-scene="turning" id="scene-turning">
  <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
    <rect width="1280" height="720" fill="var(--night)"/>
    <defs>
      <radialGradient id="turningGlow" cx="48%" cy="38%" r="52%">
        <stop offset="0%" stop-color="#d4893a" stop-opacity=".18"/>
        <stop offset="100%" stop-color="#d4893a" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <rect width="1280" height="720" fill="url(#turningGlow)"/>
    <rect x="0" y="580" width="1280" height="140" fill="#111008"/>
    <!-- 榻（同 comfort） -->
    <rect x="340" y="460" width="560" height="108" rx="20" fill="#3a2810"/>
    <rect x="328" y="442" width="584" height="28" rx="8" fill="#4a3218"/>
    <rect x="790" y="392" width="96" height="108" rx="14" fill="#4e3616"/>
    <!-- 碗放在榻前地板 -->
    <ellipse cx="640" cy="498" rx="28" ry="10" fill="#0c1808"/>
    <!-- 勾踐坐直，不拿碗 -->
    <g transform="translate(560,310)" fill="#0c1808">
      <rect x="-19" y="-32" width="38" height="10" rx="2"/>
      <path d="M-15,-32 L-11,-48 L0,-42 L11,-48 L15,-32 Z"/>
      <circle cx="0" cy="0" r="27"/>
      <path d="M-30,26 C-32,80 -30,150 -20,178 L20,178 C30,150 32,80 30,26 Z"/>
    </g>
    <!-- 窗（右上） -->
    <g transform="translate(1040,160)">
      <rect x="0" y="0" width="108" height="148" fill="#07101a"/>
      <rect x="6" y="6" width="96" height="136" fill="#0e1e2c" opacity=".7"/>
      <line x1="54" y1="6" x2="54" y2="142" stroke="#182430" stroke-width="3"/>
      <line x1="6" y1="74" x2="102" y2="74" stroke="#182430" stroke-width="3"/>
    </g>
  </svg>
  <!-- 閃回覆蓋（馬廄冷調） -->
  <div id="flashback-overlay">
    <svg viewBox="0 0 1280 720" width="1280" height="720"
         style="position:absolute;inset:0;">
      <rect width="1280" height="720" fill="#080f14"/>
      <!-- 欄杆 -->
      <g stroke="#162030" stroke-width="12" fill="none">
        <line x1="180" y1="0" x2="180" y2="720"/>
        <line x1="380" y1="0" x2="380" y2="720"/>
        <line x1="580" y1="0" x2="580" y2="720"/>
        <line x1="780" y1="0" x2="780" y2="720"/>
        <line x1="980" y1="0" x2="980" y2="720"/>
        <line x1="1180" y1="0" x2="1180" y2="720"/>
      </g>
      <rect x="0" y="296" width="1280" height="12" fill="#162030"/>
      <rect x="0" y="456" width="1280" height="12" fill="#162030"/>
      <!-- 吳王（左，高大） -->
      <g transform="translate(280,100)" fill="#18283c" opacity=".85">
        <circle cx="0" cy="0" r="34"/>
        <path d="M-44,34 C-50,148 -44,248 -34,300 L34,300 C44,248 50,148 44,34 Z"/>
      </g>
      <!-- 勾踐跪姿（中右，小） -->
      <g transform="translate(760,380)" fill="#22344a" opacity=".85">
        <circle cx="0" cy="-42" r="22"/>
        <path d="M-24,-20 C-24,24 -20,54 -12,64 L12,64 C20,54 24,24 24,-20 Z"/>
        <ellipse cx="-20" cy="70" rx="18" ry="7"/>
        <ellipse cx="20" cy="70" rx="18" ry="7"/>
      </g>
    </svg>
  </div>
  <div id="turning-hint">點一下，繼續</div>
</section>
```

- [ ] **Step 4: JS（加在自測區塊之前）**

```js
/* ===== Scene B: 逍遙 ===== */
document.addEventListener('scene:enter', e=>{
  if(e.detail.scene!=='comfort') return;
  setTimeout(()=>{ if(game.current==='comfort') game.goto('turning'); }, 2600);
});

/* ===== Scene C: 轉折 ===== */
document.addEventListener('scene:enter', e=>{
  if(e.detail.scene!=='turning') return;
  // 1.3 秒後馬廄閃回
  setTimeout(()=>{
    const ov=document.getElementById('flashback-overlay');
    ov.classList.add('show');
    setTimeout(()=>ov.classList.remove('show'), 650);
  }, 1300);
});
document.getElementById('scene-turning').addEventListener('click',()=>{
  if(game.current==='turning') game.goto('decision');
});
```

- [ ] **Step 5: 自測回歸**

Run: refresh `#selftest` → Expected: `ALL PASS (7/7)`（本 task 無新邏輯，不得弄壞既有測試）

- [ ] **Step 6: 視覺檢查**

Run: `start .\zhou\index.html` → 走到 comfort → Expected:
1. 暖光、榻、侍者、勾踐後靠拿碗
2. 2.6 秒後自動切到 turning
3. turning：坐直、碗在地、窗在右上、暖光更暗
4. 1.3 秒後：冷調馬廄閃回（欄杆＋吳王＋跪姿勾踐）0.65 秒消失
5. 點一下切到 decision（空黑，正常）

- [ ] **Step 7: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: scene-comfort and scene-turning with stable flashback"
```

---

### Task 6: 場景 D — 決定

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: CSS**

```css
/* --- decision --- */
#scene-decision svg{position:absolute;inset:0;}
#xin-wrap,#dan-wrap{
  position:absolute;bottom:72px;
  opacity:0;transition:transform .85s cubic-bezier(.18,1.2,.4,1),opacity .7s;
}
#xin-wrap{ left:148px; transform:translateX(-110px); }
#dan-wrap{ right:148px; transform:translateX(110px); }
#xin-wrap.arrived,#dan-wrap.arrived{ transform:translateX(0); opacity:1; }
```

- [ ] **Step 2: 取代空的 decision section**

```html
<section data-scene="decision" id="scene-decision">
  <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
    <rect width="1280" height="720" fill="var(--night)"/>
    <!-- 地板 -->
    <rect x="0" y="580" width="1280" height="140" fill="#0d1208"/>
    <!-- 被推到右側的榻（殘影） -->
    <rect x="830" y="462" width="380" height="90" rx="14"
          fill="#3a2810" opacity=".3"/>
    <!-- 勾踐站立 -->
    <g transform="translate(640,238)" fill="#0c1808">
      <rect x="-19" y="-32" width="38" height="10" rx="2"/>
      <path d="M-15,-32 L-11,-48 L0,-42 L11,-48 L15,-32 Z"/>
      <circle cx="0" cy="0" r="27"/>
      <path d="M-32,26 C-34,84 -32,162 -22,202 L22,202 C32,162 34,84 32,26 Z"/>
      <ellipse cx="-16" cy="210" rx="17" ry="8"/>
      <ellipse cx="16" cy="210" rx="17" ry="8"/>
    </g>
  </svg>
  <!-- 薪草（左下滑入） -->
  <div id="xin-wrap">
    <svg viewBox="0 0 200 130" width="200" height="130">
      <g stroke="var(--ember)" stroke-linecap="round" stroke-width="5">
        <line x1="22" y1="18" x2="72" y2="118"/>
        <line x1="40" y1="12" x2="88" y2="116"/>
        <line x1="58" y1="10" x2="104" y2="114"/>
        <line x1="76" y1="12" x2="118" y2="116"/>
        <line x1="94" y1="16" x2="132" y2="116"/>
        <line x1="110" y1="20" x2="146" y2="114"/>
        <line x1="128" y1="18" x2="160" y2="116"/>
        <line x1="146" y1="14" x2="174" y2="118"/>
      </g>
      <rect x="58" y="56" width="84" height="10" rx="5" fill="#7a4820"/>
    </svg>
  </div>
  <!-- 苦膽（右下滑入） -->
  <div id="dan-wrap">
    <svg viewBox="0 0 140 170" width="140" height="170">
      <ellipse cx="70" cy="110" rx="54" ry="52" fill="var(--gall)" opacity=".88"/>
      <path d="M44,66 Q70,22 96,66"
            fill="none" stroke="var(--gall)" stroke-width="13"
            stroke-linecap="round" opacity=".88"/>
      <path d="M58,56 Q70,44 82,56"
            fill="none" stroke="#4a5a18" stroke-width="6"/>
      <ellipse cx="50" cy="90" rx="12" ry="18" fill="#fff" opacity=".12"/>
    </svg>
  </div>
</section>
```

- [ ] **Step 3: JS（加在自測區塊之前）**

```js
/* ===== Scene D: 決定 ===== */
document.addEventListener('scene:enter', e=>{
  if(e.detail.scene!=='decision') return;
  const xin=document.getElementById('xin-wrap');
  const dan=document.getElementById('dan-wrap');
  xin.classList.remove('arrived'); dan.classList.remove('arrived');
  // 0.5 秒後薪和膽滑入底部
  setTimeout(()=>{
    xin.classList.add('arrived'); dan.classList.add('arrived');
    // 入場後 1.3 秒自動進入互動
    setTimeout(()=>{ if(game.current==='decision') game.goto('ritual'); }, 1300);
  }, 500);
});
```

- [ ] **Step 4: 視覺檢查**

Run: `start .\zhou\index.html` → 走到 decision → Expected:
1. 勾踐站立剪影居中，右側半透明的被推開的榻
2. 0.5 秒後薪草從左、苦膽從右以彈性動畫滑入底部
3. 1.8 秒後自動切到 ritual（空黑，正常）

- [ ] **Step 5: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: scene-decision standing goujian with xin and dan slide-in"
```

---

### Task 7: 場景 E — 雙點互動（核心）

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: 先寫純函式的失敗自測**

```js
test('driftXin 範圍 -40~40',()=>{
  for(let i=0;i<30;i++){
    const x=driftXin(Math.random()*Math.PI*2);
    assertOk(x>=-40&&x<=40,`driftXin out of range: ${x}`);
  }
});
test('driftDan 範圍 -28~28',()=>{
  for(let i=0;i<30;i++){
    const y=driftDan(Math.random()*Math.PI*2);
    assertOk(y>=-28&&y<=28,`driftDan out of range: ${y}`);
  }
});
test('ritualProgress 雙按 9 秒充滿',()=>{
  let p=0;
  for(let i=0;i<90;i++) p=ritualProgress(p,0.1,true);
  assertEq(p,1);
});
test('ritualProgress 放開不退',()=>{
  const p=ritualProgress(0.6,5,false);
  assertEq(p,0.6);
});
test('ritualProgress 夾住上限',()=>{
  assertEq(ritualProgress(0.98,9,true),1);
});
```

- [ ] **Step 2: 驗證失敗**

Run: refresh `#selftest` → Expected: 5 new FAIL（`driftXin is not defined`）

- [ ] **Step 3: 實作純函式（加在自測區塊之前）**

```js
/* ===== 雙點互動：純函式 ===== */
function driftXin(phase){ return Math.sin(phase)*40; }
function driftDan(phase){ return Math.sin(phase)*28; }
function ritualProgress(progress,dt,bothHeld){
  if(!bothHeld) return progress;
  return Math.min(1, progress+dt/9);
}
```

- [ ] **Step 4: 驗證通過**

Run: refresh `#selftest` → Expected: `ALL PASS (12/12)`

- [ ] **Step 5: CSS**

```css
/* --- ritual --- */
#scene-ritual{touch-action:none;}
#ritual-memory{position:absolute;inset:0;pointer-events:none;}
#ritual-glow{
  position:absolute;left:50%;top:50%;
  width:700px;height:700px;margin-left:-350px;margin-top:-350px;
  border-radius:50%;
  background:radial-gradient(circle,rgba(201,152,42,.55) 0%,rgba(201,152,42,0) 68%);
  opacity:0;pointer-events:none;transition:opacity .35s;
}
#goujian-rising{
  position:absolute;left:50%;top:48px;transform:translateX(-50%);
  pointer-events:none;
}
#ritual-xin{
  position:absolute;left:148px;bottom:58px;
  width:210px;height:140px;cursor:pointer;touch-action:none;
}
#ritual-dan{
  position:absolute;right:148px;bottom:50px;
  width:168px;height:188px;cursor:pointer;touch-action:none;
}
```

- [ ] **Step 6: 取代空的 ritual section**

```html
<section data-scene="ritual" id="scene-ritual">
  <!-- 上方：馬廄閃回（固定背景） -->
  <svg id="ritual-memory" viewBox="0 0 1280 720" width="1280" height="720"
       aria-hidden="true">
    <rect width="1280" height="720" fill="#080f14"/>
    <g stroke="#152030" stroke-width="10" fill="none" opacity=".75">
      <line x1="130" y1="0" x2="130" y2="560"/>
      <line x1="320" y1="0" x2="320" y2="560"/>
      <line x1="510" y1="0" x2="510" y2="560"/>
      <line x1="700" y1="0" x2="700" y2="560"/>
      <line x1="890" y1="0" x2="890" y2="560"/>
      <line x1="1080" y1="0" x2="1080" y2="560"/>
    </g>
    <rect x="0" y="310" width="1280" height="10" fill="#152030" opacity=".7"/>
    <!-- 吳王（左，高大俯視） -->
    <g transform="translate(260,90)" fill="#16273a" opacity=".82">
      <circle cx="0" cy="0" r="36"/>
      <path d="M-46,36 C-52,154 -46,258 -36,312 L36,312 C46,258 52,154 46,36 Z"/>
    </g>
    <!-- 勾踐跪（右偏中，小） -->
    <g transform="translate(840,390)" fill="#203248" opacity=".82">
      <circle cx="0" cy="-44" r="24"/>
      <path d="M-26,-22 C-26,26 -22,58 -14,68 L14,68 C22,58 26,26 26,-22 Z"/>
      <ellipse cx="-22" cy="74" rx="19" ry="8"/>
      <ellipse cx="22" cy="74" rx="19" ry="8"/>
    </g>
  </svg>

  <!-- 決心金光（隨進度擴散） -->
  <div id="ritual-glow"></div>

  <!-- 勾踐剪影（中央，姿態隨進度改變） -->
  <svg id="goujian-rising" viewBox="0 0 160 340"
       width="150" height="318" aria-hidden="true">
    <defs>
      <radialGradient id="risingGlow" cx="50%" cy="35%" r="55%">
        <stop offset="0%" stop-color="var(--gold)" stop-opacity=".3"/>
        <stop offset="100%" stop-color="var(--gold)" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <circle cx="80" cy="160" r="130" fill="url(#risingGlow)"/>
    <!-- 王冠 -->
    <rect id="gj-crown-base" x="60" y="14" width="40" height="10" rx="2"
          fill="var(--slip)" opacity=".92"/>
    <path id="gj-crown-tips" d="M64,14 L68,0 L80,6 L92,0 L96,14 Z"
          fill="var(--gold)" opacity=".92"/>
    <!-- 頭 -->
    <circle id="gj-head" cx="80" cy="46" r="28" fill="var(--slip)" opacity=".92"/>
    <!-- 上身＋腿（整體 group 用 transform 做姿態變化） -->
    <g id="gj-body-group">
      <path id="gj-body"
            d="M50,74 C48,134 50,208 56,252 L104,252 C110,208 112,134 110,74 Z"
            fill="var(--slip)" opacity=".92"/>
      <ellipse cx="62" cy="260" rx="18" ry="9" fill="var(--slip)" opacity=".92"/>
      <ellipse cx="98" cy="260" rx="18" ry="9" fill="var(--slip)" opacity=".92"/>
    </g>
  </svg>

  <!-- 薪草互動點（左下） -->
  <div id="ritual-xin">
    <svg viewBox="0 0 210 140" width="210" height="140">
      <g stroke="var(--ember)" stroke-linecap="round" stroke-width="5.5">
        <line x1="18" y1="18" x2="68" y2="128"/>
        <line x1="36" y1="12" x2="84" y2="126"/>
        <line x1="54" y1="10" x2="100" y2="124"/>
        <line x1="72" y1="12" x2="116" y2="126"/>
        <line x1="90" y1="16" x2="132" y2="124"/>
        <line x1="108" y1="18" x2="148" y2="124"/>
        <line x1="126" y1="16" x2="164" y2="126"/>
        <line x1="144" y1="12" x2="180" y2="128"/>
        <line x1="162" y1="16" x2="194" y2="126"/>
      </g>
      <rect x="54" y="56" width="102" height="11" rx="5" fill="#7a4820"/>
    </svg>
  </div>

  <!-- 苦膽互動點（右下） -->
  <div id="ritual-dan">
    <svg viewBox="0 0 168 188" width="168" height="188">
      <ellipse cx="84" cy="124" rx="62" ry="58" fill="var(--gall)" opacity=".9"/>
      <path d="M52,74 Q84,26 116,74"
            fill="none" stroke="var(--gall)" stroke-width="14"
            stroke-linecap="round" opacity=".9"/>
      <path d="M66,62 Q84,50 102,62"
            fill="none" stroke="#4a5a18" stroke-width="7"/>
      <ellipse cx="60" cy="100" rx="14" ry="20" fill="#fff" opacity=".13"/>
    </svg>
  </div>
</section>
```

- [ ] **Step 7: 互動 JS（加在自測區塊之前）**

```js
/* ===== Scene E: 雙點互動 ===== */
(()=>{
  const xinEl    = document.getElementById('ritual-xin');
  const danEl    = document.getElementById('ritual-dan');
  const glowEl   = document.getElementById('ritual-glow');
  const bodyGrp  = document.getElementById('gj-body-group');
  const headEl   = document.getElementById('gj-head');
  const crownB   = document.getElementById('gj-crown-base');
  const crownT   = document.getElementById('gj-crown-tips');

  let xinHeld=false, danHeld=false;
  let progress=0, lastTs=0;
  let xinPhase=0, danPhase=Math.PI*0.55;
  let running=false, completed=false;
  let windH=null, drumH=null;

  function applyPosture(p){
    /* 上身從前傾 -26° 旋轉到 0°（以腰部 80,165 為軸），整體上移 */
    const angle=-26*(1-p);
    const lift=-36*p;
    bodyGrp.setAttribute('transform',
      `rotate(${angle.toFixed(2)} 80 165) translate(0 ${lift.toFixed(1)})`);
    const hy=(46+lift*0.55).toFixed(1);
    headEl.setAttribute('cy',hy);
    const cy=(14+lift*0.55).toFixed(1);
    crownB.setAttribute('y',cy);
    crownT.setAttribute('transform',`translate(0 ${(lift*0.55).toFixed(1)})`);
  }

  function driftLoop(ts){
    if(!running) return;
    xinPhase+=0.0009; danPhase+=0.0012;
    xinEl.style.transform=`translateX(${driftXin(xinPhase).toFixed(1)}px)`;
    danEl.style.transform=`translateY(${driftDan(danPhase).toFixed(1)}px)`;
    requestAnimationFrame(driftLoop);
  }

  function progressLoop(ts){
    if(!running) return;
    const dt=lastTs ? Math.min((ts-lastTs)/1000,0.05) : 0; lastTs=ts;
    progress=ritualProgress(progress,dt,xinHeld&&danHeld);
    if(drumH) drumH.setIntensity(progress);
    glowEl.style.opacity=String((progress*0.82).toFixed(3));
    applyPosture(progress);
    if(progress>=1&&!completed){
      completed=true; running=false;
      if(windH){windH.stop();windH=null;}
      if(drumH){drumH.stop();drumH=null;}
      audio.resolve();
      setTimeout(()=>game.goto('ending'),820);
    } else {
      requestAnimationFrame(progressLoop);
    }
  }

  /* pointer 事件：分別捕獲各自的 pointer */
  xinEl.addEventListener('pointerdown',e=>{
    e.preventDefault(); xinHeld=true; xinEl.setPointerCapture(e.pointerId);
  });
  xinEl.addEventListener('pointerup',  ()=>{ xinHeld=false; });
  xinEl.addEventListener('pointercancel',()=>{ xinHeld=false; });
  danEl.addEventListener('pointerdown',e=>{
    e.preventDefault(); danHeld=true; danEl.setPointerCapture(e.pointerId);
  });
  danEl.addEventListener('pointerup',  ()=>{ danHeld=false; });
  danEl.addEventListener('pointercancel',()=>{ danHeld=false; });

  document.addEventListener('scene:enter',e=>{
    if(e.detail.scene==='ritual'){
      progress=0; lastTs=0; xinHeld=false; danHeld=false;
      completed=false; xinPhase=0; danPhase=Math.PI*0.55;
      applyPosture(0);
      glowEl.style.opacity='0';
      xinEl.style.transform=''; danEl.style.transform='';
      running=true;
      windH=audio.startWind();
      drumH=audio.startDrum();
      requestAnimationFrame(driftLoop);
      requestAnimationFrame(progressLoop);
    } else {
      running=false;
      if(windH){windH.stop();windH=null;}
      if(drumH){drumH.stop();drumH=null;}
    }
  });
})();
```

- [ ] **Step 8: 自測回歸**

Run: refresh `#selftest` → Expected: `ALL PASS (12/12)`

- [ ] **Step 9: 視覺＋手感檢查**

Run: `start .\zhou\index.html` → 走到 ritual → Expected:
1. 上方：馬廄冷調閃回（欄杆＋吳王＋跪姿勾踐）清晰可見
2. 中央：勾踐剪影，初始前傾姿態
3. 左下：薪草，來回緩慢左右漂移
4. 右下：苦膽，上下緩慢漂移（不同速）
5. 同時按住兩個 → 勾踐漸漸挺直，金光從中央擴散，鼓聲漸頻
6. 放開任一個 → 進度停住，不退
7. 約 9 秒撐滿 → resolve 和弦，0.82 秒後切到 ending（空黑，正常）
8. DevTools device mode → 雙指同時按住兩點可觸發

- [ ] **Step 10: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: dual-press ritual with drift, posture animation, drum build"
```

---

### Task 8: 場景 F — 收尾

**Files:**
- Modify: `zhou/index.html`

- [ ] **Step 1: CSS**

```css
/* --- ending --- */
#scene-ending{flex-direction:column;align-items:center;justify-content:center;gap:44px;}
#scene-ending.active{display:flex;}
#ending-figure{opacity:0;transition:opacity 1.1s;}
#ending-figure.show{opacity:1;}
#ending-text{
  font-size:118px;letter-spacing:.45em;color:var(--gold);margin:0;
  opacity:0;transition:opacity 1.3s;
  text-shadow:0 0 48px rgba(201,152,42,.55);
}
#ending-text.show{opacity:1;}
#ending-btn{
  font-family:var(--serif);font-size:26px;letter-spacing:.2em;
  padding:10px 42px;background:var(--jade);color:var(--slip);
  border:none;border-radius:6px;cursor:pointer;
  opacity:0;transition:opacity .8s;
}
#ending-btn.show{opacity:1;}
#ending-btn:hover{filter:brightness(1.15);}
```

- [ ] **Step 2: 取代空的 ending section**

```html
<section data-scene="ending" id="scene-ending">
  <svg id="ending-figure" viewBox="0 0 160 340" width="130" height="276">
    <defs>
      <radialGradient id="endGlow" cx="50%" cy="38%" r="52%">
        <stop offset="0%" stop-color="var(--gold)" stop-opacity=".52"/>
        <stop offset="100%" stop-color="var(--gold)" stop-opacity="0"/>
      </radialGradient>
    </defs>
    <circle cx="80" cy="170" r="148" fill="url(#endGlow)"/>
    <rect x="60" y="14" width="40" height="10" rx="2" fill="var(--slip)"/>
    <path d="M64,14 L68,0 L80,6 L92,0 L96,14 Z" fill="var(--gold)"/>
    <circle cx="80" cy="46" r="28" fill="var(--slip)"/>
    <path d="M50,74 C48,134 50,208 56,252 L104,252 C110,208 112,134 110,74 Z"
          fill="var(--slip)"/>
    <ellipse cx="62" cy="260" rx="18" ry="9" fill="var(--slip)"/>
    <ellipse cx="98" cy="260" rx="18" ry="9" fill="var(--slip)"/>
  </svg>
  <p id="ending-text">十年。</p>
  <button id="ending-btn">繼續 →</button>
</section>
```

- [ ] **Step 3: JS（加在自測區塊之前）**

```js
/* ===== Scene F: 收尾 ===== */
document.addEventListener('scene:enter',e=>{
  if(e.detail.scene!=='ending') return;
  const fig=document.getElementById('ending-figure');
  const txt=document.getElementById('ending-text');
  const btn=document.getElementById('ending-btn');
  fig.classList.remove('show'); txt.classList.remove('show'); btn.classList.remove('show');
  setTimeout(()=>fig.classList.add('show'), 280);
  setTimeout(()=>txt.classList.add('show'), 980);
  setTimeout(()=>btn.classList.add('show'), 2500);
});
/* 第三幕尚未實作：按鈕暫無動作 */
document.getElementById('ending-btn').addEventListener('click',()=>{});
```

- [ ] **Step 4: 自測回歸**

Run: refresh `#selftest` → Expected: `ALL PASS (12/12)`

- [ ] **Step 5: 視覺檢查**

Run: `start .\zhou\index.html` → 走完互動 → Expected:
1. 勾踐站立剪影（帶金光光暈）淡入
2. 0.98 秒後「十年。」金色大字淡入
3. 2.5 秒後「繼續 →」按鈕淡入（點了無動作，正常）

- [ ] **Step 6: Commit**

```powershell
git add zhou/index.html; git commit -m "feat: scene-ending ten-year card with standing goujian"
```

---

### Task 9: 整體 QA

- [ ] **Step 1: 完整走一遍（驗收清單）**

Run: `start .\zhou\index.html` → 從頭走到尾，全部 Expected 必須成立：

1. **開始畫面**：「越王的十年」大字、jade 綠按鈕、縮放視窗等比不跑版
2. **Scene A**：宮殿暖光、燈籠、跪伏臣子、勾踐右側剪影、字卡可讀、有室內溫暖聲
3. **Scene B**：勾踐坐榻拿碗、侍者、暖光比 A 更亮；2.6 秒後自動切轉折
4. **Scene C**：坐直、碗在地、音效停（暖聲消失）、1.3 秒後馬廄閃回 0.65 秒消失、右下提示、點繼續
5. **Scene D**：站立勾踐、右側半透明榻殘影、0.5 秒後薪膽從兩側彈入底部
6. **Scene E（核心）**：
   - 馬廄閃回（欄杆＋兩剪影）在上方清楚可見
   - 薪草左右漂移、苦膽上下漂移、節奏不同步
   - 雙指同時按住 → 勾踐從前傾挺身、金光擴散、鼓聲漸快
   - 放開一個 → 進度停住不退
   - 撐滿 → 和弦解決音效 → 0.82 秒後切收尾
7. **Scene F**：勾踐站立淡入→「十年。」金字→「繼續 →」按鈕
8. **縮放測試**：縮小視窗至 800×500 重走一遍，舞台等比縮放不跑版
9. **無音效測試**：不點「開始上課」直接 console 呼叫 `game.goto('ritual')`，互動仍可進行

- [ ] **Step 2: 觸控測試**

Run: Chrome DevTools → device mode → 選一個平板解析度（如 1024×768）→ 走 Scene E
Expected: 雙指可同時觸發兩個點，漂移可見，進度累積正常

- [ ] **Step 3: Final commit**

```powershell
git add zhou/; git commit -m "feat: zhou dynasty act-2 vertical slice complete — 臥薪嘗膽"
```

---

## 完成定義

- 雙擊 `zhou/index.html` 即玩，離線、`shared/style.css` 同目錄、零其他依賴
- 雙點漂移互動：薪膽不同步漂移明顯，手感流暢，進度不退
- 敘事完整：回家→逍遙→轉折→決定→互動→收尾，不靠老師補劇情
- 美術風格：剪紙剪影，周朝色盤（jade/ember/gall/gold）與商朝硃砂系有明確區隔
- 自測全過：`ALL PASS (12/12)`
