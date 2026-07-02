# 《商王的一天》Phase 2 完整四幕 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Phase 1 第一幕的基礎上完成其餘三幕（第二幕拖拉解碼、第三幕三選一派將＋驗辭、第四幕磨龍骨）＋章節圓點跳轉＋背景音樂，整體節奏約 20 分鐘。

**Architecture:** 延續單一 `index.html`。燒龜甲場景**參數化復用**（`setBurn(next, chargeTime, hint)` 設定占卜去向，第二、三幕以較短充能時間作儀式復用）；新增 8 個場景 section 接在既有 6 個之後；共用 `.dialog`／`.next-btn` 樣式類；三個新純函式（`actOfScene`、`hitTarget`、`grindStep`）走 TDD，表現層走視覺檢查清單。

**Tech Stack:** Vanilla JS、SVG、CSS animation、Canvas 2D、Web Audio API。零依賴、零建置。

## Global Constraints

- 交付物為**單一 HTML 檔** `index.html`，零外部依賴、零建置步驟、離線可用（spec §6）。
- **學生零開放式輸入**：僅按住、拖拉、點選、滑動磨擦（spec §3）。
- 同時支援滑鼠與觸控 → 一律使用 pointer events，可拖曳／可磨擦元素加 `touch-action:none`，並處理 `pointercancel`（Phase 1 教訓，commit 6fcc1e8）（spec §6）。
- Web Audio 需使用者手勢解鎖；**無音訊時所有互動仍可進行**（spec §6）。
- 舞台以 1280×720 橫向為基準，等比縮放至視窗（spec §6）。
- 色盤（spec §7）：墨黑 `#171310`、宣紙米白 `#efe3cc`、硃砂紅 `#b23a2a`、青銅綠 `#56705f`、火焰橘 `#e8823a`、高光金 `#d9a441`。
- 美術風格：剪紙／皮影＋甲骨拓片墨色；字體 `'Noto Serif TC','DFKai-SB','BiauKai','Microsoft JhengHei',serif`（spec §7）。
- 文案一律繁體中文，小學生可讀（短句、無艱澀詞）。
- 婦好領兵、驗辭結構為史實呈現；王懿榮藥鋪故事以「相傳」語氣處理（spec §9）。
- 背景音樂：程式合成五聲音階環境循環，音量低於音效，可整段關閉（spec §7）。
- 章節跳轉：左下四個小圓點，家教用，不做存檔（spec §4 節奏保險、§10）。
- **不做**：多堂課／存檔／成績、開放式輸入、語音旁白、影片、外部素材、手機直式（spec §10）。
- CSS 注意（Phase 1 教訓）：ID 選擇器一律不寫 `display`，顯示由「`ID.active`」控制，否則特異性會壓過 `.active` 切換。
- 測試方式：瀏覽器開 `index.html#selftest` 跑內嵌自測（邏輯）；開 `index.html` 走視覺檢查清單（表現層）。無 Node、無測試框架。
- 驗證指令（PowerShell，於 `C:\Users\User\Desktop\TRY`）：
  - 自測：`start chrome "$((Resolve-Path .\index.html).Path)#selftest"`（無 Chrome 則用 `msedge`）
  - 視覺：`start .\index.html`
- 程式碼插入位置慣例（單檔內的區域）：
  - CSS：加在 `</style>` 之前。
  - HTML：新場景 section 加在 `<section data-scene="glyph" …>…</section>` 之後、`</div>`（`#stage` 收尾）之前。
  - JS 實作：加在 `/* 音效測試：未實作時應 FAIL */` 註解**之前**。
  - JS 自測：加在檔案最末既有測試之後、`</script>` 之前。

---

### Task 1: 場景擴充、燒灼參數化、第一幕接軌

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Phase 1 的 `SCENES`、`game`、`BURN`、`burnStep`、`crackSequence`、`#cracks .crack`、`#end-card`、自測 API
- Produces（後續 Task 依賴）:
  - `SCENES` 擴為 14 個場景：`['start','intro','palace','burn','verdict','glyph','act2-intro','act2-decode','act3-intro','act3-choose','act3-verify','act4-intro','act4-grind','act4-finale']`，且 14 個空／實 section 都存在
  - `burnCtx = { next, hint }`（物件，全域）與 `setBurn(next, chargeTime, hint)`——設定占卜結束去向、充能秒數（寫入 `BURN.chargeTime`）、提示字。`chargeTime` 省略時還原 `3.0`，`hint` 省略時還原 `'按住龜甲，開始灼燒'`
  - burn 場景進入時自動重設裂紋（可重複占卜）
  - 共用按鈕樣式 `.next-btn`（bronze 底、serif、hover 提亮）
  - `#to-act2-btn`：第一幕收尾卡上的「天亮了，下一幕 →」→ `game.goto('act2-intro')`

- [ ] **Step 1: 改自測（既有 6 場景測試改為 14）＋新增 setBurn 失敗測試**

把既有這則測試：

```js
test('stage 含 6 個場景 section', () => {
  assertEq(document.querySelectorAll('section[data-scene]').length, 6);
});
```

改成：

```js
test('stage 含 14 個場景 section', () => {
  assertEq(document.querySelectorAll('section[data-scene]').length, 14);
});
```

並在檔案最末測試區（`</script>` 之前）新增：

```js
test('setBurn 設定占卜去向、充能時間與提示', () => {
  setBurn('act2-decode', 1.6, '再問一次');
  assertEq(burnCtx.next, 'act2-decode');
  assertEq(BURN.chargeTime, 1.6);
  assertEq(burnCtx.hint, '再問一次');
  setBurn('verdict');                       // 省略參數 = 還原預設
  assertEq(BURN.chargeTime, 3.0);
  assertEq(burnCtx.hint, '按住龜甲，開始灼燒');
});
```

- [ ] **Step 2: 驗證失敗**

Run: `start chrome "$((Resolve-Path .\index.html).Path)#selftest"`
Expected: `FAIL  stage 含 14 個場景 section — expected 14, got 6` 與 `FAIL  setBurn …（setBurn is not defined）`，其餘 9 則 PASS。

- [ ] **Step 3: 加 8 個空場景 section＋SCENES 擴充**

HTML——在 glyph 的 `</section>` 之後、`</div>` 之前加入：

```html
  <section data-scene="act2-intro" id="scene-act2-intro"></section>
  <section data-scene="act2-decode" id="scene-act2-decode"></section>
  <section data-scene="act3-intro" id="scene-act3-intro"></section>
  <section data-scene="act3-choose" id="scene-act3-choose"></section>
  <section data-scene="act3-verify" id="scene-act3-verify"></section>
  <section data-scene="act4-intro" id="scene-act4-intro"></section>
  <section data-scene="act4-grind" id="scene-act4-grind"></section>
  <section data-scene="act4-finale" id="scene-act4-finale"></section>
```

JS——`SCENES` 改為：

```js
const SCENES = ['start','intro','palace','burn','verdict','glyph',
  'act2-intro','act2-decode',
  'act3-intro','act3-choose','act3-verify',
  'act4-intro','act4-grind','act4-finale'];
```

- [ ] **Step 4: 實作 setBurn＋burn 場景參數化＋裂紋重設**

JS——加在 `const BURN = …` 與 `burnStep` 之後：

```js
/* ===== 占卜情境（燒灼場景可跨幕復用） ===== */
const burnCtx = { next: 'verdict', hint: '按住龜甲，開始灼燒' };
function setBurn(next, chargeTime, hint){
  burnCtx.next = next;
  burnCtx.hint = hint || '按住龜甲，開始灼燒';
  BURN.chargeTime = chargeTime || 3.0;
}
/* 每次進 burn 場景先復原裂紋，才能重複占卜 */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'burn') return;
  document.getElementById('cracks').setAttribute('opacity', '0');
  document.querySelectorAll('.crack').forEach(p => {
    const len = p.getTotalLength();
    p.style.transition = 'none';
    p.style.strokeDasharray = len;
    p.style.strokeDashoffset = len;
  });
});
```

`crackSequence()` 內把結尾這行：

```js
  setTimeout(() => game.goto('verdict'), 1800);
```

改成：

```js
  setTimeout(() => game.goto(burnCtx.next), 1800);
```

burn 互動 IIFE 的 `frame()` 內把：

```js
    hint.textContent = ready ? '放開！' : '按住龜甲，開始灼燒';
```

改成：

```js
    hint.textContent = ready ? '放開！' : burnCtx.hint;
```

