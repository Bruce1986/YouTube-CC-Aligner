# Gemini 執行指南

本文件是 Gemini Code Assist 專用的執行指南。

---

## 開始工作前：先讀懂這個專案

**在寫任何程式碼或發起任何任務之前，請先閱讀 [`project-handbook.md`](project-handbook.md)。**

重點確認：
1. **這個專案是什麼**（專案身份表格）
2. **這個專案不做的事**（⚠️ 警告列）
3. **目前任務清單**（只做清單裡有的事）

若任務清單是空的，代表這個專案尚未定義任何工作，**請停下來，告知使用者需要先填寫任務清單。**

設計文件與架構說明請見 [`README.md`](README.md)。

---

## PR Review 流程

收到 `/Gemini review` 指令後，依以下流程循環執行直到所有 comment 解決：

1. `git fetch origin && git rebase origin/main`
2. 修正 comment
3. 跑測試
4. `git commit && git push`
5. 留言 `/Gemini review`
6. 等待約 150 秒後，回到步驟 1

---

## 程式碼審查原則

審查時著重以下幾點，有問題就提出，不要視而不見：

- **正確性**：邏輯是否正確？有無邊界案例未處理？
- **命名**：變數、函式命名是否清晰有語意？
- **錯誤處理**：失敗路徑是否有妥善處理與 log？
- **測試**：新增的函式是否有對應的 unit test？
- **文件一致性**：若有對外介面異動，是否已更新 `../INTERFACES.md`？

---

## 撰寫程式碼的原則

- 以可驗證的事實和嚴謹邏輯為基礎，不捏造資料
- 遇到不確定的地方，直接說明而非自行假設
- 複雜任務先在 `IMPROVEMENT_PLAN.md` 制定計畫再動手
- 重要操作記錄到 `WORKLOG.md`
- 撰寫新 function 時，為其撰寫對應 unit test；若不可行，在 PR 說明原因

---

## 執行環境注意事項

預設執行環境為 **macOS**。執行前先確認當前系統，避免使用錯誤的指令。

### 指令相容性

| 需求 | 建議用法 |
|------|----------|
| 刪除檔案 | `rm <file>` |
| 刪除目錄 | `rm -rf <dir>` |
| 跨平台檔案操作 | 用 Python 腳本（`temp_ops.py`）處理 |

### Git Commit Message

若訊息含特殊字元，改用檔案方式提交：

```bash
git commit -F COMMIT_EDITMSG
```

格式：`<類型>[作用域]: <描述>`，類型使用 `feat / fix / docs / style / refactor / test / chore`。

---

## 專案特定注意事項

{{若此專案有特定的環境設定、API 金鑰管理方式、部署流程或其他 Gemini 需要知道的事，寫在這裡。若無，刪除此段。}}
