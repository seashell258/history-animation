# Act 3 Assemble-Bone Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 汰換第三幕主互動：把「三選一派將」換成「拼碎骨讀裂紋」——學生把 4 片碎骨拖回中央虛框，前 2 片看起來全是雜訊，第 4 片吸附完成的瞬間金色主脈浮現指向最左邊剪影，左邊剪影填色成婦好，浮出名字白卡。

**Architecture:** 沿用單一 `index.html` 檔的既有 `game.goto` / `scene:enter` / `audio` / selftest 骨架。既有 `act3-intro` 場景、`burn` 場景、`drumRoll` 音效、`act3-verify` 空場景**完全不動**。新增的邏輯（scatter、snap 命中、all-locked 狀態）拆成純函式便於 selftest 覆蓋；DOM／指標事件互動放在同一段 IIFE。

**Tech Stack:** 純 HTML + CSS + JS，PointerEvent（滑鼠與觸控統一），SVG 繪製剪影與碎骨。無外部依賴。

---

## Global Constraints

- **檔案：只碰 `index.html` 一個檔**（1000 多行的單頁 app；沿用既有 IIFE、audio、game、selftest 骨架）
- **不動：** `act3-intro` 場景、`audio.drumRoll`、`burn` 場景、`setBurn`、章節圓點導覽、`act3-verify` 空場景、Task 1–5 完成的第一二幕、`music` 引擎
- **必須維持：** `#selftest` 全數通過（現行 17 條；新增測試只增不減）；`stage 含 14 個場景 section` 這條測試需保持等式（rename 不改 section 數）
- **命名：** 場景 id 由 `act3-choose` 一次改到 `act3-assemble`。所有引用（`data-scene`、`section#scene-*`、`SCENE_ORDER` 陣列元素、`setBurn(..., 'act3-choose', ...)` 呼叫、selftest 內 `actOfScene('burn', 'act3-choose')` 字串）全部同步。章節圓點的 target 仍是 `act3-intro` 不受影響。
- **舞台座標：** stage 為 1280×720（既有 CSS）。本計畫使用的絕對座標都以此為基準。
- **座標與尺寸的權威來源：** 本計畫「Layout Constants」段落。所有 task 引用同一組數字，Task 3 修過就同步全案（暫時就固定用本文列的值）。
- **git author 已設定為 Kuanwei / no gpg**（Phase 2 前面 5 個 task 都跑過，工具鏈 OK）
- **驗證：** 只有 `#selftest` 加瀏覽器手動走查；沒有 Playwright／單元跑器等第三方測試框架

## Layout Constants（Task 3 起所有 SVG／CSS 都引用這組值）

```
Stage:                   1280 × 720
Silhouette size:         140 × 240
Silhouette y (top):      100
Silhouette centers x:    left=270, center=640, right=1010
Bone tile size:          220 × 220
Bone tile center:        (640, 540)
Bone tile bbox:          (530, 430) → (750, 650)
Main-spine polyline in tile-local coords (tile origin = tile center):
  P0 = (+55, +55)   startpoint 靠右下
  P1 = (+15, +10)
  P2 = (-25, -20)
  P3 = (-58, -55)
  P4 = (-100, -102) exitpoint 靠左上
Spine 出格延伸線（從 tile top-left 邊到左剪影腳下）：
  from (540, 438)  to  (280, 320)
```

碎片切分方式（4 片以「中軸不規則折線」切成 4 個象限）：
- Fragment A：top-left 象限，含 P4 與 P3-P4 段
- Fragment B：top-right 象限，**無主脈**，只有雜訊
- Fragment C：bottom-left 象限，含 P2-P3 段
- Fragment D：bottom-right 象限，含 P0-P1、P1-P2 段

每片除了自帶主脈段外，還畫 3–4 條**雜訊細裂**（半透明淺灰、走向亂）。前兩片吸附時看起來全是雜訊；第 3 片開始隱約成形；第 4 片吸附後主脈亮金、雜訊淡去。

---

## File Structure

單檔 app。所有變更集中在 `index.html`：

- **HTML** section#scene-act3-choose → 改名為 `#scene-act3-assemble` 並全面重寫內容
- **CSS** `.choose-title / #generals / .general / #fuhao-*` 全數刪除；新增 `.assemble-*` 系列樣式
- **JS 純函式** 新增 `inSnap`、`allLocked`、`scatterPositions`（於既有工具函式群旁）
- **JS 互動 IIFE** 新增第三幕拼骨的 pointer 處理、進場 scatter、all-locked 揭曉
- **selftest** 新增 3 條純函式測試；更新既有 `actOfScene('burn','act3-choose')` → `'act3-assemble'`
- **SCENE_ORDER** 陣列將 `'act3-choose'` 改為 `'act3-assemble'`
- **`burn` 流程觸發點** `setBurn('act3-choose', 1.6, ...)` 改為 `setBurn('act3-assemble', ...)`

---

### Task 1: 場景 id 改名 + 舊 choose 內容清空

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: 既有 `data-scene` 骨架、`SCENE_ORDER`、`setBurn`、`actOfScene` selftest
- Produces: `#scene-act3-assemble`（空 section）、所有引用同步、17/17 selftest 仍通過（其中 `actOfScene('burn','act3-assemble')` 這條為新字串）

