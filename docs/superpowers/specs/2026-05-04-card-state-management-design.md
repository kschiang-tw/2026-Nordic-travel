# 景點卡片狀態管理 — 設計規格

**日期**：2026-05-04
**範圍**：travel-app.html
**目的**：讓使用者在旅遊過程中標記「已完成 / 放棄 / 調整到別天」，並支援備份匯出。

---

## 1. 功能總覽

每張景點/行程卡片（`.cd`）可進入下列狀態之一：

| 狀態 | 視覺 | 行為 |
|------|------|------|
| 預設 | 原樣式 | 無變化 |
| ✅ 已完成 | 半透明 (.5) + 綠色 ✓ 徽章 + 綠色左邊框 | 卡片仍在原位 |
| 🗑️ 已放棄 | 半透明 (.4) + 刪除線 + 紅色 ✕ 徽章 | 卡片仍在原位 |
| 📅 已調整 | 卡片從原日移動到目標日的目標位置 | 不另加徽章（移動本身即代表調整） |

「已完成」和「已放棄」是互斥狀態。「已調整」可與「已完成 / 已放棄」並存（卡片移到別天後仍可標記完成）。

---

## 2. 互動設計

### 2.1 長按彈出選單

- **觸發**：使用者長按卡片 500ms（touchstart/mousedown 計時）
- **位置**：卡片右上角浮動 popover
- **內容**：三個圖示 + 文字按鈕（垂直堆疊或水平排列，視寬度而定）
  - ✅ 完成
  - 📅 調整
  - 🗑️ 放棄
- 若卡片已是「✅ 完成」或「🗑️ 放棄」狀態，選單改為：
  - ↩️ 還原（僅還原 state 為 null，不影響「已調整」位置）
  - 📅 調整
- 若使用者要把調整過的卡片移回原日，需用「📅 調整」重新選擇原日和位置（不提供一鍵歸位）
- **關閉**：點選單外任何位置即關閉

### 2.2 「調整」兩步式 modal

點「📅 調整」後彈出全螢幕 modal：

**第 1 步：選日**
- 標題：「移到哪一天？」
- 列出 D1–D16 的縮略卡（每張顯示：DAY XX · 城市名 · 日期）
- 點任一天 → 進入第 2 步

**第 2 步：選插入位置**
- 標題：「選擇插入位置」上方有「← 返回」按鈕
- 顯示目標日當前所有行程，每兩個行程之間有「↓ 插入此處」橫線
- 列表最上方和最下方也有插入點（首位 / 末位）
- 點任一插入點 → 卡片移到該位置，modal 關閉，狀態寫入 localStorage

### 2.3 視覺樣式（已完成 / 已放棄）

```css
.cd.is-done   { opacity:.5; border-left:2.5px solid #5cb85c; }
.cd.is-skip   { opacity:.4; border-left:2.5px solid #d9534f; }
.cd.is-skip .cn { text-decoration:line-through; }

.state-badge { font-size:10px; padding:2px 7px; border-radius:8px; ... }
.state-done  { background:rgba(80,180,80,.2); color:#5cb85c; }
.state-skip  { background:rgba(180,80,80,.2); color:#d9534f; }
```

徽章插入到 `.ch` 末尾（`.ctm` 之後）。

---

## 3. 資料模型

### 3.1 卡片識別（card ID）

卡片 HTML 目前無 ID。啟動時 JS 自動掃描每個 `.cd`：

```
cardId = `${originDay}-${hash(cardName)}`
```

- `originDay`：包含此卡片的 `.pg` 容器 id（例如 `d3`）
- `cardName`：`.cn` 的 textContent（去除前後空白）
- `hash`：簡單 djb2 或 cyrb53 32-bit hash → base36 字串
- ID 寫入 `data-card-id` 屬性，供後續操作使用

ID 對「卡片名稱」穩定 — 只要不改名，ID 不變。

### 3.2 localStorage schema

Key：`hygge2026-card-states`

```json
{
  "version": 1,
  "updatedAt": "2026-05-04T10:30:00Z",
  "cards": {
    "d3-a1b2c3": { "state": "done" },
    "d4-x9y8z7": { "state": "skip" },
    "d5-q1w2e3": { "state": null, "moved": { "toDay": "d7", "index": 2 } },
    "d6-p0o9i8": { "state": "done", "moved": { "toDay": "d8", "index": 0 } }
  }
}
```

- `state`：`"done"` / `"skip"` / `null`
- `moved`：若卡片被調整過，記錄目標日和插入索引；未調整則此 key 不存在
- `cards` 鍵為 cardId

### 3.3 載入順序