`#ask-btn`（第一幕問天）改為明確設定情境，避免跳幕後殘留舊設定：

```js
document.getElementById('ask-btn').addEventListener('click', () => {
  setBurn('verdict');
  game.goto('burn');
});
```

- [ ] **Step 5: 共用 .next-btn 樣式＋第一幕收尾卡接第二幕＋標題更新**

CSS——加在 `</style>` 之前：

```css
/* --- shared --- */
.next-btn{font-family:var(--serif);font-size:26px;letter-spacing:.18em;padding:10px 40px;
  background:var(--bronze);color:var(--paper);border:none;border-radius:6px;cursor:pointer;}
.next-btn:hover{filter:brightness(1.15);}
#end-card .next-btn{margin-top:26px;}
```

HTML——`#end-card` 內容改為（拿掉「垂直切片到此為止」）：

```html
    <div id="end-card">
      <p>第一幕・完</p>
      <button id="to-act2-btn" class="next-btn">天亮了，下一幕 →</button>
    </div>
```

`<title>` 改為 `商王的一天`；start 畫面副標 `<p>商朝互動歷史課・第一幕</p>` 改為 `<p>商朝互動歷史課</p>`。

JS——加在 `/* 音效測試 */` 之前：

```js
document.getElementById('to-act2-btn').addEventListener('click', () => game.goto('act2-intro'));
```

- [ ] **Step 6: 驗證通過＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (11/11)`。
Run: `start .\index.html` → 走完第一幕 → Expected:
1. 收尾卡出現「天亮了，下一幕 →」，點擊後切到 act2-intro（目前空黑，正常）；
2. 用 Console 執行 `setBurn('verdict'); game.goto('burn')` 回燒龜甲：裂紋已復原、可再燒一次、燒完進 verdict。

- [ ] **Step 7: Commit**

```powershell
git add index.html; git commit -m "feat: extend scenes to four acts, parameterize burn ritual for reuse"
```

---

### Task 2: 章節圓點導覽

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 1 的 14 場景、`burnCtx`、`game.goto`、`scene:enter`
- Produces:
  - 純函式 `actOfScene(scene, burnNext) -> 1|2|3|4`——依場景名前綴 `act2/3/4` 判定幕次，其餘為 1；共用的 `'burn'` 場景改看 `burnNext` 的前綴
  - `#chapters`：左下四個圓點（平常低調、hover 浮現），點擊跳至各幕開頭（`intro` / `act2-intro` / `act3-intro` / `act4-intro`）；當前幕的點填滿金色；start 畫面隱藏

- [ ] **Step 1: 先寫失敗的自測（檔案最末測試區）**

```js
test('actOfScene 依前綴判定幕次', () => {
  assertEq(actOfScene('palace', 'verdict'), 1);
  assertEq(actOfScene('act2-decode', 'verdict'), 2);
  assertEq(actOfScene('act3-verify', 'verdict'), 3);
  assertEq(actOfScene('act4-grind', 'verdict'), 4);
});
test('actOfScene 共用的 burn 場景看占卜去向', () => {
  assertEq(actOfScene('burn', 'verdict'), 1);
  assertEq(actOfScene('burn', 'act2-decode'), 2);
  assertEq(actOfScene('burn', 'act3-choose'), 3);
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 兩則新測試 `FAIL`（`actOfScene is not defined`），其餘 11 則 PASS。

- [ ] **Step 3: 實作 actOfScene＋圓點導覽**

JS——加在 `/* 音效測試 */` 之前：

```js
/* ===== 章節導覽 ===== */
function actOfScene(scene, burnNext){
  const key = scene === 'burn' ? burnNext : scene;
  const m = /^act(\d)/.exec(key);
  return m ? Number(m[1]) : 1;
}
const chapterNav = document.getElementById('chapters');
chapterNav.querySelectorAll('button').forEach(b =>
  b.addEventListener('click', () => game.goto(b.dataset.target)));
document.addEventListener('scene:enter', e => {
  const name = e.detail.scene;
  chapterNav.classList.toggle('hidden', name === 'start');
  const act = actOfScene(name, burnCtx.next);
  chapterNav.querySelectorAll('button').forEach((b, i) =>
    b.classList.toggle('on', i === act - 1));
});
```

HTML——加在 8 個新 section 之後、`</div>`（`#stage`）之前：

```html
  <nav id="chapters" class="hidden" aria-label="章節跳轉">
    <button data-target="intro" title="第一幕・牙痛的國王"></button>
    <button data-target="act2-intro" title="第二幕・今年會下雨嗎"></button>
    <button data-target="act3-intro" title="第三幕・羌方入侵"></button>
    <button data-target="act4-intro" title="第四幕・1899 年藥鋪"></button>
  </nav>
```

CSS——加在 `</style>` 之前：

```css
/* --- chapters --- */
#chapters{position:absolute;left:16px;bottom:14px;display:flex;gap:10px;z-index:10;
  opacity:.25;transition:opacity .3s;}
#chapters:hover{opacity:1;}
#chapters.hidden{display:none;}
#chapters button{width:14px;height:14px;border-radius:50%;border:1px solid var(--gold);
  background:transparent;cursor:pointer;padding:0;}
#chapters button.on{background:var(--gold);}
```

- [ ] **Step 4: 驗證通過＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (13/13)`。
Run: `start .\index.html` → Expected:
1. start 畫面沒有圓點；開始上課後左下出現四個淡圓點，滑過變清楚；
2. 第一幕進行中第 1 點填滿金色；點第 2 點跳到 act2-intro（空黑）且第 2 點填滿；
3. 點第 1 點回到 intro 字卡，第一幕仍可正常走完。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: chapter dot navigation with act highlighting"
```

---

### Task 3: 背景音樂引擎與開關鈕

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `audio.ctx`／`audio.unlock`／`audio.enabled`、`#start-btn`
- Produces:
  - `music.start()`（未解鎖 audio 時安全 no-op）、`music.stop()`、`music.toggle() -> boolean`（回傳切換後是否播放中）、`music.playing`、`music.timer`
  - 五聲音階（宮商角徵羽）合成編鐘隨機音＋每四拍一記低鼓，音量遠低於音效
  - `#music-btn`：右上「樂」圓鈕切換；關閉時劃線變淡。按「開始上課」自動開始播放

- [ ] **Step 1: 先寫失敗的自測（檔案最末測試區）**

```js
test('music start/stop/toggle 狀態機', () => {
  /* audio 已在前面的測試解鎖，ctx 存在 */
  music.start();
  assertOk(music.playing, 'start 後應為 playing');
  assertOk(music.timer !== null, 'start 後應有排程 timer');
  music.start();                     // 重複 start 不應疊加
  assertEq(music.toggle(), false);   // 關
  assertEq(music.playing, false);
  assertEq(music.timer, null);
  assertEq(music.toggle(), true);    // 再開
  music.stop();
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 新測試 `FAIL`（`music is not defined`），其餘 13 則 PASS。

- [ ] **Step 3: 實作音樂引擎＋開關鈕**

JS——加在 `/* 音效測試 */` 之前：

```js
/* ===== 背景音樂（五聲音階環境循環，音量低於音效） ===== */
const PENTA = [261.63, 293.66, 329.63, 392.00, 440.00]; /* 宮商角徵羽（C 調） */
const music = {
  playing: false, timer: null, step: 0,
  _bell(freq, t){
    [[freq, 0.05], [freq * 2.7, 0.012]].forEach(([f, v]) => {
      const o = audio.ctx.createOscillator(); o.type = 'sine'; o.frequency.value = f;
      const g = audio.ctx.createGain();
      g.gain.setValueAtTime(v, t);
      g.gain.exponentialRampToValueAtTime(0.0001, t + 2.4);
      o.connect(g); g.connect(audio.ctx.destination); o.start(t); o.stop(t + 2.4);
    });
  },
  _drum(t){
    const o = audio.ctx.createOscillator();
    o.frequency.setValueAtTime(80, t);
    o.frequency.exponentialRampToValueAtTime(40, t + 0.3);
    const g = audio.ctx.createGain();
    g.gain.setValueAtTime(0.09, t);
    g.gain.exponentialRampToValueAtTime(0.0001, t + 0.35);
    o.connect(g); g.connect(audio.ctx.destination); o.start(t); o.stop(t + 0.35);
  },
  start(){
    if (!audio.ctx || this.playing) return;
    this.playing = true;
    this.timer = setInterval(() => {
      if (!audio.enabled) return;
      const t = audio.ctx.currentTime + 0.05;
      this._bell(PENTA[Math.floor(Math.random() * PENTA.length)], t);
      if (this.step % 4 === 0) this._drum(t);
      this.step++;
    }, 1300);
  },
  stop(){ clearInterval(this.timer); this.timer = null; this.playing = false; },
  toggle(){ if (this.playing) this.stop(); else this.start(); return this.playing; }
};
document.getElementById('music-btn').addEventListener('click', () => {
  audio.unlock();
  document.getElementById('music-btn').classList.toggle('off', !music.toggle());
});
/* start-btn 已先綁 audio.unlock（Phase 1），此監聽在其後執行，ctx 必已存在 */
document.getElementById('start-btn').addEventListener('click', () => music.start());
```

HTML——加在 `<nav id="chapters">` 之後、`</div>` 之前：

```html
  <button id="music-btn" title="背景音樂開關">樂</button>
