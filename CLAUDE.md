# CLAUDE.md

本檔案為 Claude Code（claude.ai/code）在此專案中工作時的指引。

## 專案簡介

一個單一檔案、無建置流程、無外部依賴的 HTML 頁面（`index.html`）：25 分鐘的任務／工作倒數計時器，太空主題（銀河背景、旋轉中的半色調點陣「星球」、可切換的星球色彩主題）。沒有使用任何框架、打包工具或套件管理器。

## 專案規則（每次進來都必須遵守）

1. **深色星空背景；主色（強調色）只保留給當下要強調的那一個元素使用。** 不要讓主色／藍色同時套在多個互相競爭注意力的元素上——效果應該是「現在這一件事最重要」（例如目前選中的控制按鈕、目前啟用的星球 chip），而不是拿來當一般裝飾。
2. **星球與火箭一律只能用 canvas 或 CSS 畫，絕不引用外部圖片檔。** 註：目前**尚未符合**這條規則——`.planet-wrap` 裡的 canvas 是從 `e25a47cddf4f6300fcc5a16d4bbf363a.jpg` 取樣畫出來的（見下方「星球算圖」段落），這就是在引用外部圖片。要讓程式碼符合這條規則（例如把半色調的取樣來源換成程式產生的圖樣）是一項尚未完成的工作，不應該在處理其他無關改動時順手「默默修掉」——如果之後有人要求動到星球視覺，請主動提出這個落差。
3. **按鈕文案必須維持在航太語彙範圍內**：發射（launch）／待機（standby）／返航（return）／補給（resupply）。之後任何新增到計時器介面的控制項，命名都要從這個語彙集裡挑，不要用通用字眼。
4. **倒數的時長必須放在檔案最上面當成單一設定值，不能散落在程式碼各處。** 目前是 `var TOTAL_MS = 25 * 60 * 1000;`，位於 script 中「計時器邏輯」區塊的最上方——之後如果要加其他時長／設定常數，也要統一宣告在同一個靠前的位置，不要把數字直接寫死散在邏輯裡。

## 常用指令

這個 repo 沒有任何 build、lint、test 工具鏈——它就是一個純靜態檔案。

- **本機預覽**：直接在瀏覽器開啟 `index.html`（例如 macOS 下用 `open index.html`），或者如果 `file://` 限制導致兩張 `.jpg` 讀不到，可以用任何靜態伺服器起一個服務（例如 `python3 -m http.server`）。
- **部署**：這個專案已經透過 GitHub import 連接到 Vercel（repo：`jolinatu-svg/mission-timer`，正式網域：`mission-timer-ten.vercel.app`）。推送到 GitHub 的 `main` 分支會自動觸發 Vercel 重新部署——目前沒有在用 Vercel CLI。

## 架構

HTML、CSS、JS 全部都寫在 `index.html` 這一個檔案裡，分別放在單一個 `<style>` 區塊和單一個 `<script>` IIFE 中。沒有拆成多個模組互相參照；要理解這個檔案，從頭讀到尾就是了。

### 分層結構（由後到前）
1. `.sky`——固定滿版的 `<div>`，用 CSS `background` 帶入銀河圖片，透過 `mask-image` 從 `--boundary`（50vh）開始往下淡出，讓它跟下方的星球自然融合、不會有明顯接縫。
2. `.planet-wrap`——一個固定定位、刻意放大（直徑 `190vw`）的圓形，位置經過計算讓它只有頂端一小截「冒出」`--boundary` 之上，做出「地平線」的效果。它的 `box-shadow` 就是大氣輝光，顏色會隨目前選中的星球改變。
3. `.planet-wrap` 內的 `<canvas id="planetCanvas">`——把「星球」畫成半色調點陣，旋轉效果是透過 CSS `@keyframes` 動畫做的（不是 JS 驅動的）。
4. `<img id="planetSource">`——隱藏的，`display:none`。這才是真正的圖片素材；畫面上永遠不會直接顯示它，它的存在純粹是為了給下面的 canvas 取樣用的像素來源。
5. 前景 UI（`.win-dots`、`.planet-nav`、包含計時器／進度條／控制按鈕的 `<main>`）疊在最上層，`z-index` 為 `1`／`2`。

