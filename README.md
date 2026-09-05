# 臺大資訊系統訓練班證書通知

使用 Python + uv，每天台灣時間 **09:17** 透過 GitHub Actions 檢查
[第479期課程（6250）](https://train.csie.ntu.edu.tw/school/news/certificate.php?id=6250)。
課程狀態變成「已可領取證書」時，發送通知至 [ntfy.sh/aip479cert](https://ntfy.sh/aip479cert)，通知可點擊開啟領取頁面。

## 啟用

1. 在 ntfy 手機 App 或網頁訂閱 `aip479cert`，允許通知。
2. 將專案推送到 GitHub repository 的預設分支，並確保 GitHub Actions 已啟用。
3. 到 **Actions → Certificate monitor → Run workflow**，選預設分支執行一次。第一次會自動建立 `state` 分支，之後每天寫入紀錄，不需要額外的 secrets。

Workflow 使用內建 `GITHUB_TOKEN` 的 `contents: write` 權限。Repository／組織政策需允許此權限，且 `state` 分支的規則需允許 Actions 直接寫入。手動執行也只接受預設分支，避免從舊版本程式更新正式紀錄。

排程為 `17 1 * * *`（UTC）。GitHub 排程可能延遲；公開 repository 連續 60 天沒有活動時，排程可能自動停用，需要重新啟用。詳見 [GitHub 排程文件](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)。

## 本機執行

先安裝 [uv](https://docs.astral.sh/uv/getting-started/installation/)，接著：

```sh
# 只查詢實際網頁，不發通知、不寫入紀錄
uv run --locked python check_certificate.py --dry-run

# 正式檢查：已開放時發通知
uv run --locked python check_certificate.py

# 測試（不連線、不發通知）
uv run --locked python -m unittest discover -s tests -v
```

## 判斷與通知紀錄

只判斷符合課程標題的 `summary` 區塊，文字必須完整等於「已可領取證書」。連線失敗、標題不符或狀態區塊缺失會讓工作失敗，避免誤發通知；下次排程會重新嘗試。

每次檢查都讀寫獨立的 `state` 分支，與程式所在的預設分支分開，不需要合併：

- `status.json`：課程網址、是否通知過、通知時間，以及最近一次檢查結果。
- `history.jsonl`：每次檢查追加一行 JSON，包含 UTC 時間、網頁狀態、成功／失敗、錯誤訊息、本次是否通知、Actions 執行連結與重跑次數。
- 每次檢查產生一筆 commit；即使狀態沒變或已通知過，仍然查詢網頁並留下紀錄。

ntfy 確認收件後才記為已通知，後續只查詢、不重複通知。網站或 ntfy 發生可捕捉的錯誤時，也會保存失敗紀錄，且 Actions 保持失敗狀態。若尚未進入檢查階段（例如依賴安裝、測試或讀取分支失敗），或 runner 被強制中止，則可能沒有分支紀錄，需看 Actions log。

本機寫入 `.state/status.json` 和 `.state/history.jsonl`，不會自動推送；`--dry-run` 不寫入任何紀錄。Actions 使用 Git worktree 讀寫分支，並以 concurrency 避免排程與手動執行同時更新。

## 示範給同學看

1. 在 GitHub 的分支選單切換到 **state**。
2. 打開 **history.jsonl**，看每天的 `checked_at`、`status` 和 `result`。
3. 點開紀錄裡的 `run_url`，對照該次 Actions 的執行 log。
4. 打開分支的 **Commits**，每次檢查都有一筆提交；也可手動 Run workflow，示範立即多一筆紀錄。

紀錄沒有 cache 的淘汰期限。但通知成功後若 commit／push 失敗，下次仍可能再次通知；push 失敗會讓 Actions 顯示失敗，不會強制覆寫遠端分支。若要主動重發，在 `state` 分支的 `status.json` 將 `notified` 改成 `false` 並移除 `notified_at`，再執行。刪除整個 `state` 分支會清除通知紀錄及執行歷史。

通知使用 [ntfy JSON 發送介面](https://docs.ntfy.sh/publish/#publish-as-json)；Python 環境由 [setup-uv](https://docs.astral.sh/uv/guides/integration/github/) 管理，依賴版本固定於 `uv.lock`。