- [ ] **Step 1: 更新 selftest 字串（先動測試才叫 TDD）**

找到既有 selftest：

```js
test('actOfScene 共用的 burn 場景看占卜去向', () => {
  assertEq(actOfScene('burn', 'verdict'), 1);
  assertEq(actOfScene('burn', 'act2-decode'), 2);
  assertEq(actOfScene('burn', 'act3-choose'), 3);
});
```

改為：

```js
test('actOfScene 共用的 burn 場景看占卜去向', () => {
  assertEq(actOfScene('burn', 'verdict'), 1);
  assertEq(actOfScene('burn', 'act2-decode'), 2);
  assertEq(actOfScene('burn', 'act3-assemble'), 3);
});
```

- [ ] **Step 2: 驗證測試現在會 FAIL**

`start .\index.html#selftest`
Expected: `FAIL`（因為 `actOfScene` 的判斷分支仍寫 `startsWith('act3')`；不會失敗？其實這條測試邏輯只看 prefix，改字串後仍會通過，因為 `'act3-assemble'.startsWith('act3-')` 為 true）
→ 若沒有紅：本步驟改成「驗證測試仍通過」——因為 `actOfScene` 是 prefix-based。**這是一個 rename-only 動作，本 task 沒有紅測可看**，直接進 Step 3。

- [ ] **Step 3: 改 section 標籤 id 與 data-scene**

找到：
```html
  <section data-scene="act3-choose" id="scene-act3-choose">
    ...現行三選一 HTML...
  </section>
```

改為：
```html
  <section data-scene="act3-assemble" id="scene-act3-assemble"></section>
```

（**整段 innerHTML 直接清空**——含三個 `.general` 按鈕、`#fuhao-reveal` 白卡等）

- [ ] **Step 4: 更新 SCENE_ORDER 陣列**

找到：
```js
'act3-intro','act3-choose','act3-verify',
```

改為：
```js
'act3-intro','act3-assemble','act3-verify',
```

- [ ] **Step 5: 更新 `act3-ask-btn` 的 setBurn target**

找到：
```js
document.getElementById('act3-ask-btn').addEventListener('click', () => {
  setBurn('act3-choose', 1.6, '按住龜甲——問這一仗');
  game.goto('burn');
});
```

改為：
```js
document.getElementById('act3-ask-btn').addEventListener('click', () => {
  setBurn('act3-assemble', 1.6, '按住龜甲——問這一仗');
  game.goto('burn');
});
```

- [ ] **Step 6: 刪除舊 choose 的 CSS**

找到並整段刪掉：
```css
/* --- act3 choose --- */
.choose-title{...}
.choose-title strong{...}
#generals{...}
.general{...}
.general:hover{...}
.general span{...}
#generals.picked .general{...}
#generals.picked .general.chosen{...}
#fuhao-reveal{...}
#fuhao-reveal.show{...}
#fuhao-line1{...}
#fuhao-name{...}
#fuhao-sub{...}
#fuhao-sub strong{...}
```

保留 `/* --- act3 intro --- */` 的 `#alarm-flash / @keyframes alarm`（不動 intro）。

- [ ] **Step 7: 刪除舊 choose 的 JS 段落**

找到並整段刪掉：
```js
const FU_HAO = 1;
document.querySelectorAll('.general').forEach(btn => { ... });
document.addEventListener('scene:enter', e => {
  if (e.detail.scene !== 'act3-choose') return;
  ...
});
document.getElementById('act3-verify-btn').addEventListener('click', () => game.goto('act3-verify'));
```

保留 intro 段的：
```js
document.addEventListener('scene:enter', e => {
  if (e.detail.scene === 'act3-intro') audio.drumRoll();
});
document.getElementById('act3-ask-btn').addEventListener('click', () => {
  setBurn('act3-assemble', 1.6, '按住龜甲——問這一仗');
  game.goto('burn');
});
```

- [ ] **Step 8: 驗證 17/17 通過**

Run: `start .\index.html#selftest`
Expected: `ALL PASS (17/17)`。
Run: `start .\index.html` → 快速確認可跑到第二幕結尾點「下一幕 →」跳到第三幕 intro（狼煙、鼓聲、警報 flash）；「立刻問天」→ burn（提示「問這一仗」）→ 燒完進 `act3-assemble` 空黑場景。若空場景後點章節圓點跳其他幕仍正常，即通過。

- [ ] **Step 9: Commit**

```powershell
git add index.html; git commit -m "refactor: rename act3-choose to act3-assemble and clear old pick-a-general content"
```

---

### Task 2: 純函式 `inSnap`——碎片是否落在吸附閾值內

**Files:**
- Modify: `index.html`（既有純函式群旁邊加 `inSnap`；selftest 加新條）

**Interfaces:**
- Consumes: nothing
- Produces: `inSnap(px, py, sx, sy, threshold) -> boolean`。碎片中心 (px,py) 與槽位中心 (sx,sy) 直線距離 ≤ threshold 為 true。