```

CSS——加在 `</style>` 之前：

```css
/* --- music --- */
#music-btn{position:absolute;right:16px;top:14px;z-index:10;width:44px;height:44px;
  font-family:var(--serif);font-size:22px;color:var(--gold);background:transparent;
  border:1px solid var(--gold);border-radius:50%;cursor:pointer;opacity:.5;}
#music-btn:hover{opacity:1;}
#music-btn.off{text-decoration:line-through;opacity:.3;}
```

- [ ] **Step 4: 驗證通過＋人耳檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (14/14)`（測試中會短暫出聲，正常）。
Run: `start .\index.html` → 開始上課 → Expected:
1. 每 1.3 秒一記輕柔鐘音、每四拍一記低鼓，明顯小聲於互動音效；
2. 點右上「樂」鈕：音樂停、鈕劃線變淡；再點恢復；
3. 音樂開著走完第一幕，火聲／裂聲／ding 都清楚壓過音樂。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: pentatonic ambient music loop with toggle button"
```

---

### Task 4: 第二幕開場與儀式占卜接線

**Files:**
- Modify: `index.html`（填 `#scene-act2-intro`；新增共用 `.dialog` 樣式）

**Interfaces:**
- Consumes: `setBurn`、`game.goto`、`.next-btn`、CSS tokens
- Produces:
  - 共用對白條樣式 `.dialog`（宣紙底、置底置中；第三、四幕沿用）
  - act2-intro：晨光田野場景＋對白條＋`#act2-ask-btn`「問天 →」→ `setBurn('act2-decode', 1.6, '按住龜甲——問今年的雨')` 後 `game.goto('burn')`

- [ ] **Step 1: act2-intro 場景（HTML／CSS／JS）**

HTML——填入空的 act2-intro section：

```html
  <section data-scene="act2-intro" id="scene-act2-intro">
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <!-- 晨光 -->
      <rect width="1280" height="720" fill="#2a2218"/>
      <circle cx="1040" cy="150" r="70" fill="var(--flame)" opacity=".85"/>
      <!-- 田野 -->
      <path d="M0 520 L1280 480 L1280 720 L0 720 Z" fill="#241e18"/>
      <!-- 禾苗列 -->
      <g stroke="var(--bronze)" stroke-width="4" stroke-linecap="round" fill="none" opacity=".9">
        <path d="M200 640 v-50 M200 610 l-22 -26 M200 610 l22 -26"/>
        <path d="M400 655 v-50 M400 625 l-22 -26 M400 625 l22 -26"/>
        <path d="M620 645 v-50 M620 615 l-22 -26 M620 615 l22 -26"/>
        <path d="M850 655 v-50 M850 625 l-22 -26 M850 625 l22 -26"/>
        <path d="M1060 640 v-50 M1060 610 l-22 -26 M1060 610 l22 -26"/>
      </g>
    </svg>
    <div class="dialog">
      <p class="speaker">第二幕・今年會下雨嗎</p>
      <p>天亮了。牙痛是小事——現在是<strong>國家大事</strong>：今年的收成，全靠雨。</p>
      <p>貞人：「王，再問一次天吧——<strong>今年，會下雨嗎？</strong>」</p>
      <button id="act2-ask-btn" class="next-btn">問天 →</button>
    </div>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- shared dialog --- */
.dialog{position:absolute;left:50%;bottom:28px;transform:translateX(-50%);
  width:900px;background:var(--paper);color:var(--ink);border-radius:8px;
  padding:20px 32px 24px;box-shadow:0 6px 24px rgba(0,0,0,.6);}
.dialog p{margin:6px 0;font-size:26px;line-height:1.6;}
.dialog .speaker{color:var(--cinnabar);font-size:22px;}
.dialog strong{color:var(--cinnabar);}
.dialog .next-btn{display:block;margin:12px 0 0 auto;}
```

JS——加在 `/* 音效測試 */` 之前：

```js
/* ===== 第二幕 ===== */
document.getElementById('act2-ask-btn').addEventListener('click', () => {
  setBurn('act2-decode', 1.6, '按住龜甲——問今年的雨');
  game.goto('burn');
});
```

- [ ] **Step 2: 自測回歸＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (14/14)`。
Run: `start .\index.html` → 走完第一幕 → 「天亮了，下一幕 →」→ Expected:
1. 晨光田野：橘色朝陽、五株禾苗剪影、對白條標明第二幕；
2. 「問天 →」→ 燒龜甲提示變成「按住龜甲——問今年的雨」、約 1.6 秒即充滿（比第一幕快，儀式感）；
3. 裂開後切到 act2-decode（空黑，正常）；
4. 由圓點跳回第一幕再問天，充能恢復 3 秒、結束進 verdict（`setBurn` 互不污染）。

- [ ] **Step 3: Commit**

```powershell
git add index.html; git commit -m "feat: act2 dawn intro with ritual burn rewire"
```

---

### Task 5: 第二幕主互動——拖拉配對解碼

**Files:**
- Modify: `index.html`（填 `#scene-act2-decode`；純函式 `hitTarget`；拖拉互動）

**Interfaces:**
- Consumes: `game.goto`、`audio.ding`／`audio.thump`、`scene:enter`、`.next-btn`
- Produces:
  - 純函式 `hitTarget(x, y, targets, r) -> index|-1`——回傳距離 `(x,y)` 在半徑 `r` 內、`done === false` 的第一個目標索引
  - `stagePoint(clientX, clientY) -> {x,y}`——視窗座標轉 1280×720 舞台座標（第四幕磨龍骨也可用此概念，但 canvas 另有自己的換算）
  - 解碼完成 → `#decode-next-btn`「下一幕 →」→ `game.goto('act3-intro')`

- [ ] **Step 1: 先寫 hitTarget 失敗自測（檔案最末測試區）**

```js
test('hitTarget 命中半徑內未完成目標', () => {
  const ts = [{x:100,y:100,done:false},{x:300,y:100,done:false}];
  assertEq(hitTarget(310, 90, ts, 70), 1);
  assertEq(hitTarget(100, 100, ts, 70), 0);
  assertEq(hitTarget(640, 640, ts, 70), -1);
});
test('hitTarget 跳過已完成目標', () => {
  const ts = [{x:100,y:100,done:true},{x:112,y:100,done:false}];
  assertEq(hitTarget(100, 100, ts, 70), 1);
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 兩則新測試 `FAIL`（`hitTarget is not defined`），其餘 14 則 PASS。

- [ ] **Step 3: 實作 hitTarget（純函式，加在 `/* 音效測試 */` 之前）**

```js
/* ===== 解碼配對模型 ===== */
function hitTarget(x, y, targets, r){
  for (let i = 0; i < targets.length; i++){
    const t = targets[i];
    if (!t.done && Math.hypot(x - t.x, y - t.y) <= r) return i;
  }
  return -1;
}
```

- [ ] **Step 4: 驗證通過**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (16/16)`。

- [ ] **Step 5: 解碼場景（HTML／CSS）**

HTML——填入空的 act2-decode section（字槽中心座標＝`left`＋85、`top`＋85，須與 Step 6 的 `decode.targets` 一致）：

