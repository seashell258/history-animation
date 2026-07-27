# zhou/ — 越王勾踐十年(retrofit 進行中)

## 現況

本目錄有一個 v1 版本(`index.html`,800 行單檔),已完成六場戲但品質不合格(主角薑餅人、場景銜接斷裂、無 transition)。決定 retrofit 進新的兩階段 skill pipeline。

## 動這個 repo 之前必讀

**必先 invoke `svg-animation-preproduction-gate` skill**。沒有 character sheet + beat sheet 完成並被使用者簽收,禁止寫任何場景 code。

## Retrofit 規則(路徑 A)

- 舊 `index.html` **保留不動**,當 v1 playtest 對照 + 故事/互動 mechanic 的資料來源
- 新場景寫進 `scenes/*.html`,一場一檔
- 新的場景切換 shell 放 `index-v2.html`(不覆蓋 index.html)
- 每個場景檔只准用 `<use xlink:href="../assets/…svg#body"/>` 引用 pre-prod asset,禁止 inline 人物/道具 SVG(允許 inline 純背景幾何、字幕、UI)
- `shared/style.css` 沿用
- 舊 index.html 裡的 Web Audio 引擎、stage 縮放 fitStage、雙押 mechanic 這些 JS 邏輯是資產,可以搬到 shared/ 底下重複使用,但**不可以複製 inline SVG 塊過來**

## 故事 DNA(從 v1 抽出,交給 pre-prod 當輸入)

**場景順序**:homecoming → comfort → turning → decision → ritual → ending
**核心 mechanic**:雙押儀式(左右各按住,同步進度)、閃回覆蓋(馬廄冷調)、決心金光擴散、姿態隨進度變化
**情緒弧**:歸鄉平靜 → 沉溺舒適 → 憶苦驚醒 → 決意站立 → 十年砥礪 → 蓄勢
**目前的問題**(pre-prod 要修的)
- 主角視覺過簡(無髮髻/服飾/手臂),需重新設計達 Shang 標準
- homecoming→comfort、turning→decision 之間無 transition beat,銜接斷裂
- 各場 timing 未文件化,只散在 code 裡

## v1 遵守原則

- 禁止直接編輯 `index.html`(這是歷史對照)
- 禁止從 `index.html` 複製 inline SVG 到新場景(要重新從 character sheet 產生)
- 可以從 `index.html` 抽取:JS 邏輯、shared style、beat 順序推斷、互動 mechanic
