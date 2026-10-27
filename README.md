# Design Document: YouTube CC Aligner (暫名)

- **Date:** 2026-05-03
- **Author:** Bruce
- **Status:** Draft（本 repo 目前只有設計文件、尚無程式碼；下文的架構、API、測試與發布流程皆為規劃）

## Summary
這是一個基於 Manifest V3 的瀏覽器擴充套件。旨在解決 YouTube 單頁應用程式（SPA）無法針對單一頻道記憶字幕（CC）設定的痛點。透過注入腳本攔截 YouTube 播放器底層 API，實現「Record & Replay（錄製與重播）」機制，確保觀眾在不同語系的頻道間切換時，能自動載入符合該頻道情境的精準字幕設定。

## Overview
本系統為純客戶端架構，核心由三個模組組成：
1. **Event Interceptor (Isolated World Content Script):** 監聽 SPA 導航事件（`yt-navigate-finish`），並作為 Main World 與 Storage 之間的唯一橋接。
2. **Player Controller (Main World Content Script):** 一般 Content Script 跑在隔離環境（Isolated World），看不到網頁的 JS 物件，因此這一段要在網頁的 Main World 執行，才能與 `movie_player` 物件互動、讀寫字幕狀態。注入方式採 `content_scripts` 的 `"world": "MAIN"`（Chrome 111 起支援，不需 `scripting` 權限）；與 Isolated World 之間以 `window.postMessage` 或 CustomEvent 通訊。播放器狀態與字幕變更也在這一段監聽；Main World 腳本不能呼叫 `chrome.*` API，讀寫設定都要透過訊息請 Isolated World 代為存取。
3. **State Manager (Isolated World):** 唯一呼叫 `chrome.storage.local` 的模組，基於它 的輕量級狀態機，負責比對並快取頻道與語言的映射關係。

## Background
依作者的使用觀察，YouTube 的字幕偏好是全域的、不會按頻道記憶（背後機制未查證）。使用者若在英文頻道開啟英文字幕，切換到母語頻道時，往往會被迫看著系統自動生成的突兀字幕，必須手動關閉；切換回日文頻道時，又得重新手動開啟日文字幕。這種繁瑣的重複操作嚴重干擾了多語系使用者的觀影體驗。

## Goals & Non-Goals
**Goals:**
*   精準記憶使用者在「特定頻道」的最後一次字幕開關與語言設定。
*   對於未記錄過的新頻道，透過 Metadata 嗅探（Heuristic Sniffing）提供合理的預設降級（Fallback）策略。
*   擴充套件必須輕量、無痕，且不影響影片載入速度。

**Non-Goals:**
*   不處理影片自動翻譯（Auto-translate）邏輯，僅操作上傳者提供或系統原生辨識的字幕軌道。
*   不依賴任何外部 NLP（自然語言處理）API 來辨識影片語言。
*   不提供跨設備的雲端同步（第一階段先求 Local 穩定，避免引入權限爭議）。

## Alternatives Considered
*   **模擬 UI 點擊（DOM Manipulation）：** 透過腳本尋找並點擊畫面上的 `[CC]` 按鈕與設定選單。
    *   *捨棄原因：* 極度脆弱。YouTube 的 DOM 結構頻繁變動，且多語言介面下的字串匹配容易出錯。
*   **全域偏好語言設定：** 在套件選單設定「優先顯示繁體中文，其次英文」。
    *   *捨棄原因：* 無法滿足「英文頻道看英文、日文頻道看日文」的「原音配原字」需求。

## Detailed Design

### API / Interface Definition
系統將與 YouTube 的未公開 API（Undocumented API）進行互動，這也是技術風險最高的部分，需建立嚴謹的 Try-Catch 與 Retry 機制。

> 以下方法名稱、參數形式與事件名稱都沒有官方文件，是社群觀察所得、尚未驗證，列為 Phase 1 PoC 要先確認的項目。同樣待 PoC 確認的還有：如何偵測使用者手動切換字幕、如何避免把 Replay 自己呼叫的 `setOption` 誤記成使用者選擇（例如 replay 期間以旗標忽略回呼）、Channel ID 從哪裡取得與在哪個 world 讀取，以及 `yt-navigate-finish` 的 detail 能否跨 world 讀取。

1.  **注入點 API:**
    *   `document.getElementById('movie_player')`
2.  **核心控制方法:**
    *   獲取現有字幕軌：`movie_player.getOption('cc', 'tracklist')`
    *   設定字幕軌：`movie_player.setOption('cc', 'track', {languageCode: 'zh-TW'})`
    *   開/關字幕：`movie_player.toggleSubtitlesOn()`, `movie_player.toggleSubtitlesOff()`