```html
  <section data-scene="act2-decode" id="scene-act2-decode">
    <p id="decode-title">貞人翻出「上次問雨」的甲骨紀錄——<strong>把圖拖到對的字上</strong>，解開它。</p>
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <!-- 甲骨板 -->
      <path d="M340 130 C 260 170 240 300 270 430 C 295 540 380 600 640 600
               C 900 600 985 540 1010 430 C 1040 300 1020 170 940 130
               C 840 90 440 90 340 130 Z"
            fill="var(--paper)" stroke="#c9b992" stroke-width="4"/>
    </svg>
    <!-- 三個字槽：甲骨文 ⇄ 楷書 -->
    <div class="decode-slot" id="slot-rain" style="left:390px;top:230px;">
      <svg viewBox="0 0 200 200" width="170" height="170">
        <!-- 甲骨文「雨」：一横天，點點往下落 -->
        <g fill="none" stroke="var(--ink)" stroke-width="10" stroke-linecap="round">
          <path d="M45 55 H155"/>
          <path d="M65 85 v30 M100 92 v30 M135 85 v30"/>
        </g>
      </svg>
      <div class="kai">雨</div>
    </div>
    <div class="decode-slot" id="slot-grain" style="left:555px;top:230px;">
      <svg viewBox="0 0 200 200" width="170" height="170">
        <!-- 甲骨文「禾」：垂穗的莖與葉 -->
        <g fill="none" stroke="var(--ink)" stroke-width="10" stroke-linecap="round">
          <path d="M100 55 V160"/>
          <path d="M100 55 Q80 30 58 42"/>
          <path d="M100 95 L68 72 M100 95 L132 72"/>
          <path d="M100 130 L70 112 M100 130 L130 112"/>
          <path d="M100 160 L78 178 M100 160 L122 178"/>
        </g>
      </svg>
      <div class="kai">禾</div>
    </div>
    <div class="decode-slot" id="slot-sun" style="left:720px;top:230px;">
      <svg viewBox="0 0 200 200" width="170" height="170">
        <!-- 甲骨文「日」：圓形帶中點 -->
        <g fill="none" stroke="var(--ink)" stroke-width="10" stroke-linecap="round">
          <circle cx="100" cy="105" r="52"/>
          <path d="M100 95 v20"/>
        </g>
      </svg>
      <div class="kai">日</div>
    </div>
    <!-- 三張可拖的圖（左右順序刻意與字槽不同） -->
    <div class="decode-tile" data-key="grain" style="left:300px;">
      <svg viewBox="0 0 100 100" width="90" height="90">
        <g fill="none" stroke="var(--bronze)" stroke-width="6" stroke-linecap="round">
          <path d="M50 88 V45"/>
          <path d="M50 60 Q28 52 22 32"/>
          <path d="M50 60 Q72 52 78 32"/>
          <path d="M50 45 Q42 24 50 12"/>
        </g>
      </svg>
      <span>禾苗</span>
    </div>
    <div class="decode-tile" data-key="sun" style="left:580px;">
      <svg viewBox="0 0 100 100" width="90" height="90">
        <g stroke="var(--gold)" stroke-width="5" stroke-linecap="round">
          <circle cx="50" cy="50" r="22" fill="var(--gold)"/>
          <path d="M50 12 v12 M50 76 v12 M12 50 h12 M76 50 h12 M23 23 l9 9 M68 68 l9 9 M77 23 l-9 9 M32 68 l-9 9" fill="none"/>
        </g>
      </svg>
      <span>太陽</span>
    </div>
    <div class="decode-tile" data-key="rain" style="left:860px;">
      <svg viewBox="0 0 100 100" width="90" height="90">
        <g stroke="#7d9bb5" stroke-width="6" stroke-linecap="round">
          <path d="M25 40 Q25 22 45 22 Q50 10 65 14 Q82 16 80 34 Q90 38 86 48 Q80 56 68 54 L30 54 Q18 52 25 40 Z" fill="#7d9bb5"/>
          <path d="M35 66 l-6 14 M55 66 l-6 14 M75 66 l-6 14" fill="none"/>
        </g>
      </svg>
      <span>雨滴</span>
    </div>
    <p id="decode-done">
      你剛剛讀懂了 3000 年前的紀錄——<br>
      <strong>那一年：下了雨，禾苗曬著太陽長大。好年。</strong>
    </p>
    <button id="decode-next-btn" class="next-btn">下一幕 →</button>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act2 decode --- */
#decode-title{position:absolute;top:30px;width:100%;text-align:center;font-size:28px;
  letter-spacing:.08em;margin:0;color:var(--paper);}
#decode-title strong{color:var(--gold);}
.decode-slot{position:absolute;width:170px;height:170px;}
.decode-slot svg{position:absolute;inset:0;transition:opacity .9s;}
.decode-slot .kai{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;
  font-size:120px;color:var(--ink);opacity:0;transition:opacity .9s;}
.decode-slot.matched svg{opacity:0;}
.decode-slot.matched .kai{opacity:1;}
.decode-tile{position:absolute;top:600px;width:110px;text-align:center;cursor:grab;
  touch-action:none;user-select:none;}
.decode-tile span{display:block;font-size:20px;color:var(--gold);letter-spacing:.1em;}
.decode-tile.back{transition:transform .35s cubic-bezier(.3,1.4,.5,1);}
.decode-tile.gone{opacity:0;pointer-events:none;transition:opacity .4s;}
#decode-done{position:absolute;left:50%;top:435px;transform:translateX(-50%);width:900px;
  text-align:center;font-size:26px;line-height:1.7;margin:0;color:var(--ink);
  opacity:0;transition:opacity .8s;}
#decode-done.show{opacity:1;}
#decode-done strong{color:var(--cinnabar);}
#decode-next-btn{position:absolute;left:50%;bottom:36px;transform:translateX(-50%);
  opacity:0;transition:opacity .5s;}
#decode-next-btn.show{opacity:1;}
```

- [ ] **Step 6: 拖拉互動 JS（加在 Task 4 的第二幕程式之後、`/* 音效測試 */` 之前）**

```js
/* ===== 第二幕：拖拉解碼 ===== */
const decode = {
  targets: [
    { key:'rain',  x:475, y:315, done:false, el:'slot-rain'  },
    { key:'grain', x:640, y:315, done:false, el:'slot-grain' },
    { key:'sun',   x:805, y:315, done:false, el:'slot-sun'   }
  ],
  HIT_R: 95
};
function stagePoint(clientX, clientY){
  const r = document.getElementById('stage').getBoundingClientRect();
  return { x: (clientX - r.left) / (r.width / 1280),
           y: (clientY - r.top) / (r.height / 720) };
}
(() => {
  document.querySelectorAll('.decode-tile').forEach(tile => {
    let sx = 0, sy = 0, dragging = false;
    function snapBack(){
      tile.classList.add('back');
      tile.style.transform = 'translate(0,0)';
    }
    tile.addEventListener('pointerdown', e => {
      e.preventDefault();
      dragging = true;
      tile.setPointerCapture(e.pointerId);
      tile.classList.remove('back');
      const p = stagePoint(e.clientX, e.clientY);
      sx = p.x; sy = p.y;
    });
    tile.addEventListener('pointermove', e => {
      if (!dragging) return;
      const p = stagePoint(e.clientX, e.clientY);
      tile.style.transform = `translate(${p.x - sx}px,${p.y - sy}px)`;
    });
    tile.addEventListener('pointerup', e => {
      if (!dragging) return;
      dragging = false;
      const p = stagePoint(e.clientX, e.clientY);
      const i = hitTarget(p.x, p.y, decode.targets, decode.HIT_R);
      if (i !== -1 && decode.targets[i].key === tile.dataset.key){
        decode.targets[i].done = true;
        tile.classList.add('gone');
        document.getElementById(decode.targets[i].el).classList.add('matched');
        audio.ding();
        if (decode.targets.every(t => t.done)){
          setTimeout(() => {
            document.getElementById('decode-done').classList.add('show');
            document.getElementById('decode-next-btn').classList.add('show');
            audio.ding();
          }, 1000);
        }
      } else {
        if (i !== -1) audio.thump();   /* 拖到別的字：悶一聲提示 */
        snapBack();
      }
    });
    tile.addEventListener('pointercancel', () => {
      if (!dragging) return;
      dragging = false;
      snapBack();
    });
  });
  document.addEventListener('scene:enter', e => {
    if (e.detail.scene !== 'act2-decode') return;
    decode.targets.forEach(t => {
      t.done = false;
      document.getElementById(t.el).classList.remove('matched');
    });
    document.querySelectorAll('.decode-tile').forEach(t => {
      t.classList.remove('gone','back');
      t.style.transform = 'translate(0,0)';
    });
    document.getElementById('decode-done').classList.remove('show');
    document.getElementById('decode-next-btn').classList.remove('show');
  });
  document.getElementById('decode-next-btn').addEventListener('click', () => game.goto('act3-intro'));
})();
```