- [ ] **Step 1: 寫失敗測試（selftest 區塊末尾、最後一條 `hitTarget 跳過已完成目標` 之後）**

```js
test('inSnap 於閾值內回傳 true，於閾值外回傳 false', () => {
  assertEq(inSnap(100, 100, 100, 100, 60), true);        /* 完全重疊 */
  assertEq(inSnap(140, 100, 100, 100, 60), true);         /* 距離 40 < 60 */
  assertEq(inSnap(160, 100, 100, 100, 60), true);         /* 距離 60，含閾值 */
  assertEq(inSnap(161, 100, 100, 100, 60), false);        /* 距離 61 */
  assertEq(inSnap(100, 100, 130, 140, 60), true);         /* 距離 = 50 */
  assertEq(inSnap(100, 100, 140, 130, 40), false);        /* 距離 = 50，閾值 40 */
});
```

- [ ] **Step 2: 驗證 FAIL**

Run: `start .\index.html#selftest`
Expected: 新測試 `FAIL`（`inSnap is not defined`），其餘 17 條 PASS。

- [ ] **Step 3: 加 `inSnap` 純函式**

找到既有 `hitTarget` 函式（selftest 用得到的那條），在其後（或函式群任一位置，但要在 selftest 之前執行）加：

```js
function inSnap(px, py, sx, sy, threshold){
  const dx = px - sx, dy = py - sy;
  return dx*dx + dy*dy <= threshold*threshold;
}
```

- [ ] **Step 4: 驗證 PASS**

Run: `start .\index.html#selftest`
Expected: `ALL PASS (18/18)`。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "test+feat: inSnap snap-distance helper"
```

---

### Task 3: 純函式 `allLocked`——判斷 4 片是否都吸附完成

**Files:**
- Modify: `index.html`（純函式群 + selftest）

**Interfaces:**
- Consumes: nothing
- Produces: `allLocked(pieces) -> boolean`。`pieces` 是 `[{locked: bool}, ...]`；全部 `locked === true` 為 true，任何一片為 false（或陣列為空）都回傳 false。

- [ ] **Step 1: 寫失敗測試（接在 `inSnap` 測試之後）**

```js
test('allLocked 全為 true 才回 true', () => {
  assertEq(allLocked([]), false);
  assertEq(allLocked([{locked:true}]), true);
  assertEq(allLocked([{locked:true},{locked:true},{locked:true},{locked:true}]), true);
  assertEq(allLocked([{locked:true},{locked:false},{locked:true},{locked:true}]), false);
  assertEq(allLocked([{locked:false},{locked:false},{locked:false},{locked:false}]), false);
});
```

- [ ] **Step 2: 驗證 FAIL**

Run: `start .\index.html#selftest`
Expected: 新測試 `FAIL`（`allLocked is not defined`）。

- [ ] **Step 3: 加 `allLocked` 純函式**

```js
function allLocked(pieces){
  return pieces.length > 0 && pieces.every(p => p.locked);
}
```

- [ ] **Step 4: 驗證 PASS**

Run: `start .\index.html#selftest`
Expected: `ALL PASS (19/19)`。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "test+feat: allLocked assembly state helper"
```

---

### Task 4: 純函式 `scatterPositions`——把 4 片隨機灑在中央外的環狀範圍

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `scatterPositions(count, cx, cy, rInner, rOuter, rng) -> [{x, y}]`
  - 回傳長度 `count` 的陣列，每個座標 `(x,y)` 滿足 `rInner ≤ distance((x,y), (cx,cy)) ≤ rOuter`
  - `rng()` 是可注入的偽隨機來源（避免測試不穩）；預設用 `Math.random`
  - 各點之間**不強制不重疊**（灑的位置有些許重疊在畫面上還能接受，且要保證 100% 產出）

- [ ] **Step 1: 寫失敗測試**

```js
test('scatterPositions 每點都落在環狀內', () => {
  /* 可預測 rng */
  let i = 0;
  const seq = [0.1, 0.9, 0.3, 0.7, 0.5, 0.4, 0.8, 0.2, 0.6, 0.05, 0.95, 0.35];
  const rng = () => seq[i++ % seq.length];
  const pts = scatterPositions(4, 640, 540, 250, 320, rng);
  assertEq(pts.length, 4);
  pts.forEach((p, idx) => {
    const dx = p.x - 640, dy = p.y - 540;
    const d = Math.sqrt(dx*dx + dy*dy);
    assertOk(d >= 250 - 0.001 && d <= 320 + 0.001, `pt ${idx} d=${d} out of [250,320]`);
  });
});
```

- [ ] **Step 2: 驗證 FAIL**

Expected: `scatterPositions is not defined`。

- [ ] **Step 3: 加 `scatterPositions` 純函式**

```js
function scatterPositions(count, cx, cy, rInner, rOuter, rng){
  rng = rng || Math.random;
  const out = [];
  for (let i = 0; i < count; i++){
    const angle = rng() * Math.PI * 2;
    const r = rInner + rng() * (rOuter - rInner);
    out.push({ x: cx + Math.cos(angle) * r, y: cy + Math.sin(angle) * r });
  }
  return out;
}
```

- [ ] **Step 4: 驗證 PASS**

Expected: `ALL PASS (20/20)`。

- [ ] **Step 5: Commit**

```powershell
git add index.html; git commit -m "test+feat: scatterPositions helper for bone fragments"
```

---

### Task 5: 靜態場景骨架——三剪影（全黑）＋中央虛框＋隱藏白卡

**Files:**
- Modify: `index.html`（`#scene-act3-assemble` 內填入 HTML；`</style>` 之前加 CSS）

