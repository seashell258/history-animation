# zhou/assets — 角色資產規範

Pre-production Step 1 產物。**場景 code 只准 `<use>` 引用本目錄,禁止 inline 人物 SVG。**

## 目前狀態

| 類別 | 狀態 |
|------|------|
| `characters/` | ✅ 已完成(5 角 15 姿勢) |
| `props/` | ✅ 已完成(3 件 6 symbol) |
| `scene-elements/` | ✅ 已完成(4 件 9 symbol) |
| `docs/beat-sheet.md` | ⬜ 未做(Step 2) |

## 道具 props

| 檔案 | symbols | 用於 | 錨點 |
|------|---------|------|------|
| `props/bowl.svg` | `bowl-full` `bowl-empty` | comfort → turning 的 match cut | 碗口 y=10 |
| `props/xin.svg` | `xin-pile` `xin-scatter` | decision 滑入 / ritual 左互動點 | 地面 y=132 |
| `props/dan.svg` | `dan-hanging` `dan-sac` | decision 滑入 / ritual 右互動點 | 吊繩起點 (65,0) |

**碗的兩個 symbol 外形完全相同**,只有內容物不同 —— match cut 就是靠這個成立的,
改其中一個時另一個必須跟著改。

## 場景元素 scene-elements

| 檔案 | symbols | 用於 | 錨點 |
|------|---------|------|------|
| `scene-elements/couch.svg` | `couch` `couch-screen` | comfort / turning / decision | **榻面 y=40**、地面 y=160 |
| `scene-elements/lamp.svg` | `lamp-stand` `lamp-flame` | homecoming 宮室 | 地面 y=252、燈盤 (60,118) |
| `scene-elements/window.svg` | `window-lattice` `window-moonlit` | turning 驚醒 | — |
| `scene-elements/stable.svg` | `stable-bay` `stable-trough` `stable-hay` | 馬廄閃回 | `stable-bay` 寬 200 可水平平鋪 |

**燈與 v1 不同**:v1 用懸掛紙燈籠,但紙燈籠是漢以後才有的,春秋時期是青銅豆形油燈。
這裡做了歷史正確的版本 —— 若要改回燈籠請說。

`lamp-flame` 獨立成一個 symbol,是為了讓場景能單獨對火焰做動畫
(外部 `<use>` 進不去 symbol 內部,所以需要分開的部件才能個別驅動)。

### 榻與半躺姿勢的對位

`gj-seated-relaxed` 的榻面在該 symbol 的 `y=286`,`couch` 的榻面在 `y=40`。
角色放在 `(cx, cy)` 且 `height=340` 時,榻要放在 **`by = cy + 246`**。

## 角色清單

| 檔案 | 角色 | 姿勢 symbols | 出場 |
|------|------|------|------|
| `characters/goujian.svg` | 越王勾踐(主角) | `gj-walking` `gj-seated-relaxed` `gj-seated-upright` `gj-laboring` `gj-rising` `gj-standing` `gj-standing-ten` | 全劇 |
| `characters/fanli.svg` | 范蠡(謀臣) | `fl-kneeling` `fl-standing` | homecoming / decision |
| `characters/wenzhong.svg` | 文種(大夫) | `wz-kneeling` `wz-standing` | homecoming / decision |
| `characters/fuchai.svg` | 吳王夫差(反派) | `fc-looming` `fc-pointing` | 馬廄閃回 |
| `characters/servant.svg` | 侍者 | `sv-offering` `sv-standing` | comfort |

## 檔案關係:唯一真相來源是 `characters/*.svg`

`contact-sheet.html` **不含任何圖形資料**,它只有 5 個 `<img src="characters/*.svg">`。
每個角色 SVG 檔本身也分兩段,圖形只存在其中一段:

| 區段 | 用途 | 場景會用到嗎 |
|---|---|---|
| `<defs>…</defs>` | **真正的資產** — 零件 + 姿勢 symbol | ✅ 場景 `<use>` 的就是這裡 |
| `</defs>` 之後 | 預覽排版(深色底 + 中文標籤 + `<use>` 排成一列) | ❌ 純粹給人審查用 |

預覽區塊裡沒有任何 `<path>`,它只是 `<use>` 引用同一批 symbol。所以:

```
contact-sheet.html → <img> → 檔尾預覽 <use> → <defs> 裡的 symbol（圖形只在這裡）
```

**改圖只改 `<defs>`。** 三層會自動跟著變。

> ⚠️ 做內嵌 build step 時**只能取 `<defs>` 的內容**,檔尾預覽必須丟掉 ——
> 否則那塊 `1700×430` 的深色 rect 和中文標籤會被塞進場景畫面裡。

## 統一座標契約

**每個姿勢 symbol 的 `viewBox` 都是 `0 0 200 340`,地面線一律 `y=332`。**
場景端因此可以用同一組座標擺放任何角色,不需個別調整:

```html
<use xlink:href="../assets/characters/goujian.svg#gj-standing"
     href="../assets/characters/goujian.svg#gj-standing"
     x="540" y="260" width="200" height="340"/>
```