- [ ] **Step 7: 自測回歸＋視覺／手感檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (16/16)`。
Run: `start .\index.html` → 圓點跳第二幕 → 問天 → 燒 → 解碼 → Expected:
1. 甲骨板上三個甲骨文（雨＝一橫加落點、禾＝垂穗、日＝圓帶點），下方三張圖（順序打亂）；
2. 拖「雨滴」到「雨」：ding、圖消失、甲骨文交叉淡化成楷書「雨」（字形演化）；
3. 拖錯（如「太陽」放到「禾」上）：悶響、圖片彈回原位；放到空白處：無聲彈回；
4. 三個都對：一秒後浮出解讀文案＋「下一幕 →」，再 ding 一聲；
5. 縮放視窗到約 800×500 再拖一次——拖曳跟手（舞台座標換算正確）；
6. 圓點跳走再跳回第二幕重玩：圖與字槽全部復位。

- [ ] **Step 8: Commit**

```powershell
git add index.html; git commit -m "feat: act2 drag-to-decode oracle glyphs with kai morph"
```

---

### Task 6: 第三幕開場（鼓聲警報）與三選一派將

**Files:**
- Modify: `index.html`（填 `#scene-act3-intro`、`#scene-act3-choose`；`audio` 物件加 `drumRoll`）

**Interfaces:**
- Consumes: `setBurn`、`game.goto`、`scene:enter`、`.dialog`／`.next-btn`、`audio`
- Produces:
  - `audio.drumRoll()`——五連鼓（末拍最重）；未解鎖時靜默返回
  - act3-intro：紅光警報閃爍＋城牆狼煙＋`#act3-ask-btn` → `setBurn('act3-choose', 1.6, '按住龜甲——問這一仗')` → burn
  - act3-choose：三名將領剪影三選一，任選皆揭曉婦好（`#fuhao-reveal`）；`#act3-verify-btn`「出征！ →」→ `game.goto('act3-verify')`

- [ ] **Step 1: 先寫 drumRoll 失敗自測（檔案最末測試區）**

```js
test('audio.drumRoll 存在且可呼叫', () => {
  assertEq(typeof audio.drumRoll, 'function');
  audio.drumRoll();   /* 已解鎖，不應拋錯 */
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 新測試 `FAIL`（`expected function, got undefined`），其餘 16 則 PASS。

- [ ] **Step 3: audio 物件加 drumRoll**

在 `audio` 物件的 `thump(){…}` 方法之後加逗號，接著新增：

```js
  drumRoll(){
    if (!this.ctx || !this.enabled) return;
    const t0 = this.ctx.currentTime;
    [0, 0.22, 0.44, 0.66, 0.9].forEach((dt, i) => {
      const t = t0 + dt;
      const o = this.ctx.createOscillator();
      o.frequency.setValueAtTime(110, t);
      o.frequency.exponentialRampToValueAtTime(45, t + 0.18);
      const g = this.ctx.createGain();
      g.gain.setValueAtTime(i === 4 ? 0.9 : 0.55, t);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.22);
      o.connect(g); g.connect(this.ctx.destination); o.start(t); o.stop(t + 0.25);
    });
  }
```

- [ ] **Step 4: 驗證通過**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (17/17)`。

- [ ] **Step 5: act3-intro 場景（HTML／CSS／JS）**

HTML——填入空的 act3-intro section：

```html
  <section data-scene="act3-intro" id="scene-act3-intro">
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <rect width="1280" height="720" fill="#1c1210"/>
      <!-- 狼煙 -->
      <path d="M1000 520 Q980 440 1010 380 Q1040 320 1010 250 Q990 200 1015 150"
            fill="none" stroke="#5a5248" stroke-width="26" stroke-linecap="round" opacity=".8"/>
      <!-- 城牆與垛口 -->
      <rect y="560" width="1280" height="160" fill="#0d0b09"/>
      <g fill="#0d0b09">
        <rect x="60" y="520" width="70" height="40"/>
        <rect x="260" y="520" width="70" height="40"/>
        <rect x="460" y="520" width="70" height="40"/>
        <rect x="660" y="520" width="70" height="40"/>
        <rect x="860" y="520" width="70" height="40"/>
        <rect x="1060" y="520" width="70" height="40"/>
      </g>
    </svg>
    <div id="alarm-flash"></div>
    <div class="dialog">
      <p class="speaker">第三幕・羌方入侵</p>
      <p>急報——<strong>羌方打過來了！</strong>邊境的狼煙升起來了。</p>
      <p>這一次不是牙痛，是<strong>戰爭</strong>。商王照樣——先問天。</p>
      <button id="act3-ask-btn" class="next-btn">立刻問天 →</button>
    </div>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act3 intro --- */
#alarm-flash{position:absolute;inset:0;background:var(--cinnabar);opacity:0;pointer-events:none;}
#scene-act3-intro.active #alarm-flash{animation:alarm 1.2s 2;}
@keyframes alarm{0%,100%{opacity:0}15%{opacity:.35}30%{opacity:0}45%{opacity:.25}60%{opacity:0}}
```

JS——加在第二幕程式之後、`/* 音效測試 */` 之前：

```js
/* ===== 第三幕 ===== */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene === 'act3-intro') audio.drumRoll();
});
document.getElementById('act3-ask-btn').addEventListener('click', () => {
  setBurn('act3-choose', 1.6, '按住龜甲——問這一仗');
  game.goto('burn');
});
```

- [ ] **Step 6: act3-choose 場景（HTML／CSS／JS）**

HTML——填入空的 act3-choose section（中間那位＝婦好：髮髻＋硃砂紅戈）：

```html
  <section data-scene="act3-choose" id="scene-act3-choose">
    <p class="choose-title">裂紋說：可以打。那——<strong>派誰帶兵？</strong></p>
    <div id="generals">
      <button class="general" data-idx="0">
        <svg viewBox="0 0 200 300" width="180" height="270" aria-hidden="true">
          <g fill="#0d0b09">
            <circle cx="100" cy="60" r="30"/>
            <path d="M60 100 h80 l14 130 h-108 Z"/>
          </g>
          <line x1="160" y1="40" x2="160" y2="230" stroke="var(--bronze)" stroke-width="8"/>
          <path d="M160 40 l22 18 -22 8 Z" fill="var(--bronze)"/>
        </svg>
        <span>將軍・望乘</span>
      </button>
      <button class="general" data-idx="1">
        <svg viewBox="0 0 200 300" width="180" height="270" aria-hidden="true">
          <g fill="#0d0b09">
            <circle cx="100" cy="62" r="28"/>
            <path d="M84 30 q16 -22 34 -2 q12 14 -6 20 Z"/>
            <path d="M64 100 h72 l12 130 h-96 Z"/>
          </g>
          <line x1="152" y1="50" x2="152" y2="230" stroke="var(--cinnabar)" stroke-width="8"/>
          <path d="M132 62 h40 v14 h-40 Z" fill="var(--cinnabar)"/>
        </svg>
        <span>？？？</span>
      </button>
      <button class="general" data-idx="2">
        <svg viewBox="0 0 200 300" width="180" height="270" aria-hidden="true">
          <g fill="#0d0b09">
            <circle cx="100" cy="60" r="32"/>
            <path d="M52 100 h96 l16 130 h-128 Z"/>
          </g>
          <line x1="44" y1="60" x2="44" y2="230" stroke="var(--bronze)" stroke-width="8"/>
          <path d="M44 60 q-30 10 -26 42 q20 -6 26 -14 Z" fill="var(--bronze)"/>
        </svg>
        <span>將軍・沚戛</span>
      </button>
    </div>
    <div id="fuhao-reveal">
      <p id="fuhao-line1"></p>
      <p id="fuhao-name">婦好</p>
      <p id="fuhao-sub">商王的王后——也是帶兵打仗的<strong>女將軍</strong>。<br>
        這不是編的：甲骨上真的刻著，她帶了<strong>一萬三千人</strong>出征。</p>
      <button id="act3-verify-btn" class="next-btn">出征！ →</button>
    </div>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act3 choose --- */
.choose-title{position:absolute;top:44px;width:100%;text-align:center;font-size:32px;
  letter-spacing:.08em;margin:0;}
.choose-title strong{color:var(--cinnabar);}
#generals{position:absolute;top:140px;left:50%;transform:translateX(-50%);display:flex;gap:70px;}
.general{background:transparent;border:2px solid transparent;border-radius:10px;
  cursor:pointer;padding:12px;transition:opacity .6s,border-color .3s;}
.general:hover{border-color:var(--gold);}
.general span{display:block;margin-top:8px;font-family:var(--serif);font-size:22px;
  color:var(--gold);letter-spacing:.15em;}
#generals.picked .general{pointer-events:none;opacity:.12;}
#generals.picked .general.chosen{opacity:1;}
#fuhao-reveal{position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);
  width:760px;background:var(--paper);color:var(--ink);border-radius:10px;
  padding:30px 40px;text-align:center;box-shadow:0 10px 40px rgba(0,0,0,.7);
  opacity:0;pointer-events:none;transition:opacity .7s;}
#fuhao-reveal.show{opacity:1;pointer-events:auto;}
#fuhao-line1{font-size:24px;margin:0 0 6px;}
#fuhao-name{font-size:88px;color:var(--cinnabar);letter-spacing:.2em;margin:6px 0;font-weight:bold;}
#fuhao-sub{font-size:24px;line-height:1.7;margin:8px 0 16px;}
#fuhao-sub strong{color:var(--cinnabar);}
```