**Interfaces:**
- Consumes: Layout Constants 段的座標
- Produces:
  - `#scene-act3-assemble` 內含三剪影 SVG（全部無臉黑影）、中央虛框 SVG、`#fuhao-reveal` 白卡（`show` 未加時透明不可點）
  - CSS 樣式：`.assemble-title`、`.assemble-silhouette`（描邊、顯形時新增 `.revealed`）、`#bone-frame` 虛框樣式、`#fuhao-reveal` 白卡樣式（沿用命名）
  - 尚未有碎片與互動

- [ ] **Step 1: 填 HTML**

找到 `<section data-scene="act3-assemble" id="scene-act3-assemble"></section>`，改為：

```html
  <section data-scene="act3-assemble" id="scene-act3-assemble">
    <p class="assemble-title">裂紋說：可以打。<strong>誰去？</strong>王親自看甲骨。</p>
    <svg class="assemble-svg" viewBox="0 0 1280 720" width="1280" height="720" aria-hidden="true">
      <!-- 三剪影 -->
      <g class="assemble-silhouette" data-idx="0">
        <path d="M270 160 q-30 0 -30 40 t30 40 q30 0 30 -40 t-30 -40 M200 280 h140 l16 190 h-172 Z" fill="#0d0b09"/>
      </g>
      <g class="assemble-silhouette" data-idx="1">
        <path d="M640 160 q-30 0 -30 40 t30 40 q30 0 30 -40 t-30 -40 M570 280 h140 l16 190 h-172 Z" fill="#0d0b09"/>
      </g>
      <g class="assemble-silhouette" data-idx="2">
        <path d="M1010 160 q-30 0 -30 40 t30 40 q30 0 30 -40 t-30 -40 M940 280 h140 l16 190 h-172 Z" fill="#0d0b09"/>
      </g>
      <!-- 中央虛框：甲骨原位 -->
      <rect id="bone-frame" x="530" y="430" width="220" height="220" rx="20" ry="24"
            fill="none" stroke="#8b7a5e" stroke-width="2" stroke-dasharray="6 8" opacity=".6"/>
      <!-- 碎骨 group：Task 6 動態填 -->
      <g id="bone-pieces"></g>
      <!-- 主脈金線（拼完顯現，先不畫）與延伸線：Task 7 動態顯示 -->
      <g id="bone-spine" opacity="0"></g>
      <g id="bone-extend" opacity="0"></g>
      <!-- 婦好識別特徵：Task 7 顯示 -->
      <g id="fuhao-marks" opacity="0">
        <!-- 髮髻 -->
        <path d="M255 148 q15 -20 30 0 q-15 -8 -30 0 Z" fill="#5a4d3a"/>
        <!-- 硃砂紅戈（左剪影腰間斜挎） -->
        <line x1="240" y1="330" x2="310" y2="470" stroke="var(--cinnabar)" stroke-width="7" stroke-linecap="round"/>
        <path d="M310 470 l14 20 -18 0 Z" fill="var(--cinnabar)"/>
        <!-- 金光描邊 -->
        <path d="M270 160 q-30 0 -30 40 t30 40 q30 0 30 -40 t-30 -40 M200 280 h140 l16 190 h-172 Z"
              fill="none" stroke="var(--gold)" stroke-width="3" opacity=".85"/>
      </g>
    </svg>
    <div id="fuhao-reveal">
      <p id="fuhao-line1">這位——</p>
      <p id="fuhao-name">婦好</p>
      <p id="fuhao-sub">商王的王后——也是帶兵打仗的<strong>女將軍</strong>。<br>
        這不是編的：甲骨上真的刻著，她帶了<strong>一萬三千人</strong>出征。</p>
      <button id="act3-verify-btn" class="next-btn">出征！ →</button>
    </div>
  </section>
```

- [ ] **Step 2: 加 CSS（`</style>` 之前）**

```css
/* --- act3 assemble --- */
.assemble-title{position:absolute;top:44px;width:100%;text-align:center;font-size:30px;
  letter-spacing:.08em;margin:0;color:var(--paper);}
.assemble-title strong{color:var(--cinnabar);}
.assemble-svg{position:absolute;inset:0;}
#bone-frame{}
#bone-pieces .bone-piece{cursor:grab;transition:filter .2s;}
#bone-pieces .bone-piece:hover{filter:drop-shadow(0 0 6px var(--gold));}
#bone-pieces .bone-piece.dragging{cursor:grabbing;filter:drop-shadow(0 0 10px var(--gold));}
#bone-pieces .bone-piece.locked{cursor:default;pointer-events:none;}
#bone-spine{transition:opacity .6s;}
#bone-extend{transition:opacity .6s;}
#fuhao-marks{transition:opacity .8s;}
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

- [ ] **Step 3: 手動視覺驗證**

Run: `start .\index.html` → 從章節圓點跳到第三幕 → 進 assemble：
Expected:
1. 三個全黑剪影（左中右），大小一致、都無臉、都無識別特徵
2. 中央虛框（虛線細）
3. 上方標題「裂紋說：可以打。誰去？王親自看甲骨。」
4. 沒有碎片、沒有白卡出現、沒有金光

Run: `start .\index.html#selftest` → Expected: `ALL PASS (20/20)`（結構沒動測試）。