`x`/`y` 是左上角,所以角色腳底 = `y + 332 × (height/340)`。等比縮放改 `width`/`height` 即可。

**兩個例外是 `0 0 300 340`**(橫向體態需要橫向空間,比例與其他姿勢相同,
擺放時給 `width="300" height="340"` 即可):

- `gj-seated-relaxed`(癱躺側面)— 基準不是地面而是**榻面 `y=286`**,場景的榻請對齊這條線。
  腳離地是刻意的(沉溺在舒適裡)。碗 prop 放在手心 `(144,218)`。
- `gj-laboring`(馬廄伏地挺身側面)— 基準是地面 `y=332`,手掌與腳尖都踩在這條線上。

### 勾踐的三顆頭

| 頭 | 用於 | 特徵 |
|---|---|---|
| `gj-head` | 正面所有姿勢 | 髮髻 + 金笄 + 額帶冠 |
| `gj-head-profile` | `gj-seated-relaxed`、`gj-laboring` | 側臉(鼻/唇/下顎輪廓),髻在後腦 |
| `gj-head-ten` | `gj-standing-ten` | 白鬚白髭白鬢 + 灰髮 + 法令紋眼袋 |

**三顆頭都保留髮髻與額帶冠** —— 角色辨識度優先,任何場次都要一眼認出是同一個人。
馬廄的狼狽感不靠改頭,靠 `gj-arm-*-coarse`(袖捲到肘上、前臂裸露)、赤足、
草繩腰帶取代玉組綬、破爛下襬、污漬與汗滴。

## 身高即身分(刻意設計)

頸根 y 座標越小 = 人越高。這是視覺化的權力階序,不要在場景裡改掉:

| 角色 | 頸根 y | 意義 |
|------|--------|------|
| 夫差 | 88 | 最高 — 俯視壓迫 |
| 勾踐 | 106 | 主角 |
| 范蠡 / 文種 | 118 | 臣 |
| 侍者 | 150 | 最矮 |

## 手臂關節構造(硬性規格)

所有角色的手臂都是**兩段 + 肘關節**,不是一根線:

```
<g transform="translate(肩x,肩y) rotate(上臂角)">
  <use href="#xx-arm-upper"/>              <- 肩 → 肘
  <g transform="translate(0,上臂長) rotate(前臂角)">
    <use href="#xx-arm-fore"/>             <- 肘 → 腕 → 袖口 → 手
  </g>
</g>
```

`#xx-arm-fore` 的頂端有一顆深色 `<circle>` = 肘關節,所以彎曲處永遠有實體交代。
**要改姿勢請改 `rotate` 角度,不要重畫手臂路徑。**

上臂長度(即前臂 `translate` 的 y):勾踐/文種/范蠡 = 52–56,夫差 = 60,侍者 = 46。

## 調色盤

角色檔一律用**字面 hex,不用 `var(--…)`** — 因為 asset 檔要能單獨開啟檢視,那時沒有 `shared/style.css`。
色值與 `shared/style.css` 對齊:

| 角色 | 主袍 | 暗面 | 亮面 | 緣飾 | 帶 |
|------|------|------|------|------|-----|
| 勾踐 | `#3f5d55` | `#2c4640` | `#4a6b62` | `#d9ceae` | `#8a5a24` |
| 范蠡 | `#4a5a78` | `#35435e` | `#5c6d8c` | `#cdd6e0` | `#2f3a4d` |
| 文種 | `#6d4a3c` | `#4f342a` | `#825b4a` | `#e0cdb8` | `#3d2a1f` |
| 夫差 | `#2d3f5c` | `#1e2c43` | `#3b5273` | `#8a2f2f` | `#6a2020` |
| 侍者 | `#5c5a48` | `#454434` | `#6e6c58` | `#a8a48c` | `#3a3a2c` |

膚色 `#d8b48c`(夫差 `#d0aa80`、侍者 `#c9a078`)、金 `#c9982a`、髮 `#1a1512`。

**閃回冷調不要改 asset 檔**,由場景端疊 overlay 或 `filter` 處理(夫差檔存的是正常色)。

## 改資產的規則

發現角色哪裡不對 → **回來改這裡的 asset 檔**,不要在場景 code 就地 patch。
所有場景都 `<use>` 同一份,改一次全部更新。這是本 pipeline 的核心;就地 patch 就是 v1 崩塌的路。

## ⚠️ 外部 `<use>` 與 `file://` 的已知限制

`<use href="../assets/…svg#id">` 跨檔引用在瀏覽器直接用 `file://` 開啟時**會靜默失敗**(每個檔案被視為不同 origin,人物整個不出現)。

- **開發/驗收時**:在 `zhou/` 跑 `python -m http.server` 然後開 `http://localhost:8000/…`,一切正常。
- **最終交付**:execution 階段做一個 build step,把 asset 的 `<defs>` 內嵌進單檔 HTML。
  上面已驗證跨檔 **id 無衝突(46 個 id 全域唯一)**,可以安全合併。

授課用單檔需求不影響本規範 — 作者端仍然只寫 `<use>`,內嵌交給 build。