JS——接在第三幕程式之後：

```js
const FU_HAO = 1;
document.querySelectorAll('.general').forEach(btn => {
  btn.addEventListener('click', () => {
    const idx = Number(btn.dataset.idx);
    document.getElementById('generals').classList.add('picked');
    btn.classList.add('chosen');
    document.getElementById('fuhao-line1').textContent =
      idx === FU_HAO ? '你選中的這位——' : '商王想了想，卻點了中間那位——';
    setTimeout(() => {
      document.getElementById('fuhao-reveal').classList.add('show');
      audio.ding();
    }, 900);
  });
});
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'act3-choose') return;
  document.getElementById('generals').classList.remove('picked');
  document.querySelectorAll('.general').forEach(b => b.classList.remove('chosen'));
  document.getElementById('fuhao-reveal').classList.remove('show');
});
document.getElementById('act3-verify-btn').addEventListener('click', () => game.goto('act3-verify'));
```

- [ ] **Step 7: 自測回歸＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (17/17)`。
Run: `start .\index.html` → 第二幕結尾「下一幕 →」（或圓點跳第三幕）→ Expected:
1. 進第三幕：鼓聲五連擊、紅光閃兩輪、城牆狼煙剪影；
2. 「立刻問天」→ 燒（1.6 秒、提示「問這一仗」）→ 裂 → 進三選一；
3. 三剪影：長矛、髮髻持紅戈（標「？？？」）、大斧；hover 有金框；
4. 點「？？？」：其餘變暗，浮出白卡「你選中的這位——婦好」＋ding；
5. 重玩點「望乘」：文案變「商王想了想，卻點了中間那位——」，同樣揭曉婦好（無懲罰）；
6. 「出征！ →」→ act3-verify（空黑，正常）。

- [ ] **Step 8: Commit**

```powershell
git add index.html; git commit -m "feat: act3 alarm intro and pick-a-general reveal of Fu Hao"
```

---

### Task 7: 驗辭場景——同一片甲骨補刻結果

**Files:**
- Modify: `index.html`（填 `#scene-act3-verify`；`audio` 物件加 `carve`）

**Interfaces:**
- Consumes: `scene:enter`、`audio.ding`、`.next-btn`、`game.goto`
- Produces:
  - `audio.carve()`——短促高頻刻痕聲（帶通噪音 60ms）；未解鎖時靜默返回。**第四幕磨粉聲也復用此函式**
  - act3-verify：甲骨上先示出征前卜辭，再逐字「刻」出驗辭「果然——勝。」（每字一記 carve），點題後 `#act4-btn`「下一幕 →」→ `game.goto('act4-intro')`

- [ ] **Step 1: audio 物件加 carve**

在 Task 6 加入的 `drumRoll(){…}` 之後加逗號，接著新增：

```js
  carve(){
    if (!this.ctx || !this.enabled) return;
    const t = this.ctx.currentTime;
    const src = this.ctx.createBufferSource(); src.buffer = this._noise(0.05);
    const bp = this.ctx.createBiquadFilter();
    bp.type = 'bandpass'; bp.frequency.value = 2600; bp.Q.value = 2;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(0.5, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.06);
    src.connect(bp); bp.connect(g); g.connect(this.ctx.destination); src.start(t);
  }
```

- [ ] **Step 2: 場景 HTML／CSS**

HTML——填入空的 act3-verify section：

```html
  <section data-scene="act3-verify" id="scene-act3-verify">
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <path d="M340 110 C 260 150 240 290 270 430 C 295 545 380 610 640 610
               C 900 610 985 545 1010 430 C 1040 290 1020 150 940 110
               C 840 70 440 70 340 110 Z"
            fill="var(--paper)" stroke="#c9b992" stroke-width="4"/>
    </svg>
    <div id="verify-card">
      <p class="v-label">出征那天，甲骨上刻著——</p>
      <p class="v-q">「婦好帶一萬三千人伐羌，會贏嗎？——<strong>吉。</strong>」</p>
      <p class="v-label" id="v-label2">幾個月後，貞人回到<strong>同一片甲骨</strong>，補刻了一行小字：</p>
      <p id="verify-answer"></p>
      <p id="verify-punch">商朝人不只問天——還回頭記下<strong>到底準不準</strong>。<br>
        這行補刻叫「驗辭」。歷史，就是這樣被記下來的。</p>
      <button id="act4-btn" class="next-btn">下一幕 →</button>
    </div>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act3 verify --- */
#verify-card{position:absolute;left:50%;top:130px;transform:translateX(-50%);width:640px;
  color:var(--ink);text-align:center;}
#verify-card .v-label{font-size:22px;color:#6a5c44;margin:14px 0 4px;opacity:0;transition:opacity .8s;}
#verify-card .v-q{font-size:32px;line-height:1.6;margin:4px 0;opacity:0;transition:opacity .8s;}
#verify-card strong{color:var(--cinnabar);}
#verify-answer{font-size:46px;letter-spacing:.15em;color:var(--cinnabar);
  min-height:66px;margin:8px 0;font-weight:bold;}
#verify-punch{font-size:24px;line-height:1.8;margin:14px 0;opacity:0;transition:opacity .8s;}
#verify-card #act4-btn{opacity:0;transition:opacity .5s;}
#verify-card .shown{opacity:1 !important;}
```

- [ ] **Step 3: 時序 JS（接在第三幕程式之後、`/* 音效測試 */` 之前）**

```js
/* ===== 驗辭 ===== */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'act3-verify') return;
  const card = document.getElementById('verify-card');
  card.querySelectorAll('.v-label, .v-q, #verify-punch, #act4-btn')
    .forEach(el => el.classList.remove('shown'));
  const ans = document.getElementById('verify-answer');
  ans.textContent = '';
  const labels = card.querySelectorAll('.v-label');
  setTimeout(() => labels[0].classList.add('shown'), 400);
  setTimeout(() => card.querySelector('.v-q').classList.add('shown'), 1000);
  setTimeout(() => labels[1].classList.add('shown'), 2800);
  const text = '「果然——勝。」';
  [...text].forEach((ch, i) => setTimeout(() => {
    ans.textContent += ch;
    if (ch !== '「' && ch !== '」') audio.carve();
  }, 4400 + i * 260));
  const after = 4400 + text.length * 260;
  setTimeout(() => document.getElementById('verify-punch').classList.add('shown'), after + 700);
  setTimeout(() => {
    document.getElementById('act4-btn').classList.add('shown');
    audio.ding();
  }, after + 1500);
});
document.getElementById('act4-btn').addEventListener('click', () => game.goto('act4-intro'));
```

- [ ] **Step 4: 自測回歸＋視覺檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (17/17)`。
Run: `start .\index.html` → 走到驗辭（可用圓點跳第三幕加速）→ Expected:
1. 甲骨板上先淡入「出征那天…吉。」；
2. 停一拍後「幾個月後…補刻了一行小字：」淡入；
3. 「果然——勝。」逐字浮現，每個字伴一記短促刻痕聲，硃砂紅大字；
4. 點題文案（驗辭）淡入 → ding → 「下一幕 →」浮現 → 點擊進 act4-intro（空黑，正常）；
5. 跳走再跳回：時序從頭重播，不殘留上次文字。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "feat: act3 verification inscription carved onto the same bone"
```

---

### Task 8: 第四幕——1899 藥鋪與磨龍骨互動

**Files:**
- Modify: `index.html`（填 `#scene-act4-intro`、`#scene-act4-grind`；純函式 `grindStep`）