- [ ] **Step 4: Commit**

```powershell
git add index.html; git commit -m "feat: act3-assemble static scaffold with uniform silhouettes and bone frame"
```

---

### Task 6: 4 片碎骨渲染 + 拖拉互動 + 吸附／彈回

**Files:**
- Modify: `index.html`（在第三幕 IIFE 內或緊接其後加拼骨 IIFE）

**Interfaces:**
- Consumes: `scatterPositions`、`inSnap`、`allLocked`、Layout Constants、`audio.pop`、`scene:enter`
- Produces:
  - 進場（`scene:enter === 'act3-assemble'`）：清空 `#bone-pieces` 並依 `scatterPositions` 灑 4 片
  - 每片一個 `<g class="bone-piece">`，內含：多邊形本體（褐白填色）、多條雜訊細裂線、主脈段（褐色，跟雜訊同色系，先不亮）
  - Pointer 拖動：down 抓、move 跟手、up 若在對應槽位 `inSnap` 內則吸附（動畫 200ms、播 `audio.pop`、加 `.locked`）；否則彈回原本 scatter 位置（動畫 300ms）
  - `pointercancel` / 拖到 viewport 外的 `pointerup` 視同「拖錯」→ 彈回
  - `allLocked` 為 true 時派發自訂事件 `assemble:complete`（Task 7 監聽）

**Fragment data 常數**（tile 中心 = (640, 540)，各片以 tile 本地座標定義形狀 & 主脈段）：

```
Piece A (idx 0, top-left):
  slot: (640, 540)  /* 每片吸附完成後畫在同一 tile 中心；SVG 內部再各自位移 */
  shape (tile-local): "M -110 -110 L 0 -110 L -5 0 L -110 20 Z"
  noise cracks (tile-local):
    "M -90 -70 l 12 8 l -5 14 l 15 6",
    "M -60 -30 l 8 -12 l -10 -8",
    "M -95 -30 l 20 -6"
  spine seg (tile-local): "M -100 -102 L -58 -55"

Piece B (idx 1, top-right, 無主脈):
  shape: "M 0 -110 L 110 -110 L 110 -10 L 5 -20 L -5 0 Z"
  noise cracks:
    "M 50 -80 l 10 6 l -3 12",
    "M 80 -40 l -14 -10 l 4 16",
    "M 30 -60 l 12 -14"

Piece C (idx 2, bottom-left):
  shape: "M -110 20 L -5 0 L 0 110 L -110 110 Z"
  noise cracks:
    "M -70 40 l 10 -8 l 4 16",
    "M -95 70 l 20 4 l -6 14"
  spine seg: "M -58 -55 L -25 -20"    /* 主脈中段：離開 A 進入下方象限 */

Piece D (idx 3, bottom-right):
  shape: "M 5 -20 L 110 -10 L 110 110 L 0 110 Z"
  noise cracks:
    "M 40 40 l -12 8 l 8 14",
    "M 80 60 l 6 -12 l 12 8",
    "M 50 90 l 14 -6"
  spine seg: "M -25 -20 L 15 10 L 55 55"
```

（tile 座標 -110..+110 對應絕對座標 (530..750, 430..650)。渲染時：`g` 使用 `transform` 把自己搬到「拖動位置」或「locked 位置」，內部所有 path 用 tile-local 座標。）

- [ ] **Step 1: 加拼骨 IIFE**

在既有第三幕 JS（`document.getElementById('act3-ask-btn').addEventListener(...)` 那段）之後、`/* 音效測試 */` 之前，插入：