1. 讀 localStorage
2. 對每張卡片注入 `data-card-id`
3. 套用 `moved`：將卡片 DOM 節點移動到目標日的目標索引（包覆於 `.tli` 容器內，保持 timeline 結構）
4. 套用 `state`：加上 `.is-done` / `.is-skip` class 與徽章

---

## 4. 備份功能

### 4.1 設定頁 / 工具列入口

travel-app.html 目前沒有「設定」頁。新增方式：

- 在底部 nav bar 增加一顆「⚙️」按鈕，或
- 在 hero 區的右上角放一個小齒輪圖示

點下後彈出設定 modal，包含兩顆按鈕。

### 4.2 匯出

- 按鈕：「📤 匯出備份」
- 行為：產生 `hygge2026-backup-YYYY-MM-DD.json` 並觸發瀏覽器下載
- 內容：完整的 localStorage JSON

### 4.3 匯入

- 按鈕：「📥 匯入備份」→ 觸發 `<input type="file" accept=".json">`
- 行為：讀取檔案 → 驗證格式（檢查 `version` 與 `cards` 結構）→ 確認 dialog（「將覆蓋目前所有狀態，確定？」）→ 寫入 localStorage → reload 頁面

匯入失敗（格式錯誤）時 alert 提示。

---

## 5. 實作架構

travel-app.html 為單檔，新增程式碼直接附加到現有 `<style>` 與檔末 `<script>` 內。

### 5.1 模組劃分（單一 IIFE 內的內部分區）

```js
(function cardStateManager() {
  // --- Storage layer ---
  function load()           // 讀 localStorage
  function save(state)      // 寫 localStorage

  // --- Card identification ---
  function generateCardId(dayId, cardName)
  function indexAllCards()  // 啟動時掃描所有 .cd 注入 data-card-id

  // --- DOM application ---
  function applyMoved(state)   // 重排卡片
  function applyVisualState(state)  // 加 class + 徽章

  // --- Long-press popover ---
  function attachLongPress(card)
  function showPopover(card)
  function hidePopover()

  // --- Adjust modal ---
  function openAdjustModal(cardId)
  function renderDayList()
  function renderInsertPoints(targetDayId)
  function performMove(cardId, targetDayId, index)

  // --- Export/Import ---
  function exportBackup()
  function importBackup(file)

  // --- Init ---
  init()
})();
```

### 5.2 Modal / Popover DOM

新增到 `<body>` 末尾（在現有 script 之前）：

```html
<div id="card-popover" class="card-popover" hidden>...</div>
<div id="adjust-modal" class="adjust-modal" hidden>
  <div class="am-backdrop"></div>
  <div class="am-panel">
    <div class="am-header">...</div>
    <div class="am-body"></div>
  </div>
</div>
<div id="settings-modal" class="settings-modal" hidden>...</div>
```

### 5.3 樣式

新增 CSS 區塊放在現有 `<style>` 末尾：
- `.card-popover` 浮動選單
- `.adjust-modal` 全螢幕 modal
- `.cd.is-done`, `.cd.is-skip` 卡片狀態
- `.state-badge` 徽章
- `.insert-point` 「↓ 插入此處」橫線
- 設定齒輪按鈕

---

## 6. 邊界情況與考量

| 情境 | 處理 |
|------|------|
| 使用者長按連結（`.dd a`）或 chip | 不觸發長按選單（檢查 event.target，若為 anchor 或 chip 則跳過計時） |
| 同名卡片在不同天 | 不衝突（cardId 含 originDay，即使移動後 ID 仍以原日為前綴） |
| 卡片 name 改了（HTML 更新） | ID 變化，舊狀態失效；視為已讀資料遺失（屬於罕見情況，由匯出備份保險） |
| localStorage 容量超限 | 不太可能（資料量小於 10KB），萬一錯誤則 try/catch 後 alert |
| 匯入舊版 JSON | 用 `version` 欄位判斷；目前只有 v1，未來升級時加 migration |
| 移動到原本所在的位置 | 不視為錯誤，正常寫入即可（idempotent） |
| 卡片在 hero 區或 stat grid 等非 `.tli` 容器 | 只處理 `.tli > .cd` 結構的卡片；其他區塊（如「精選看點」摘要）不附加長按 |

---

## 7. 不在範圍內

- 跨裝置即時同步（不做 Google Drive / URL 分享，只做匯出/匯入）
- 撤銷（undo）— 操作後不提供 toast undo，要還原請直接再長按「↩️ 還原」
- 操作歷史紀錄
- 多人共享狀態
- 自訂卡片內容（編輯名稱、新增景點）
