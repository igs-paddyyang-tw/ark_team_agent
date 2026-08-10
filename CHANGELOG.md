# CHANGELOG

所有版本的重要變更記錄於此，格式依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)。

版本號遵循 [Semantic Versioning](https://semver.org/lang/zh-TW/)：`MAJOR.MINOR.PATCH`

---

## [1.2.4] — 2026-08-10

- N5 Phase B1：內建 `_builtin:memory-consolidate`（各 agent daily log >14 天搬 archive/、memory.md 超標記 degraded；純機械、無 LLM、非破壞性）。

## [1.2.3] — 2026-08-10

- 修 TG 出口 ANSI 殘留：`reply()` 路徑新增 `_strip_ansi`，套 `_send_to_topic`/`_send_to_private` 最終出口。

## [1.2.2] — 2026-08-10

- N4 Session 派工脈絡：`_last_dispatch` + `/api/send` 記錄/注入；`communication.dispatch_context_ttl_minutes`（30，0=關）。

## [1.2.1] — 2026-08-10

- 修 `/start` Web Dashboard URL：寫死 23030 → 動態 `health_port+300`；僅在 team-website 存在時顯示。

## [1.2.0] — 2026-08-10

- 整合升級（全 additive）：per-instance `task_prefix/suffix`、`knowledge_search_order`、`/api/health` 增 `alive`、Reply Template System（6 具名模板）。

## [1.1.13] — 2026-08-10

### 新增

- **`send_to_instance` peer-reply timeout 語意優化** — 當目標 agent 超時未回應時，回傳明確的 timeout 狀態而非靜默失敗，呼叫端可據此決策是否重試或升級

---

## [1.1.12] — 2026-08-10

### 修正

- **agent 程式碼誤判治本** — 修正 error-class 分類邏輯將正常 code output 誤標為「生成失敗」
- **補生成失敗/崩潰觀測** — 新增可觀測性指標，區分「真正的崩潰」與「程式碼區塊誤判」

---

## [1.1.11] — 2026-08-09

### 修正

- **生成失敗誤判 auth** — 修正 authentication 狀態被錯誤歸類為生成失敗
- **`awaiting_reply` 永久卡住** — 修正 agent 狀態機在特定錯誤路徑下停留在 awaiting_reply 不回復

---

## [1.1.10] — 2026-08-09

### 新增

- **`_md_to_html` 擴充** — Telegram 訊息渲染新增支援斜體（`*text*`）、刪除線（`~~text~~`）、引用（`> text`）格式

---

## [1.1.9] — 2026-08-08

### 修正

- **多組架構訊息路由缺陷修復** — 修正多個 team 同時運行時，訊息路由可能發送到錯誤 group/topic 的問題（fish-team-agent 實戰回報）

---

## [1.1.8] — 2026-08-08

### 修正

- **重啟自動觸發邏輯** — 多 leader 架構下不再只挑第一個 leader 觸發重啟；無中斷任務時靜默重啟（不發通報）

### 測試

- 802 passed

---

## [1.1.7] — 2026-08-08

### 修正

- **ToolTracker 版面人性化** — 工具執行狀態通報改為單行階段摘要，取代原始工具清單（減少訊息噪音）

---

## [1.1.6] — 2026-08-07

### 修正

- **P2P 通報改發 General** — 跨 agent P2P 通訊的通報訊息改發 General Topic（而非各自 topic），避免訊息分散
- **放寬 admin 數量** — 支援多 leader / 多 admin 架構，不再限制只能有一個 admin

---

## [1.1.5] — 2026-08-07

### 修正

- **Telegram UX 衛生** — 長訊息分段 + HTML 正規化 + 通報去噪 + 移除代號硬編碼（FR-1/2/3/5）

### 測試

- 795 passed

---

## [1.1.4] — 2026-08-07

### 修正

- **`wiki_query` 多櫃子掃描** — 知識庫查詢改為掃描所有已註冊 wiki 櫃子，不再只查第一個
- **not-found 診斷** — 查無結果時回傳診斷訊息（列出已掃描的櫃子和可用文件），協助使用者定位問題

---

## [1.1.3] — 2026-08-07

> 硬編碼治理 Wave 3（完結）：skip_resume 設定化 + template + validate + CI 掃描

### 新增

- **`skip_resume` 進設定檔** — 可在 team.yaml 的 instance 設定，role 自動預設（admin/manager 接續，worker fresh）
- **`validate` 語意檢查** — `scheduler.target` 和 `authority-matrix` 的 owner 必須是有效 instance 名稱
- **`preflight` / `degraded` 可觀測** — 啟動前檢查失敗不直接 crash，改為降級模式運行並通報
- **CI 字面值掃描** — 新增掃描腳本偵測殘留的硬編碼字串

### 修正

- **template 明示** — init 產生的範本加入明確佔位符和註解（移除 nana 專屬名稱）
- **`team_doc` 設定化** — TEAM.md 內容由 team.yaml 驅動，不再寫死

---

## [1.1.2] — 2026-08-06

> 硬編碼治理 Wave 2：決策系統去硬編碼

### 修正

- **決策系統去硬編碼** — 決策域（domain）和決策者由 `authority-matrix.yml` 推導，不再寫死 `cto-agent` / `ceo-agent`
- **移除 TEAM.md 硬塞 cto-agent** — 依 ADR-001 決策，本體（非啟動 agent）不在 TEAM.md 宣告
- **範本 authority-matrix 去 nana 名** — 改為中性佔位符 + 註解

---

## [1.1.1] — 2026-08-06

> 硬編碼治理 Wave 1

### 修正

- **火影字串移除** — 系統提詞中的火影角色名稱改為動態讀取 team.yaml description
- **`general_topic` 動態化** — 不再寫死 topic ID，從 team.yaml `channel.general_topic_id` 讀取
- **`agent.json` 動態生成** — `init` 時根據 instance 名稱和 role 自動生成，不硬編碼路徑

---

## [1.1.0] — 2026-08-06

> 重大功能：Group Topic Routing

### 新增

- **Group Topic Routing** — worker agent 的輸出訊息改發所屬 leader 的 topic（而非 General），實現訊息歸類
- **`_last_source` 追蹤** — 記錄每則訊息的來源 agent，支援追溯和 debug
- **失敗退 General 標記** — 當 leader topic 發送失敗時，自動 fallback 到 General Topic 並標記 `[fallback]`
- **P2P 升級優先 group leader** — 跨 agent 通訊升級時優先通知同 group 的 leader，不直上 admin

### 向後相容

- 未設定 topic 的 agent 行為不變（仍發 General）
- `TEAM.md` policy 改為 `once`（首次生成後不覆寫，保留手動修改）

### 測試

- 補完 FR-3/4/5 自動化測試

---

## [1.0.5] — 2026-08-06

### 新增

- **Topic 持久化** — `state/topics.json` 保存已建立的 Telegram topic 映射，重啟不遺失
- **BRAIN.md / CODE.md 鷹架** — `init` 時為每個 agent 產生 steering 文件骨架
- **結構清理** — 移除冗餘模板檔案，統一目錄命名

### 修正

- `__version__` 同步為 1.0.5（match pyproject.toml）

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

### 文件

- README 新增「產出結構」章節：`init` 鷹架 vs `team start` 自動產出，含各檔 policy、
  `state/` 8 個 DB 清單、`{home}/instances/{name}/.kiro/` 路徑、手改無效項

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