```js
/* ===== 第三幕：拼骨讀裂紋 ===== */
(() => {
  const TILE_CX = 640, TILE_CY = 540;
  const SNAP_THRESHOLD = 60;
  const PIECE_DATA = [
    { shape: 'M -110 -110 L 0 -110 L -5 0 L -110 20 Z',
      noise: ['M -90 -70 l 12 8 l -5 14 l 15 6','M -60 -30 l 8 -12 l -10 -8','M -95 -30 l 20 -6'],
      spine: 'M -100 -102 L -58 -55' },
    { shape: 'M 0 -110 L 110 -110 L 110 -10 L 5 -20 L -5 0 Z',
      noise: ['M 50 -80 l 10 6 l -3 12','M 80 -40 l -14 -10 l 4 16','M 30 -60 l 12 -14'],
      spine: null },
    { shape: 'M -110 20 L -5 0 L 0 110 L -110 110 Z',
      noise: ['M -70 40 l 10 -8 l 4 16','M -95 70 l 20 4 l -6 14'],
      spine: 'M -58 -55 L -25 -20' },
    { shape: 'M 5 -20 L 110 -10 L 110 110 L 0 110 Z',
      noise: ['M 40 40 l -12 8 l 8 14','M 80 60 l 6 -12 l 12 8','M 50 90 l 14 -6'],
      spine: 'M -25 -20 L 15 10 L 55 55' },
  ];
  const SVG_NS = 'http://www.w3.org/2000/svg';
  let pieces = [];         /* {el, spineEl, noiseEls, locked, scatter:{x,y}, dragging} */
  let dragState = null;    /* {piece, offX, offY} */

  function build(){
    const layer = document.getElementById('bone-pieces');
    layer.innerHTML = '';
    const scatter = scatterPositions(4, TILE_CX, TILE_CY, 250, 320);
    pieces = PIECE_DATA.map((d, idx) => {
      const g = document.createElementNS(SVG_NS, 'g');
      g.classList.add('bone-piece'); g.dataset.idx = idx;
      /* 碎片本體 */
      const shape = document.createElementNS(SVG_NS, 'path');
      shape.setAttribute('d', d.shape);
      shape.setAttribute('fill', '#e0d3b8'); shape.setAttribute('stroke', '#8b7a5e');
      shape.setAttribute('stroke-width', '2');
      g.appendChild(shape);
      /* 雜訊細裂 */
      const noiseEls = d.noise.map(nd => {
        const p = document.createElementNS(SVG_NS, 'path');
        p.setAttribute('d', nd); p.setAttribute('stroke', '#6b5a44');
        p.setAttribute('stroke-width', '1.5'); p.setAttribute('fill', 'none');
        p.setAttribute('opacity', '.7'); g.appendChild(p); return p;
      });
      /* 主脈段（拼完前跟雜訊同色，不亮） */
      let spineEl = null;
      if (d.spine){
        spineEl = document.createElementNS(SVG_NS, 'path');
        spineEl.setAttribute('d', d.spine);
        spineEl.setAttribute('stroke', '#6b5a44');
        spineEl.setAttribute('stroke-width', '2'); spineEl.setAttribute('fill', 'none');
        spineEl.setAttribute('stroke-linecap', 'round');
        g.appendChild(spineEl);
      }
      const s = scatter[idx];
      const piece = { el: g, spineEl, noiseEls, locked: false, scatter: s, dragging: false };
      applyTransform(piece, s.x, s.y);
      g.addEventListener('pointerdown', e => onDown(e, piece));
      layer.appendChild(g);
      return piece;
    });
  }

  function applyTransform(piece, x, y){
    piece.el.setAttribute('transform', `translate(${x} ${y})`);
  }
  function animateTo(piece, x, y, ms, onDone){
    const from = getXY(piece);
    const t0 = performance.now();
    const step = now => {
      const t = Math.min(1, (now - t0) / ms);
      const e = t*t*(3 - 2*t); /* smoothstep */
      applyTransform(piece, from.x + (x - from.x) * e, from.y + (y - from.y) * e);
      if (t < 1) requestAnimationFrame(step); else if (onDone) onDone();
    };
    requestAnimationFrame(step);
  }
  function getXY(piece){
    const m = piece.el.getAttribute('transform').match(/translate\(([-\d.]+)\s+([-\d.]+)\)/);
    return { x: parseFloat(m[1]), y: parseFloat(m[2]) };
  }

  function onDown(e, piece){
    if (piece.locked) return;
    dragState = { piece, offX: 0, offY: 0 };
    const cur = getXY(piece);
    const pt = svgPoint(e);
    dragState.offX = pt.x - cur.x;
    dragState.offY = pt.y - cur.y;
    piece.el.classList.add('dragging');
    piece.el.setPointerCapture(e.pointerId);
    e.preventDefault();
  }
  function onMove(e){
    if (!dragState) return;
    const pt = svgPoint(e);
    applyTransform(dragState.piece, pt.x - dragState.offX, pt.y - dragState.offY);
  }
  function onUp(e){
    if (!dragState) return;
    const piece = dragState.piece;
    piece.el.classList.remove('dragging');
    const cur = getXY(piece);
    if (inSnap(cur.x, cur.y, TILE_CX, TILE_CY, SNAP_THRESHOLD)){
      piece.locked = true;
      piece.el.classList.add('locked');
      animateTo(piece, TILE_CX, TILE_CY, 200, () => {
        audio.pop();
        if (allLocked(pieces)){
          document.dispatchEvent(new CustomEvent('assemble:complete'));
        }
      });
    } else {
      animateTo(piece, piece.scatter.x, piece.scatter.y, 300);
    }
    dragState = null;
  }

  function svgPoint(e){
    const svg = document.querySelector('#scene-act3-assemble .assemble-svg');
    const pt = svg.createSVGPoint(); pt.x = e.clientX; pt.y = e.clientY;
    return pt.matrixTransform(svg.getScreenCTM().inverse());
  }

  window.addEventListener('pointermove', onMove);
  window.addEventListener('pointerup', onUp);
  window.addEventListener('pointercancel', onUp);

  document.addEventListener('scene:enter', e => {
    if (e.detail.scene !== 'act3-assemble') return;
    build();
    /* 重置揭曉狀態（Task 7 補上剩下的） */
    document.getElementById('fuhao-reveal').classList.remove('show');
    document.getElementById('bone-spine').style.opacity = 0;
    document.getElementById('bone-extend').style.opacity = 0;
    document.getElementById('fuhao-marks').style.opacity = 0;
  });
})();
```