**Interfaces:**
- Consumes: `game.goto`、`scene:enter`、`audio.carve`／`audio.pop`／`audio.ding`、`.dialog`／`.next-btn`、`#stage`（shake）
- Produces:
  - `GRIND = { needed: 9000 }` 與純函式 `grindStep(progress, dist) -> number`——累積磨擦距離換算 0..1 進度，夾在 [0,1]，負距離不動
  - act4-intro：「3000 年後」時空跳轉卡 → 藥鋪場景 → `#grind-btn` → `game.goto('act4-grind')`
  - act4-grind：按住來回磨的刮開互動；進度 ≥ 0.55 觸發「住手——！！」＋學者對白；`#finale-btn`「後來呢？ →」→ `game.goto('act4-finale')`

- [ ] **Step 1: 先寫 grindStep 失敗自測（檔案最末測試區）**

```js
test('grindStep 累積磨擦距離成進度', () => {
  let p = grindStep(0, GRIND.needed / 2);
  assertEq(p, 0.5);
  p = grindStep(p, GRIND.needed / 2);
  assertEq(p, 1);
});
test('grindStep 上限夾在 1', () => {
  assertEq(grindStep(0.9, GRIND.needed), 1);
});
test('grindStep 負距離不動', () => {
  assertEq(grindStep(0.3, -50), 0.3);
});
```

- [ ] **Step 2: 驗證失敗**

Run: 重新整理 `#selftest` → Expected: 三則新測試 `FAIL`（`grindStep is not defined`），其餘 17 則 PASS。

- [ ] **Step 3: 實作 grindStep（純函式，加在 `/* 音效測試 */` 之前）**

```js
/* ===== 磨龍骨進度模型 ===== */
const GRIND = { needed: 9000 };   /* 需累積的磨擦距離（舞台像素） */
function grindStep(progress, dist){
  return Math.min(1, progress + Math.max(0, dist) / GRIND.needed);
}
```

- [ ] **Step 4: 驗證通過**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (20/20)`。

- [ ] **Step 5: act4-intro 場景（HTML／CSS／JS）**

HTML——填入空的 act4-intro section：

```html
  <section data-scene="act4-intro" id="scene-act4-intro">
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <rect width="1280" height="720" fill="#20180f"/>
      <!-- 藥櫃 -->
      <rect x="120" y="140" width="440" height="400" fill="#120d08"/>
      <g stroke="#3a2d1c" stroke-width="4">
        <path d="M120 240 h440 M120 340 h440 M120 440 h440"/>
        <path d="M230 140 v400 M340 140 v400 M450 140 v400"/>
      </g>
      <!-- 掛牌「藥」 -->
      <line x1="900" y1="60" x2="900" y2="130" stroke="#3a2d1c" stroke-width="6"/>
      <rect x="850" y="130" width="100" height="150" fill="var(--paper)" rx="4"/>
      <text x="900" y="235" text-anchor="middle" font-size="72" fill="var(--ink)"
            font-family="serif">藥</text>
      <!-- 櫃檯、石臼與龍骨 -->
      <rect y="540" width="1280" height="180" fill="#0d0b09"/>
      <ellipse cx="760" cy="540" rx="130" ry="34" fill="#241e18"/>
      <path d="M700 520 q60 -40 120 -6 q-20 22 -60 22 q-40 0 -60 -16 Z" fill="var(--paper)"/>
      <!-- 掌櫃剪影 -->
      <g fill="#0d0b09">
        <circle cx="380" cy="420" r="34"/>
        <path d="M330 460 h100 l16 110 h-132 Z"/>
        <rect x="352" y="382" width="56" height="12" rx="5"/>
      </g>
    </svg>
    <div class="dialog">
      <p class="speaker">藥鋪掌櫃</p>
      <p>「客官，這味藥叫<strong>龍骨</strong>——老骨頭磨成粉，什麼病都治！」</p>
      <p>「來，幫我把這塊<strong>磨成粉</strong>。」</p>
      <button id="grind-btn" class="next-btn">開始磨藥 →</button>
    </div>
    <p id="timejump">3000 年後——<br><span>西元 1899 年・北京・一間中藥鋪</span></p>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act4 intro --- */
#timejump{position:absolute;inset:0;background:var(--ink);z-index:5;display:flex;
  flex-direction:column;align-items:center;justify-content:center;
  font-size:56px;letter-spacing:.2em;text-align:center;margin:0;line-height:1.9;
  transition:opacity 1s;}
#timejump span{font-size:30px;color:var(--gold);letter-spacing:.25em;}
#timejump.hide{opacity:0;pointer-events:none;}
```

JS——加在驗辭程式之後、`/* 音效測試 */` 之前：

```js
/* ===== 第四幕 ===== */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'act4-intro') return;
  const tj = document.getElementById('timejump');
  tj.classList.remove('hide');
  setTimeout(() => tj.classList.add('hide'), 2600);
});
document.getElementById('grind-btn').addEventListener('click', () => game.goto('act4-grind'));
```

- [ ] **Step 6: act4-grind 場景（HTML／CSS）**

HTML——填入空的 act4-grind section（骨上的甲骨文＝第二幕解過的「雨」「禾」）：

```html
  <section data-scene="act4-grind" id="scene-act4-grind">
    <svg viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <!-- 石臼底 -->
      <ellipse cx="640" cy="400" rx="430" ry="235" fill="#241e18"/>
      <!-- 龍骨（下層，帶字） -->
      <path d="M300 300 C 320 240 420 220 640 220 C 860 220 960 240 980 300
               C 1000 360 980 480 900 510 C 800 545 480 545 380 510
               C 300 480 280 360 300 300 Z"
            fill="var(--paper)" stroke="#c9b992" stroke-width="4"/>
      <g fill="none" stroke="var(--ink)" stroke-width="9" stroke-linecap="round">
        <!-- 甲骨文「雨」 -->
        <path d="M500 300 H610"/>
        <path d="M520 330 v28 M555 336 v28 M590 330 v28"/>
        <!-- 甲骨文「禾」 -->
        <path d="M760 290 V400"/>
        <path d="M760 290 Q740 268 718 278"/>
        <path d="M760 330 L730 308 M760 330 L790 308"/>
        <path d="M760 365 L732 348 M760 365 L788 348"/>
      </g>
    </svg>
    <canvas id="powder" width="1280" height="720"></canvas>
    <p id="grind-hint">按住，來回磨——把龍骨磨成粉</p>
    <div id="stop-shout">住手——！！</div>
    <div class="dialog" id="scholar-dialog">
      <p class="speaker">買藥的學者・王懿榮</p>
      <p>「等等……這上面<strong>有字</strong>！這不是藥——這是老祖宗刻的字啊！」</p>
      <p class="note">——相傳，1899 年，學者王懿榮就是這樣發現了甲骨文。</p>
      <button id="finale-btn" class="next-btn">後來呢？ →</button>
    </div>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act4 grind --- */
#powder{position:absolute;inset:0;touch-action:none;cursor:grab;}
#grind-hint{position:absolute;left:50%;top:60px;transform:translateX(-50%);
  font-size:30px;letter-spacing:.2em;color:var(--gold);margin:0;transition:opacity .5s;}
#stop-shout{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;
  font-size:110px;font-weight:bold;color:var(--cinnabar);letter-spacing:.2em;
  background:rgba(13,11,9,.55);opacity:0;pointer-events:none;transition:opacity .15s;}
