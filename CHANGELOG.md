# CHANGELOG

所有版本的重要變更記錄於此，格式依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)。

版本號遵循 [Semantic Versioning](https://semver.org/lang/zh-TW/)：`MAJOR.MINOR.PATCH`

---

## [1.0.4] — 2026-08-05

> 依 `docs/reports/ark-team-agent-package-analysis-2026-08-05.md` 的 P0 優先序修正。

### 安全性

- **`reply_file` 加入路徑白名單** — 原本接受任意絕對路徑（docstring 自述「發送任意檔案」），
  而 `file_path` 由 LLM 呼叫端提供、可能受提詞注入影響，等同 `.env` / `secrets/` / `~/.ssh`
  的外洩管道。改為只允許 home 與 `agents/{instance}` 底下的
  `output` / `artifacts` / `docs` / `knowledge`（agent 另含 `scripts`）。
- **`wiki_ingest` 加入路徑封閉檢查** — 原本 `wiki_root/"raw"/source_path` 未做 containment，
  含 `../` 即可跳出；且 fallback 為 `Path(source_path)`，明確允許任意絕對路徑。
  新增共用工具 `_contained_path()` 限制在 `raw/` 底下，並區分「越界」與「找不到」。

### 修正

- **排程的 cron 日/月/星期欄位現在真正生效** — 舊 `_parse_cron` 只回傳
  `(minute_set, hour_set)`，docstring 自述 "only minute + hour used"，導致
  `0 18 * * 1`（每週一）每天觸發、`30 8 * * 1-5`（平日）週末照跑。
  移除手寫解析器，改用早已是依賴的 apscheduler `CronTrigger`。
  ⚠️ APScheduler 的 `day_of_week` 數字是 0=週一、標準 cron 是 0=週日，
  故新增 `_normalize_dow()` 先轉成名稱形式，避免所有排程偏移一天。
- **`job.timezone` 現在會影響「是否觸發」** — 舊實作先用全域時區比對 cron，
  比對通過後才讀 `job.timezone` 重算，而該值只用於展開 `{date}`，
  設定者以為改了執行時間但實際沒有。

### 測試

- 703 → 733 passed（新增 30 個）
- 新增 `tests/test_mcp_path_guard.py`：封閉檢查、目錄穿越、絕對路徑、`~` 展開、多 root，
  以及兩個 handler 的越界／不存在區分與正常路徑
- `tests/test_scheduler.py` 改測 `_normalize_dow` / `_to_crontab` / `_cron_matches`，
  含日／月／星期的迴歸案例

---

## [1.0.3] — 2026-07-31

### 修正

- startup 通報去重：相同 `chat_id` 只發一條啟動訊息（修正每個 agent 各發一條的問題）

---

## [1.0.2] — 2026-07-31

### 修正

- `chat_router`：動態找 leader/admin，不 hardcode `pm-agent`（支援各專案自訂 agent 名稱）
- `api.py`：補 `import os`（修正 `send_to_instance` 500 error）

---

## [1.0.1] — 2026-07-31

### 修正

- `wiki_ingest` 路徑修正 + `index`/`log` 自建
- `daemon` mkdir 修正
- scheduler race condition 修正
- LLM retry 機制
- `decision_manager` / `team_mcp` 相對路徑改用 `get_home()`
- token error、port env、`file_lock` mkdir 修正

---

## [1.0.0] — 2026-07-30

### 新增

**Decision Loop（agent 自主拍板迴路）**
- `config/authority-matrix.yml`：L1/L2/L3 三層決策權限矩陣
- `DecisionManager`：decision-request 路由、reply() hook 解析 decision 塊、DecisionStore（SQLite）
- ark-agent fallback 拍板：60 分鐘 ceo/cto 未回應 → ark 自主決策（有前例）或升 Paddy（無前例）
- 每日 09:00 拍板日報 + 翻案按鈕（24h 窗口）
- `policy_update` 意圖（第 8 種）：口語方針 → 結構化 YAML → 寫入 BRAIN.md + Wiki
- 週翻案率統計 + 矩陣升降級提案

**Watchdog 兜底規則**
- `legacy-question-convert`：舊格式開放問句自動轉譯為 decision-request
- `decision-remind`：30 分鐘未拍板重發提醒
- `decision-escalate`：60 分鐘未拍板升 ark-agent（有 fallback）或 Paddy（無 fallback）
- `decider-stuck`：ceo/cto 自身卡住 30 分鐘直升 Paddy

**Telegram UX 優化**
- 啟動通知 3 版（正常 / 重啟 / 部分失敗）+ HTML 格式 + 預算顯示
- 掛起通報：上次活動時間 + 按鈕優化
- `/status` HTML 分組（指揮 / 專案 / 執行層）+ 中文狀態

**ark-agent 私訊架構**
- 8 種意圖分類（route_cto / route_ceo / route_pm / knowledge_qa / chat / log_journal / unclassified / policy_update）
- `fallback-decision.md` steering：recall → 自拍或升 Paddy → 知識回饋

**基礎設施**
- ToolTracker soft_reset + delete（消除多條工具狀態訊息）
- queue worker delay 5s + pipe broken overflow
- 啟動通報誤報修正（排除 awaiting_reply 狀態）

### 測試
- 21 個單元測試通過（DecisionStore + DecisionManager 全覆蓋）
- 驗收標準：AT-1 ~ AT-12 + AT-F1 ~ AT-F3

---

*初始版本，由 paddyyang 開發，AI Team（nana-team-agent）協同產出文件與測試。*