3.  **事件掛載:**
    *   SPA 路由監聽：`document.addEventListener('yt-navigate-finish', handler)`（事件派發的目標物件也待 PoC 確認）

### Data Model / Storage
使用 `chrome.storage.local`。Schema 採用扁平化設計（Flat Structure），以 Channel ID 為 Key：

```json
{
  "UC_Example_Channel_ID_1": {
    "cc_enabled": true,
    "lang_code": "en",
    "updated_at": 1777766400
  },
  "UC_Example_Channel_ID_2": {
    "cc_enabled": false,
    "lang_code": null,
    "updated_at": 1777766460
  }
}
```

*註：`updated_at` 為 Unix 時間（秒；JS 的 `Date.now()` 是毫秒，寫入前要除以 1000）。加入 `updated_at` 有助於未來實作 LRU（Least Recently Used）快取清理機制，避免 Storage 無限膨脹。*

### Dependencies
*   **建置工具：** Vite + TypeScript (提供強型別支援，降低重構風險)。
*   **套件規範：** Manifest V3 (Google Chrome 強制要求)。

## Security
*   **XSS 防禦：** Main World 腳本是 manifest `content_scripts` 宣告的靜態打包檔案，嚴禁使用 `eval()`、`new Function` 或動態拼接字串執行。
*   **訊息通道驗證：** `window.postMessage` 可被頁面上任何腳本偽造。Isolated World 收訊時須檢查 `event.source === window`、固定的訊息 type 與欄位 schema，並驗證 Channel ID、`lang_code` 格式後才寫入 Storage。
*   **權限最小化：** `manifest.json` 的 `permissions` 只宣告 `["storage"]`；注入範圍由 `content_scripts.matches`（`https://*.youtube.com/*`）限定，`host_permissions`（MV3 中是獨立的頂層欄位）預設不宣告，PoC 確認需要時才加。不申請 `activeTab`、`tabs`、`scripting` 等其他權限。

## Test Plan
針對 SPA 前端自動化的痛點，測試計畫是此專案的靈魂：
*   **Unit Test (Vitest):** 針對 State Manager 的讀寫邏輯、以及 Heuristic Sniffing (Metadata 解析) 邏輯進行純函數測試。
*   **E2E Test (Playwright / Puppeteer):**
    *   這是最重要的防線。撰寫自動化腳本，實際開啟瀏覽器載入 YouTube。載入擴充套件需要 headed 模式或 `--headless=new`；對真實 YouTube 自動化也可能被機器人偵測擋下，PoC 階段要一併確認可行性。
    *   *Test Case 1:* 進入 A 頻道影片 1 -> 開啟 CC（en） -> 進入 B 頻道影片 -> 斷言 B 頻道依自己的設定（未記錄則走預設降級） -> 進入 A 頻道影片 2 -> 斷言（Assert）CC 已自動開啟且語言為 en。
    *   *持續整合（CI）：* 規劃每週在 GitHub Actions 定期跑一次 E2E，作為 YouTube 改版導致 API 失效的預警（最差約延遲一週發現；能否通過機器人偵測待 PoC 評估）。

## Rollout / Deployment Plan
1.  **Alpha Release:** 發布至 GitHub Releases，提供 ZIP 檔供開發者以「未封裝擴充功能」載入測試。
2.  **Public Release:** 提交至 Chrome Web Store 審查，並開源 GitHub Repository。

## Privacy
*   **Zero Data Collection:** 本套件將不包含任何 Analytics、追蹤碼或遠端遙測（Telemetry）。使用者的觀看紀錄、頻道偏好與字幕設定 100% 留存在本地硬碟的 Chrome Storage 中。
*   在 Chrome Web Store 的隱私權聲明中，將明確勾選「不收集使用者資料」。

## Caveats
*   **YouTube API 變更：** `movie_player` 屬於 YouTube 內部未公開的 API。沒有任何穩定性保證，Google 隨時可能將其變更、混淆（Obfuscate）或移除。
*   **廣告干擾：** YouTube 插播廣告時播放器狀態可能與正片不同（具體判定方式待 PoC 確認），攔截腳本需能判斷當前是否為廣告（Ad-playing state），避免將字幕設定套用或記錄到廣告上。

## Work Estimate
*   **Phase 1 (PoC):** 驗證 Content Script 注入與 `movie_player` API 控制可行性 (約 1-2 天)。
*   **Phase 2 (MVP):** 完成 Record & Replay 核心邏輯與 Storage 串接，能順暢處理 SPA 切換 (約 3-4 天)。
*   **Phase 3 (Polish):** 加入 Cold Start 嗅探邏輯、防禦性邊界測試與 Playwright E2E 腳本 (約 3-5 天)。
