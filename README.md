# ark-team-agent

> Multi AI Agent 團隊管理框架，基於 [Kiro CLI](https://kiro.dev) 後端，支援 Telegram Bot 互動、跨 Agent 協作與自主拍板決策。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.2.15-orange)](https://github.com/igs-paddyyang-tw/ark_team_agent/releases)

**作者**：paddyyang（[@igs-paddyyang-tw](https://github.com/igs-paddyyang-tw)）

---

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.2.15/ark_team_agent-1.2.15-py3-none-any.whl
```

**需求**：Python ≥ 3.11、[Kiro CLI](https://kiro.dev) 已安裝

---

## 快速開始

```bash
# 1. 安裝套件
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.2.15/ark_team_agent-1.2.15-py3-none-any.whl

# 2. 初始化專案結構
ark-team-agent init

# 3. 準備 team.yaml + .env
cp team.yaml.example team.yaml   # 或自行建立
echo "TELEGRAM_BOT_TOKEN=your-token" > .env

# 4. 啟動
python start.py
```

**最簡 start.py：**

```python
import asyncio
from pathlib import Path
from dotenv import load_dotenv
from ark_team_agent.team import run_team

load_dotenv()

if __name__ == "__main__":
    asyncio.run(run_team(Path("team.yaml")))
```

---

## 產出結構

分兩個時機產生檔案：`init` 一次性鷹架、`team start` 每次啟動自動產生。

### `ark-team-agent init` — 一次性鷹架（`if not exists`，可安全重跑）

```
{home}/
├── team.yaml · scheduler.yaml · README.md · .env.example · .gitignore
├── start-team.sh / .bat        # 含 restart.flag 自動重啟迴圈
├── .kiro/steering/             # SOUL · BRAIN · CODE · MEMORY · TEAM · USER.md
├── config/authority-matrix.yml # 決策權限矩陣 L1/L2/L3
├── knowledge/                  # 完整子結構（schema/index/log + wiki + raw）
├── docs/                       # specs designs plans reports one-pagers references
├── prompts/                    # 5 派工模板
└── tasks/                      # board.json + items/

{working_directory}/            # 每個 agent（team.yaml instances 迴圈）
├── knowledge/ · docs/ · memory/
└── .kiro/                      # steering(SOUL/BRAIN/CODE/MEMORY/USER) + agents/{name}.json + skills/(6)
```

內建 6 skills：`ark-grill-me · ark-superpowers · ark-spec-executor · ark-code-spec-validator · ark-skill-creator · ark-wiki-engine`

### `team start` — 每次啟動自動產生／覆寫

> 啟動時 backend 寫的 `.kiro/` 檔位於 **`{home}/instances/{name}/.kiro/`**（kiro-cli 子進程 cwd）。

| 產出 | policy |
|------|--------|
| `.kiro/steering/TEAM.md` | **always（每次覆寫）** |
| `.kiro/steering/SOUL·BRAIN·CODE·MEMORY·USER.md` | once |
| `.kiro/settings/mcp.json`（+ wrapper 腳本） | always（可設 skip/once） |
| `.kiro/agents/{name}.json` | once（可設 skip） |
| `.kiro/.meta.json` · `instances/{name}/.mcp_hash` · `team.pid` | 每次 |
| `state/topics.json` | 首次建，之後 restore |
| `state/*.db` | 自動建立 |

`state/` 下 8 個檔：`conversations · message_queue · message_overflow · events · decisions · costs · sessions`（`.db`）+ `heartbeat`

### ⚠️ 手改無效（每次啟動被覆寫）

- `TEAM.md`：改成員請改 `team.yaml` 後重啟
- `mcp.json`：改 MCP 注入邏輯在 `backend.py`
- `AGENTS.md` 自 v1.0.4 起**不再由啟動同步**，改由 `TEAM.md` 承接

---

## 功能特色

### 多 Agent 協調
- `team.yaml` 定義 agent，4 層角色（admin / manager / leader / worker）
- 支援多 leader / 多 admin 架構（v1.1.6+）
- `skip_resume` 可設定：admin/manager 自動接續，worker 預設 fresh start（v1.1.3+）

### Group Topic Routing（v1.1.0+）
- Worker 輸出改發所屬 leader 的 Telegram topic（訊息歸類）
- 失敗自動 fallback 到 General Topic 並標記
- P2P 升級優先通知同 group leader

### Telegram Bot
- Forum Topic 路由、HTML 通報（支援粗體/斜體/刪除線/引用 v1.1.10+）
- ToolTracker 人性化版面（v1.1.7+）
- InlineKeyboard、重啟確認按鈕（v1.2.5+）
- 長訊息自動分段（4000 字上限，v1.1.5+）、出口自動清 ANSI 殘留（v1.2.3+）
- Reply Template：`reply(text, template=...)` 具名模板 —— `news-daily` / `status-report` / `ops-report` / `task-dispatch` / `error-alert`（v1.2.0+）

#### 指令（v1.2.5 起五指令目標分明）

| 指令 | 用途 |
|------|------|
| `/start` | 系統介紹 + **你的 Telegram ID** + 授權狀態 + 如何對話（未授權者亦可查 ID 申請）|
| `/status` | agent 分層狀態 + 主程式健康行（版本 / 運行數 / degraded）|
| `/tasks` | 任務派工狀況 |
| `/restart` | 重啟服務 —— **先跳確認鍵，按「確定重啟」才執行**（限私聊）|
| `/help` | 細部介紹：指令清單 + 完整成員（職責導向，取自 `description`）+ `@mention` 語法 |
| `/allow` `/deny` | 動態授權管理（admin only，免重啟）|

`@mention` 派工：`@{agent} {需求}` → 直接發給該 agent。

### 拍板迴路（Decision Loop）
- L1 自決 / L2 CEO‑CTO 拍板 / L3 升級
- ark-agent fallback 拍板（有前例自決、無前例升 Paddy）
- 每日日報 + 翻案按鈕（24h 窗口）
- `authority-matrix.yml` 動態推導決策者（v1.1.2+）

### MCP 通訊
- 15+ tools，跨 Agent P2P / broadcast / wiki
- `send_to_instance` peer-reply timeout 語意回傳（v1.1.13+）
- `wiki_query` 多櫃子掃描 + not-found 診斷（v1.1.4+）

### 生命週期管理
- 崩潰重啟、掛起偵測（hang_detector）、成本控制（cost_guard）
- 排程引擎（完整 cron 支援，含日/月/星期 v1.0.4+）
- Topic 持久化（state/topics.json，重啟不遺失 v1.0.5+）
- Preflight / degraded 模式（v1.1.3+）

### 安全性
- `reply_file` 路徑白名單（v1.0.4+）
- `wiki_ingest` 路徑封閉檢查（v1.0.4+）
- 硬編碼治理 3 波（v1.1.1~v1.1.3）：所有系統提詞和設定動態化

---

## 設定速查（v1.1~v1.2 新增）

### `team.yaml` → `communication`

| 設定 | 預設 | 說明 |
|------|------|------|
| `peer_reply_timeout_minutes` | `15` | A 請 B 後 N 分無回覆 → 主動 nudge A 重試/改派（`0`=關）v1.1.13+ |
| `dispatch_context_ttl_minutes` | `30` | 派工脈絡保留時間；使用者續問時注入「你上一步請 X 處理 Y」（`0`=關）v1.2.2+ |
| `tool_detail` | `false` | ToolTracker 顯示模式：`false`=人性化單行、`true`=完整工具序列（維運排查）v1.1.9+ |
| `p2p.*` | — | worker↔worker 直接通訊策略（`enabled` / `max_rounds` / `cc_leader` / `emergency_mode`）|

### `team.yaml` → 頂層 / `instances.<name>`

| 設定 | 層級 | 說明 |
|------|------|------|
| `knowledge_search_order` | 頂層 | 知識庫櫃子搜尋優先序，如 `[shared, hoyeah]`；未設＝字母序 v1.2.0+ |
| `task_prefix` / `task_suffix` | instance | 送往此 agent 的訊息自動包夾（如「只輸出結論、≤3000 字」）；空＝no-op v1.2.0+ |
| `group` | instance (worker) | 指向所屬 leader，輸出改發該 leader 的 topic v1.1.0+ |
| `skip_resume` | instance | `true`=開新對話、`false`=`--resume` 接續；未設依 role（admin/manager 接續）v1.1.3+ |
| `multi_user` | instance (admin) | per-user session 隔離（`max_user_sessions` / `session_idle_timeout_minutes`）|
| `cost_guard.per_instance_limits` | 頂層 | 各 agent 獨立日預算，超限只暫停該 agent |

### 模型分工（成本優化 pattern，零程式）

借鏡「快模型路由／強模型幹活」的做法 —— 用既有的 per-instance `model` 設定即可，不需額外 LLM router：

```yaml
instances:
  entry-agent:            # 入口／路由：判斷意圖、分流，工作輕
    model: auto           # 或指定較快/較便宜的模型
  architect-agent:        # 重推理：架構設計、可行性評估
    model: claude-opus-4.6
    model_failover: [claude-sonnet-4.6]   # 主模型不可用時降級
```

入口 agent 每則訊息都會被喚醒，用快模型可明顯降低成本；重推理 worker 才用強模型。

### `scheduler.yaml` 內建 job

| `target` | 說明 |
|------|------|
| `_builtin:spec-validator` | Code ↔ Spec 一致性驗證（drift report）|
| `_builtin:output-ttl` | Output 超期提醒：依 BRAIN.md 分類 TTL 掃 `output/`，超期記 `degraded`（**只提醒不刪**）v1.2.7+ |
| `_builtin:memory-consolidate` | 記憶治理：`memory/daily/*.md` >14 天搬 `archive/`、`memory.md` 超標記 `degraded`（純機械、無 LLM、非破壞性）v1.2.4+ |

### 健康端點

`GET /api/health` → `instances: {running, alive, total}`（`alive`＝running + awaiting_reply，v1.2.0+）、`degraded[]`（降級/中斷事件，含 `deferred_message:*`、`restart_interrupted:*`、`memory_oversized:*`）。

---

## 部署案例

| 專案 | 版本 | 說明 |
|------|------|------|
| [nana-team-agent](https://github.com/igs-paddyyang-tw/nana-team-agent) | editable（框架源） | 小娜 AI Team（10 agents）|
| [paddy-team-agent](https://github.com/igs-paddyyang-tw/paddy-team-agent) | v1.0.1 | Paddy 個人 AI 團隊（5 agents）|
| [fish-team-agent](https://github.com/igs-paddyyang/fish_team_agent) | v1.0.5 | 捕魚遊戲團隊（10 agents）|

---

## 版本歷史

完整變更見 [CHANGELOG.md](CHANGELOG.md)。

| 版本 | 日期 | 摘要 |
|------|------|------|
| **1.2.15** | 2026-08-13 | 修 1.2.14 的 `mcp_undeclared_moved` 假警報沒修乾淨（只分類本次搬移 → 舊 `_disabled` 條目永遠被誤報）|
| **1.2.14** | 2026-08-13 | 崩潰迴圈治理（`max_retries` 被活動偵測歸零 → 改時間窗計數 + 遞增冷卻 + 會停手）、MCP 啟動觀測（`rc=3` 正確歸因為 MCP 失敗、外部全域 server 落差可見化）、修 1.2.13 的 `mcp_undeclared_moved` 假警報 |
| **1.2.13** | 2026-08-13 | MCP 設定治理：啟動前預檢（單一壞 server 不再擊倒 agent）、`team.yaml` 宣告 `mcp_servers` / `defaults.mcp` / `mcp_exclude`、`env` 經 wrapper 生效、未宣告者移至 `_disabled` |
| **1.2.12** | 2026-08-13 | 內建 skill 版控補齊（範本未追蹤 29→0）、`skills_policy: sync` 下發、對齊 canonical 庫（metadata 11/11）、打包前清 `build/` + wheel skill 集合驗證；併入原 1.2.11 崩潰鑑識 |
| 1.2.10 | 2026-08-12 | templates/skills 新增 html-report + md-report、移除 decision-digest + policy-translate |
| 1.2.9 | 2026-08-11 | 修 v1.2.7 引入的 UnboundLocalError 生產回歸 + log_to_leader 代號 + 4 端點補實作 |
| 1.2.8 | 2026-08-11 | code-spec-validator 假警報 57%→0、API 端點參考文件（45 端點）|
| **1.2.7** | 2026-08-11 | 取經整合：CI 反模式掃描、`_builtin:output-ttl`、Skill 自薦提示、chat trace；修 ToolTracker 步數低報 |
| **1.2.6** | 2026-08-10 | 修專案根定位缺陷（pip 環境失效）：`paths.py` 哨兵搜尋、`task_screenshot.py` 入套件、scheduler `project_root` 注入 |
| **1.2.5** | 2026-08-10 | TG 五指令目標明確化（/help 新增、/restart 確認鍵、/start 顯示 ID + 授權狀態）|
| **1.2.4** | 2026-08-10 | N5-B1 內建記憶治理 `_builtin:memory-consolidate`（機械歸檔，無 LLM）|
| 1.2.3 | 2026-08-10 | 修 TG 出口 ANSI 殘留（reply() 路徑 _strip_ansi）|
| 1.2.2 | 2026-08-10 | N4 Session 派工脈絡（多輪續問注入上一步派工）|
| 1.2.1 | 2026-08-10 | 修 /start Web Dashboard URL（寫死 23030 → health_port+300）|
| 1.2.0 | 2026-08-10 | 整合升級：task_prefix/suffix、knowledge_search_order、health alive、Reply Template |
| **1.1.13** | 2026-08-10 | `send_to_instance` peer-reply timeout 語意優化 |
| 1.1.12 | 2026-08-10 | 治本 agent 程式碼誤判 + 生成失敗/崩潰觀測 |
| 1.1.11 | 2026-08-09 | 生成失敗誤判 auth + awaiting_reply 卡住修復 |
| 1.1.10 | 2026-08-09 | `_md_to_html` 擴充斜體/刪除線/引用 |
| 1.1.9 | 2026-08-08 | 多組架構訊息路由缺陷修復（fish 實戰回報）|
| 1.1.8 | 2026-08-08 | 重啟自動觸發：多 leader 不挑第一個 + 靜默重啟 |
| 1.1.7 | 2026-08-08 | ToolTracker 版面人性化（單行階段摘要）|
| 1.1.6 | 2026-08-07 | P2P 通報改發 General + 多 leader/admin 架構 |
| 1.1.5 | 2026-08-07 | Telegram UX：分段/HTML正規化/通報衛生 |
| 1.1.4 | 2026-08-07 | `wiki_query` 多櫃子掃描 + not-found 診斷 |
| 1.1.3 | 2026-08-07 | 硬編碼治理 Wave 3：skip_resume/template/validate/CI |
| 1.1.2 | 2026-08-06 | 硬編碼治理 Wave 2：決策系統去硬編碼 |
| 1.1.1 | 2026-08-06 | 硬編碼治理 Wave 1：火影/general_topic/agent.json |
| 1.1.0 | 2026-08-06 | Group Topic Routing + TEAM.md policy once |
| 1.0.5 | 2026-08-06 | Topic 持久化 + BRAIN/CODE 鷹架 |
| 1.0.4 | 2026-08-05 | 安全性：路徑白名單 + 封閉檢查；cron 完整支援 |
| 1.0.3 | 2026-07-31 | startup 通報去重 |
| 1.0.2 | 2026-07-31 | `chat_router` 動態找 leader/admin |
| 1.0.1 | 2026-07-31 | wiki_ingest/scheduler/LLM retry 等 7 項修正 |
| 1.0.0 | 2026-07-30 | 初始版本：Decision Loop、TG UX、8 種意圖分類 |

---

## 版權

Copyright (c) 2026 paddyyang — [MIT License](LICENSE)