- [ ] **Step 2: 驗證 selftest 仍 20/20**

Run: `start .\index.html#selftest`
Expected: `ALL PASS (20/20)`。

- [ ] **Step 3: 手動視覺驗證**

Run: `start .\index.html` → 章節圓點跳第三幕 → 進 assemble：
Expected:
1. 4 片褐白碎骨飄浮於畫面環狀範圍（不重疊到剪影與中央虛框太近）
2. 每片上有雜訊細裂線，且**看起來沒有明顯方向**
3. 滑鼠按住碎片可以拖動、跟手
4. 拖到中央虛框內放手：碎片動畫吸附到中心（重疊在虛框位置）、播 pop 音、鎖定（再點無反應）
5. 拖到中央虛框外放手：碎片動畫彈回原本 scatter 位置
6. 4 片全部吸附後：主脈金線 / 白卡 **暫時還不會顯示**（Task 7 才做）；但檢查 devtools console 應無錯

Run: `start .\index.html#selftest` → 再確認 20/20。

- [ ] **Step 4: Commit**

```powershell
git add index.html; git commit -m "feat: act3-assemble bone fragment drag-snap-drift interaction"
```

---

### Task 7: 拼完揭曉——主脈金線 + 婦好剪影填色 + 白卡

**Files:**
- Modify: `index.html`（監聽 `assemble:complete`；`#bone-spine` 動態填內容並淡入；`#bone-extend`；`#fuhao-marks`；白卡顯示；「出征！」跳頁）

**Interfaces:**
- Consumes: `assemble:complete` 事件、Layout Constants、`audio.ding`
- Produces:
  - 進場（`scene:enter === 'act3-assemble'`）：`#bone-spine` 動態填「4 段主脈連起來」的金色 path（tile 座標系）、`#bone-extend` 填「tile top-left → 左剪影腳下」的延伸段（stage 絕對座標）、`#fuhao-marks` 保持 opacity:0 隱藏
  - `assemble:complete`：淡出雜訊（4 片所有 `noiseEls` 動畫 opacity 到 0）、淡入 `#bone-spine`、稍後淡入 `#bone-extend`、稍後淡入 `#fuhao-marks`、稍後播 `audio.ding` 並顯示 `#fuhao-reveal.show`
  - `#act3-verify-btn` 點擊 → `game.goto('act3-verify')`

- [ ] **Step 1: 在拼骨 IIFE 內加 `buildSpine()` 與 `reveal()`**

修改 Task 6 加的 IIFE，於既有 `scene:enter` handler 內的 `build()` 呼叫之後加 `buildSpine()`；並在 IIFE 內加以下函式與 event listener：

```js
function buildSpine(){
  const spineLayer = document.getElementById('bone-spine');
  spineLayer.innerHTML = '';
  /* tile 中心平移到絕對座標，畫在 stage 座標系。合併 4 段主脈為一條 polyline。 */
  const path = document.createElementNS(SVG_NS, 'path');
  /* 主脈四段合起來（tile-local 座標）：D 起 → C → A 出 top-left */
  path.setAttribute('d',
    `M ${TILE_CX + 55} ${TILE_CY + 55} ` +
    `L ${TILE_CX + 15} ${TILE_CY + 10} ` +
    `L ${TILE_CX - 25} ${TILE_CY - 20} ` +
    `L ${TILE_CX - 58} ${TILE_CY - 55} ` +
    `L ${TILE_CX - 100} ${TILE_CY - 102}`);
  path.setAttribute('stroke', 'var(--gold)');
  path.setAttribute('stroke-width', '3.5');
  path.setAttribute('fill', 'none');
  path.setAttribute('stroke-linecap', 'round');
  path.setAttribute('stroke-linejoin', 'round');
  path.setAttribute('filter', 'drop-shadow(0 0 6px var(--gold))');
  spineLayer.appendChild(path);

  const extendLayer = document.getElementById('bone-extend');
  extendLayer.innerHTML = '';
  const ext = document.createElementNS(SVG_NS, 'path');
  /* 從 tile 出口 (540, 438) 走到左剪影腳下 (280, 320)，帶一點鋸齒 */
  ext.setAttribute('d', 'M 540 438 L 480 400 L 420 380 L 360 350 L 320 335 L 280 320');
  ext.setAttribute('stroke', 'var(--gold)');
  ext.setAttribute('stroke-width', '3');
  ext.setAttribute('fill', 'none');
  ext.setAttribute('stroke-linecap', 'round');
  ext.setAttribute('filter', 'drop-shadow(0 0 6px var(--gold))');
  extendLayer.appendChild(ext);
}

function reveal(){
  /* Step A: 淡出所有碎片的雜訊線 */
  pieces.forEach(p => {
    p.noiseEls.forEach(n => { n.style.transition = 'opacity .5s'; n.style.opacity = 0; });
    if (p.spineEl){ p.spineEl.style.transition = 'opacity .5s'; p.spineEl.style.opacity = 0; }
  });
  /* Step B: 550ms 後點亮主脈 */
  setTimeout(() => {
    document.getElementById('bone-spine').style.opacity = 1;
  }, 550);
  /* Step C: 900ms 後延伸線指向左剪影 */
  setTimeout(() => {
    document.getElementById('bone-extend').style.opacity = 1;
  }, 900);
  /* Step D: 1400ms 後左剪影填色顯形＋ding */
  setTimeout(() => {
    document.getElementById('fuhao-marks').style.opacity = 1;
    audio.ding();
  }, 1400);
  /* Step E: 2100ms 後浮出白卡 */
  setTimeout(() => {
    document.getElementById('fuhao-reveal').classList.add('show');
  }, 2100);
}

document.addEventListener('assemble:complete', reveal);
document.getElementById('act3-verify-btn').addEventListener('click', () => game.goto('act3-verify'));
```