#stop-shout.show{opacity:1;}
#scholar-dialog{opacity:0;pointer-events:none;transition:opacity .6s;z-index:6;}
#scholar-dialog.show{opacity:1;pointer-events:auto;}
#scholar-dialog .note{font-size:19px;color:#6a5c44;}
```

- [ ] **Step 7: 磨擦互動 JS（接在第四幕程式之後、`/* 音效測試 */` 之前）**

```js
/* ===== 磨龍骨刮開互動 ===== */
(() => {
  const cv = document.getElementById('powder');
  const c = cv.getContext('2d');
  let holding = false, lastP = null, progress = 0, revealed = false, scrapeAt = 0;
  function paintPowder(){
    c.globalCompositeOperation = 'source-over';
    c.clearRect(0, 0, 1280, 720);
    c.fillStyle = '#cbb98f';
    c.beginPath(); c.ellipse(640, 385, 370, 180, 0, 0, 7); c.fill();
    for (let i = 0; i < 400; i++){   /* 粉末顆粒感 */
      const a = Math.random() * 7, rr = Math.sqrt(Math.random());
      c.fillStyle = Math.random() < .5 ? '#b9a67c' : '#ddcda6';
      c.fillRect(640 + Math.cos(a) * rr * 360, 385 + Math.sin(a) * rr * 172, 3, 3);
    }
  }
  function toCanvas(e){
    const r = cv.getBoundingClientRect();
    return { x: (e.clientX - r.left) * (1280 / r.width),
             y: (e.clientY - r.top) * (720 / r.height) };
  }
  cv.addEventListener('pointerdown', e => {
    if (revealed) return;
    e.preventDefault();
    holding = true; lastP = toCanvas(e);
    cv.setPointerCapture(e.pointerId);
  });
  cv.addEventListener('pointermove', e => {
    if (!holding || revealed) return;
    const p = toCanvas(e);
    const dist = Math.hypot(p.x - lastP.x, p.y - lastP.y);
    c.globalCompositeOperation = 'destination-out';
    c.lineWidth = 60; c.lineCap = 'round';
    c.beginPath(); c.moveTo(lastP.x, lastP.y); c.lineTo(p.x, p.y); c.stroke();
    progress = grindStep(progress, dist);
    lastP = p;
    if (performance.now() - scrapeAt > 130 && dist > 4){
      audio.carve();                       /* 磨粉沙沙聲，復用刻痕聲 */
      scrapeAt = performance.now();
    }
    if (progress >= 0.55) reveal();
  });
  function stopGrind(){ holding = false; lastP = null; }
  cv.addEventListener('pointerup', stopGrind);
  cv.addEventListener('pointercancel', stopGrind);
  function reveal(){
    revealed = true;
    audio.pop();
    document.getElementById('stage').classList.add('shake');
    setTimeout(() => document.getElementById('stage').classList.remove('shake'), 450);
    document.getElementById('stop-shout').classList.add('show');
    document.getElementById('grind-hint').style.opacity = 0;
    setTimeout(() => {
      document.getElementById('stop-shout').classList.remove('show');
      document.getElementById('scholar-dialog').classList.add('show');
      audio.ding();
    }, 1600);
  }
  document.addEventListener('scene:enter', e => {
    if (e.detail.scene !== 'act4-grind') return;
    progress = 0; revealed = false; holding = false; lastP = null;
    paintPowder();
    document.getElementById('stop-shout').classList.remove('show');
    document.getElementById('scholar-dialog').classList.remove('show');
    document.getElementById('grind-hint').style.opacity = 1;
  });
  document.getElementById('finale-btn').addEventListener('click', () => game.goto('act4-finale'));
})();
```

- [ ] **Step 8: 自測回歸＋視覺／手感檢查**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (20/20)`。
Run: `start .\index.html` → 圓點跳第四幕 → Expected:
1. 「3000 年後——1899 年北京中藥鋪」黑卡先出，2.6 秒後淡出露出藥鋪（藥櫃、掛牌、掌櫃、石臼）；
2. 「開始磨藥 →」→ 磨粉畫面：米黃粉層蓋著龍骨，帶顆粒感；
3. 按住來回磨：沙沙聲、粉被磨掉的軌跡下**露出甲骨文「雨」「禾」**（第二幕解過的字！）；
4. 磨到過半：「啪」＋畫面震動＋全屏「住手——！！」→ 學者對白（含「相傳」小字）；
5. 出對白後再磨無效（互動已凍結）；「後來呢？ →」→ act4-finale（空黑，正常）；
6. 縮放視窗到約 800×500 重磨一次：磨的位置跟手（canvas 座標換算正確）；
7. 跳走再跳回第四幕磨粉：粉層復原、可重新磨。

- [ ] **Step 9: Commit**

```powershell
git add index.html; git commit -m "feat: act4 pharmacy time-jump and grind-the-dragon-bone reveal"
```

---

### Task 9: 結尾點題卡與全程 20 分鐘 QA

**Files:**
- Modify: `index.html`（填 `#scene-act4-finale`）

**Interfaces:**
- Consumes: `scene:enter`、`audio.ding`、`game.goto`、`.next-btn`
- Produces: act4-finale——三行點題字卡逐行淡入 → 「商王的一天・完」→ 「再上一次 →」回 start。全劇終點。

- [ ] **Step 1: 場景（HTML／CSS／JS）**

HTML——填入空的 act4-finale section：

```html
  <section data-scene="act4-finale" id="scene-act4-finale">
    <p class="f-line">你剛剛讀的那些字，差點被磨成藥粉，吃掉。</p>
    <p class="f-line">商王刻下的問題——牙痛、下雨、打仗——</p>
    <p class="f-line">成了我們認識商朝<strong>唯一的證據</strong>。</p>
    <p id="finale-title">商王的一天・完</p>
    <button id="replay-btn" class="next-btn">再上一次 →</button>
  </section>
```

CSS——加在 `</style>` 之前：

```css
/* --- act4 finale --- */
#scene-act4-finale{flex-direction:column;align-items:center;justify-content:center;gap:18px;}
#scene-act4-finale.active{display:flex;}
.f-line{font-size:36px;letter-spacing:.12em;line-height:1.9;margin:0;text-align:center;
  opacity:0;transition:opacity 1s;}
.f-line strong{color:var(--cinnabar);}
.f-line.shown{opacity:1;}
#finale-title{font-size:60px;letter-spacing:.3em;margin:26px 0 0;opacity:0;transition:opacity 1.2s;}
#finale-title.shown{opacity:1;}
#replay-btn{opacity:0;transition:opacity .8s;}
#replay-btn.shown{opacity:1;}
```

JS——加在磨龍骨程式之後、`/* 音效測試 */` 之前：

```js
/* ===== 結尾點題 ===== */
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'act4-finale') return;
  const els = [...document.querySelectorAll('#scene-act4-finale .f-line'),
    document.getElementById('finale-title'),
    document.getElementById('replay-btn')];
  els.forEach(el => el.classList.remove('shown'));
  els.forEach((el, i) => setTimeout(() => {
    el.classList.add('shown');
    if (el.id === 'finale-title') audio.ding();
  }, 800 + i * 1600));
});
document.getElementById('replay-btn').addEventListener('click', () => game.goto('start'));
```

- [ ] **Step 2: 自測回歸**

Run: 重新整理 `#selftest` → Expected: `ALL PASS (20/20)`。

- [ ] **Step 3: 全程 QA（Phase 2 驗收清單，對照 spec §8 Phase 2）**

Run: `start .\index.html`，從頭完整走四幕並計時。Expected 全部成立：

1. **流程**：開始上課 → 第一幕（字卡→宮殿→燒→裂→吉→齒）→ 天亮了 → 第二幕（田野→儀式燒→拖拉解碼→字形演化）→ 第三幕（鼓聲警報→儀式燒→三選一→婦好→驗辭逐字刻）→ 第四幕（時空跳轉→藥鋪→磨龍骨→住手→相傳字卡→點題收尾）→ 完。全程無 Console 錯誤、無鍵盤需求；
2. **四種機制各自可玩**（spec §5）：按住燒（三處）、拖拉配對、三選一、磨擦刮開；
3. **節奏**：家教口述留白下全程約 15–20 分鐘；純操作快轉走完約 6–8 分鐘；
4. **章節跳轉**：左下四圓點任意跳幕，每幕跳入後互動皆從頭正常可玩（重點回歸：跳回第一幕燒龜甲裂紋已復原；跳回第二幕圖塊復位；跳回第四幕粉層復原）；
5. **背景音樂**：全程低音量循環、不壓過音效；「樂」鈕可整段關閉再開啟；
6. **收尾**：「再上一次 →」回到開始畫面，再走一輪第一幕正常；
7. **離線單檔**：關網路（或直接以檔案開啟）一切照常；`index.html` 為唯一交付物；
8. 視窗縮到約 800×500 抽查四幕互動各一次：等比縮放、拖拉與磨擦跟手；
9. 觸控檢查（如有觸控螢幕或 DevTools device mode）：燒、拖、選、磨四機制手指皆可操作。

如有任何一項不成立：修復後重跑本清單，再進 Step 4。

- [ ] **Step 4: Commit**

```powershell
git add index.html; git commit -m "feat: act4 finale card and full four-act 20-minute experience"
```

---

## 完成定義（Phase 2 驗收，對照 spec §8）

- 四種機制各自可玩：按住燒、拖拉配對、三選一、磨擦刮開。
- 家教可用左下章節圓點跳幕，不做存檔。
- 背景音樂五聲音階循環、可整段關閉。
- 全程離線單檔 `index.html`，雙擊即玩。
- 內嵌自測 `#selftest` 顯示 `ALL PASS (20/20)`。