### 星球算圖（從照片產生半色調圖樣的技巧）
`buildSampleData()` 把 `planetSource` 畫進一個 900×900 的離屏 canvas（用 cover 方式裁切），並讀取一次它的像素資料。`drawHalftone(rgb)` 接著以 7px 為間距走訪這份像素資料，在可見的 canvas 上逐格畫一個點，點的半徑／透明度由一個亮度公式決定（`0.22 + rawLum*0.85`，刻意墊高基準值，讓圖片較暗的區域也能畫出可見的點），點的顏色則是目前星球主題的 `rgb`。這個算圖只在切換主題時執行一次，不是每畫格都重繪——你看到的旋轉效果，純粹是 CSS 動畫在轉動這個已經畫好的 canvas 元素而已。

### 星球色彩主題
`PLANET_THEMES` 把 `earth|mars|moon|jupiter|saturn` 各自對應到一組 `rgb` 三元組。點擊 `.planet-nav` 裡的 `.chip` 會設定 `activeChip`、透過 `drawHalftone` 用新顏色重繪半色調 canvas，並透過 `glowShadow(rgb)` 更新 `.planet-wrap` 的輝光顏色。只有地球有「真正對應星球」的意義——其餘四個都是同一張來源照片換個顏色而已，並沒有為每顆星球準備個別的美術素材。

### 計時器狀態機
共四種狀態：`idle | running | paused | finished`，用一個單純的 `state` 字串搭配 `remainingMs`／`endTime` 記錄。剩餘時間是透過 `Date.now()` 的時間差（`endTime - Date.now()`）即時算出來的，而不是靠每次 tick 遞減一個計數器——`tick()`（由 200ms 的 `setInterval` 驅動）只是根據這個計算結果重新渲染，所以就算分頁跑到背景、或計時器有延遲，畫面顯示的時間也不會跟實際時間脫節。`updateUI()` 是唯一負責把「狀態 → DOM」同步起來的地方：計時器文字、進度條寬度、火箭位置、狀態列文字、按鈕的 `disabled`、以及哪個控制按鈕該套上 `.is-selected`（對應規則：running→發射、paused→待機、其餘→返航）。

### 需要延續的既有慣例
- 顏色統一集中定義在 `:root` 的 CSS 自訂屬性裡（`--blue*`、`--panel*`、`--text*`、`--void-*`）——不要在別處直接寫死新的十六進位色碼。
- 大量使用 `border-radius:999px`（膠囊／圓形）在按鈕、chip、進度條圓點上；這是已經確立的造型語言。
- 因為歷史因素，目前存在兩種語意相同但命名不同的「選中」class：`.is-active`（星球 chip 用）和 `.is-selected`（控制按鈕用）。修改時請沿用該元素原本在用的那一套命名，不要自行統一成同一種。
- CSS 的 `:` 後面一律不加空格（例如 `color:var(--text)`），縮排統一用 2 個空格——修改時請維持這個寫法。
- JS 是單一個 `"use strict"` 的 IIFE，全部用 `var` 宣告，沒有用 class／模組／建置流程。新增程式碼請寫在既有的 IIFE 裡面，不要另外開新的 `<script>` 標籤或外部 JS 檔。

### 已知的未盡事宜
- `ed0b7b6bbc9f85599a461b1f98250c21.jpg` 是舊版星球視覺留下來的檔案——`index.html` 目前完全沒有任何地方在引用它。
- `--primary`／`--primary-dark`／`--primary-glow` 這幾個 CSS 變數雖然有定義，但沒有任何地方在使用（是更早期橘色配色方案留下的殘留）。
- `preview-deploy.html` 是先前測試「圖片轉 base64 內嵌後部署」方案時產生的暫存檔；它沒有被 git 追蹤，內容也跟目前的 `index.html` 不同步（用的圖片素材、壓縮方式都不一樣）。可以安全刪除，並不是正式應用程式的一部分。
- 本機 git repo 唯一的一次 commit（「Add mission timer app」）從來沒有真的被推送出去過——GitHub 上的 `main` 分支（`jolinatu-svg/mission-timer`）是另外透過 GitHub 網頁編輯器直接 commit 出來的。兩邊的歷史已經分岔；直接從這個本機 repo 執行 `git push` 不會是 fast-forward。
