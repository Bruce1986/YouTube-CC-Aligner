# Design Document: YouTube CC Aligner (暫名)

**Date:** 2026-05-03
**Author:** Bruce
**Status:** Draft

## Summary
這是一個基於 Manifest V3 的瀏覽器擴充套件。旨在解決 YouTube 單頁應用程式（SPA）無法針對單一頻道記憶字幕（CC）設定的痛點。透過注入腳本攔截 YouTube 播放器底層 API，實現「Record & Replay（錄製與重播）」機制，確保觀眾在不同語系的頻道間切換時，能自動載入符合該頻道情境的精準字幕設定。

## Overview
本系統為純客戶端架構，核心由三個模組組成：
1. **Event Interceptor (Content Script):** 監聽 SPA 導航事件（`yt-navigate-finish`）與播放器狀態變更。
2. **Player Controller (Injected Script):** 跨越隔離環境（Isolated World），直接與網頁 Main World 的 `movie_player` 全域物件互動，讀寫字幕狀態。
3. **State Manager (Storage):** 基於 `chrome.storage.local` 的輕量級狀態機，負責比對並快取頻道與語言的映射關係。

## Background
YouTube 目前的字幕設定是全域且基於 Session/Cookie 的。使用者若在英文頻道開啟英文字幕，切換到母語頻道時，往往會被迫看著系統自動生成的突兀字幕，必須手動關閉；切換回日文頻道時，又得重新手動開啟日文字幕。這種繁瑣的重複操作嚴重干擾了多語系使用者的觀影體驗。

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

1.  **注入點 API:**
    *   `document.getElementById('movie_player')`
2.  **核心控制方法:**
    *   獲取現有字幕軌：`movie_player.getOption('cc', 'tracklist')`
    *   設定字幕軌：`movie_player.setOption('cc', 'track', {languageCode: 'zh-TW'})`
    *   開/關字幕：`movie_player.toggleSubtitlesOn()`, `movie_player.toggleSubtitlesOff()`
3.  **事件掛載:**
    *   SPA 路由監聽：`window.addEventListener('yt-navigate-finish', handler)`

### Data Model / Storage
使用 `chrome.storage.local`。Schema 採用扁平化設計（Flat Structure），以 Channel ID 為 Key：

```json
{
  "UC_Example_Channel_ID_1": {
    "cc_enabled": true,
    "lang_code": "en",
    "updated_at": 1714732156 
  },
  "UC_Example_Channel_ID_2": {
    "cc_enabled": false,
    "lang_code": null,
    "updated_at": 1714732200
  }
}
```
*註：加入 `updated_at` 有助於未來實作 LRU（Least Recently Used）快取清理機制，避免 Storage 無限膨脹。*

### Dependencies
*   **建置工具：** Vite + TypeScript (提供強型別支援，降低重構風險)。
*   **套件規範：** Manifest V3 (Google Chrome 強制要求)。

## Security
*   **XSS 防禦：** 注入 Main World 的腳本必須是靜態打包好的字串或檔案，嚴禁使用 `eval()` 或動態拼接使用者輸入的字串來執行。
*   **權限最小化：** `manifest.json` 中的 `permissions` 僅申請 `storage` 與特定的 `host_permissions` (`*://*[.youtube.com/](https://.youtube.com/)*`)，絕對不申請 `activeTab` 以外的不必要權限。

## Test Plan
針對 SPA 前端自動化的痛點，測試計畫是此專案的靈魂：
*   **Unit Test (Vitest):** 針對 State Manager 的讀寫邏輯、以及 Heuristic Sniffing (Metadata 解析) 邏輯進行純函數測試。
*   **E2E Test (Playwright / Puppeteer):**
    *   這是最重要的防線。撰寫自動化腳本，實際開啟無頭瀏覽器（Headless Browser），載入 YouTube。
    *   *Test Case 1:* 進入影片 A -> 開啟 CC -> 切換至影片 B -> 確認 CC 狀態 -> 切換回影片 A 頻道的新影片 -> 斷言（Assert）CC 已自動開啟並對齊語言。
    *   *持續整合（CI）：* 設定 GitHub Actions 每週定期跑一次 E2E，若 YouTube 偷偷改版導致 API 失效，我們能在第一時間收到警報。

## Rollout / Deployment Plan
1.  **Alpha Release:** 發布至 GitHub Releases，提供 ZIP 檔供開發者以「未封裝擴充功能」載入測試。
2.  **Public Release:** 提交至 Chrome Web Store 審查，並開源 GitHub Repository。

## Privacy
*   **Zero Data Collection:** 本套件不包含任何 Analytics、追蹤碼或遠端遙測（Telemetry）。使用者的觀看紀錄、頻道偏好與字幕設定 100% 留存在本地硬碟的 Chrome Storage 中。
*   在 Chrome Web Store 的隱私權聲明中，將明確勾選「不收集使用者資料」。

## Caveats
*   **YouTube API 變更：** `movie_player` 屬於 YouTube 內部未公開的 API。雖然歷史證明它非常穩定（存在了十年以上），但 Google 仍可能在未來將其混淆（Obfuscate）或移除。
*   **廣告干擾：** YouTube 插播廣告時，載入的影片容器與主影片不同，攔截腳本需具備判斷當前是否為廣告（Ad-playing state）的能力，避免將字幕設定套用到廣告上。

## Work Estimate
*   **Phase 1 (PoC):** 驗證 Content Script 注入與 `movie_player` API 控制可行性 (約 1-2 天)。
*   **Phase 2 (MVP):** 完成 Record & Replay 核心邏輯與 Storage 串接，能順暢處理 SPA 切換 (約 3-4 天)。
*   **Phase 3 (Polish):** 加入 Cold Start 嗅探邏輯、防禦性邊界測試與 Playwright E2E 腳本 (約 3-5 天)。

---

這份文件抽掉了伺服器 SLA、容量規劃等不適用的項目，並加重了 Test Plan 與 Caveats 的比重。您可以直接把這個架構拿去當作 GitHub Repo 的 `README.md` 或 `CONTRIBUTING.md` 的基礎。

【輸出結束】