同時在 IIFE 內既有 `scene:enter` handler 內加一行 `buildSpine();`：

```js
  document.addEventListener('scene:enter', e => {
    if (e.detail.scene !== 'act3-assemble') return;
    build();
    buildSpine();                                                       /* 新增 */
    /* 重置揭曉狀態 */
    document.getElementById('fuhao-reveal').classList.remove('show');
    document.getElementById('bone-spine').style.opacity = 0;
    document.getElementById('bone-extend').style.opacity = 0;
    document.getElementById('fuhao-marks').style.opacity = 0;
    /* 雜訊線 opacity 復原（給重入本場景用） */
    pieces.forEach(p => {
      p.noiseEls.forEach(n => { n.style.transition = 'none'; n.style.opacity = ''; });
      if (p.spineEl){ p.spineEl.style.transition = 'none'; p.spineEl.style.opacity = ''; }
    });
  });
```

- [ ] **Step 2: 驗證 selftest 仍 20/20**

Run: `start .\index.html#selftest` → Expected: `ALL PASS (20/20)`。

- [ ] **Step 3: 手動視覺完整走查**

Run: `start .\index.html` → 從第二幕結尾按「下一幕 →」：
Expected 全流程：
1. **act3-intro：** 狼煙城牆剪影、紅光警報閃兩輪、鼓聲五連擊
2. 點「立刻問天」→ **burn** 場景，提示「按住龜甲——問這一仗」，按住約 1.6 秒
3. → **act3-assemble：** 三黑影（無識別特徵）、中央虛框、4 片碎骨飄浮
4. 拖第 1 片到虛框內：吸附＋pop；此時只看到 1 片碎骨在中間，其餘 3 片仍飄；**看不出方向**
5. 拖第 2 片吸附後：畫面有 2 片、雜訊線多起來，**仍看不出方向**
6. 拖第 3 片吸附後：**主脈開始隱約成形**（但仍是褐色不亮）
7. 拖第 4 片吸附後：
   - ~0.5 秒後：碎片上的雜訊線淡出 → 金色主脈點亮
   - ~0.9 秒：金色延伸線從骨頭伸向左剪影腳下
   - ~1.4 秒：左剪影浮出髮髻＋硃砂紅戈＋金光描邊，同時 ding
   - ~2.1 秒：白卡「這位——婦好」浮出
8. 點「出征！ →」→ **act3-verify**（空黑場景，正常）
9. 用章節圓點跳回第三幕 intro → 再走進 assemble：碎骨重新散開位置（**跟上輪不同**）、剪影還原黑影、白卡收起
10. 手機測試（若可）：F12 → device mode → 觸控拖動應與滑鼠一致

**若視覺瑕疵（例如碎片飄到剪影上、拼合後主脈與雜訊角度不吻合、延伸線起點跟主脈出口對不齊）：** 微調 Layout Constants 段的常數，同步兩個地方（`PIECE_DATA` spine 座標 + `buildSpine()` 的絕對座標公式）。改完再走一遍。

- [ ] **Step 4: Commit**

```powershell
git add index.html; git commit -m "feat: act3-assemble reveal — gold spine, extend line, Fu Hao fill-in"
```

---

## 完成定義（本計畫驗收）

- [ ] `#selftest` 為 `ALL PASS (20/20)`（原 17 + 新增 3：`inSnap`、`allLocked`、`scatterPositions`）
- [ ] 走一遍第三幕：intro → burn → assemble → verify（空）流程順暢
- [ ] 4 片碎骨每次進場位置不同
- [ ] 拖到虛框內會吸附＋咔；拖離會彈回原本位置
- [ ] 前 2 片拼完看不出方向；第 4 片瞬間出現向左金色主脈與延伸線
- [ ] 左剪影填色顯形（髮髻＋紅戈＋金光），中右仍黑影
- [ ] 白卡浮出「婦好…一萬三千人出征」
- [ ] `出征！ →` 進 act3-verify
- [ ] 章節圓點跳回第三幕 intro：assemble 場景重入時碎骨重新散開、剪影還原、白卡收起
- [ ] 手機／觸控可玩（PointerEvent 兼容）
