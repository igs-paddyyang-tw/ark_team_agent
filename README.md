# ark-team-agent

> **把多個 Kiro CLI 開成一個會分工的團隊。**
> 你在 Telegram 講一句話，它決定誰該做、派下去、追蹤進度、把結果收回來給你。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.4.3-orange)](https://github.com/igs-paddyyang-tw/ark_team_agent/releases)
[![Tests](https://img.shields.io/badge/tests-1417%20passed-brightgreen)](#測試與品質)

**作者**：paddyyang（[@igs-paddyyang-tw](https://github.com/igs-paddyyang-tw)）
**規模**：52 模組 / 約 18,500 行 / 1417 個測試 / 6 個生產部署

---

## 這個套件解決什麼問題

一個 AI CLI（Kiro / Claude Code / Codex…）一次只能陪你做一件事。你要它同時盯著服務、寫規格、跑測試、產日報，就得開好幾個終端機，然後**自己當那個調度員** —— 記得誰在做什麼、把 A 的產出貼給 B、發現某個視窗已經卡死兩小時。

`ark-team-agent` 把那個調度員自動化：

| 你原本要做的事 | 套件怎麼處理 |
|---|---|
| 開 N 個終端機、記得誰在做什麼 | 一份 `team.yaml` 宣告編制，daemon 管生命週期 |
| 把 A 的產出貼給 B | MCP 工具 `send_to_instance` / `delegate_task`，agent 自己傳 |
| 盯著某個視窗有沒有卡死 | hang 偵測 + 崩潰重啟 + `degraded` 事件簿 |
| 切視窗看進度 | Telegram：每個 agent 一個 Topic，訊息自動歸類 |
| 記得昨天決定了什麼 | 知識庫 + 拍板紀錄 + 每日日報 |
| 擔心 API 費用爆掉 | `cost_guard` 日限額，逐 agent 可獨立設 |

**不需要寫程式。** 一份 YAML + 一個 5 行的 `start.py`。

### 適合 / 不適合

| ✅ 適合 | ❌ 不適合 |
|---|---|
| 長期常駐、有多個平行職能的工作（維運 + 開發 + 分析） | 一次性的單一任務（直接用 CLI 更快） |
| 想用手機（Telegram）指揮的場景 | 需要低延遲同步回應的互動 |
| 願意讓 agent 各自累積記憶與知識 | 要求每次都從乾淨狀態開始 |

---

## 30 秒理解架構

```
                    ┌──────────────── Telegram ────────────────┐
   你 ──私訊──────► │  入口 agent（意圖分類 / 路由）             │
                    │  Topic #2 CTO   Topic #3 CEO   Topic #4… │
                    └──────────────┬───────────────────────────┘
                                   │ python-telegram-bot
                    ┌──────────────▼───────────────────────────┐
                    │        ark-team-agent daemon             │
                    │  ┌────────┬─────────┬──────────┬──────┐  │
                    │  │ 生命週期│ 排程引擎 │ 拍板迴路 │ 成本 │  │
                    │  └────────┴─────────┴──────────┴──────┘  │
                    │        HTTP API :13030 ── Web Dashboard  │
                    └──────┬────────┬────────┬─────────┬───────┘
                           │        │        │         │  asyncio subprocess
                    ┌──────▼──┐ ┌───▼───┐ ┌──▼────┐ ┌──▼─────┐
                    │kiro-cli │ │kiro-  │ │kiro-  │ │kiro-cli│
                    │ admin   │ │cli PM │ │cli QA │ │ …      │
                    └────┬────┘ └───┬───┘ └───┬───┘ └───┬────┘
                         └──────────┴─── MCP ┴─────────┘
                              （agent 之間互相傳話、查知識庫）
```

**一句話**：daemon 用 `asyncio.create_subprocess_exec` 開 N 個 `kiro-cli` 子進程，每個有自己的工作目錄與人格檔；它們透過套件內建的 **MCP server** 互相通訊，透過 **Telegram** 跟你通訊。

---

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.4.3/ark_team_agent-1.4.3-py3-none-any.whl
```

**需求**：Python ≥ 3.11、[Kiro CLI](https://kiro.dev) 已安裝並在 `PATH`

> ⚠️ **本 repo 是 private** —— 上面的 URL 匿名抓會回 **404**（不是 403，所以看起來像不存在）。
> 需要憑證時用 API 端點：
>
> ```bash
> gh release download v1.3.2 --repo igs-paddyyang-tw/ark_team_agent -p '*.whl'   # 需 GH_TOKEN
> ```
>
> 或 `curl` + **API url** + `Accept: application/octet-stream`（缺這個 header 會拿到 JSON metadata，
> 而它會被存成 `.whl` → 安裝時報看不懂的錯）。**存檔要用原始檔名**，`uv` 會拒絕 `w.whl` 這種名字。

---

## 快速開始

```bash
# 1. 產生鷹架（team.yaml、steering、knowledge、skills…）
ark-team-agent init

# 2. 填 .env
echo "TELEGRAM_BOT_TOKEN=your-token" > .env

# 3. 改 team.yaml：填 group_id、allowed_users、定義你的 agent

# 4. 啟動
python start.py
```

**最小 `start.py`：**

```python
import asyncio
from pathlib import Path
from dotenv import load_dotenv
from ark_team_agent.team import run_team

load_dotenv()

if __name__ == "__main__":
    asyncio.run(run_team(Path("team.yaml")))
```

**最小 `team.yaml`：**

```yaml
channel:
  bot_token_env: TELEGRAM_BOT_TOKEN
  group_id: -100xxxxxxxxxx        # Telegram 群組（需開啟 Topics）
  general_topic_id: 1

access:
  mode: locked                    # locked=白名單 / group=群組訊息全放行
  allowed_users: [123456789]      # 你的 Telegram user ID（/start 會告訴你）

defaults:
  backend: kiro-cli
  model: auto

team_doc:                         # 🔴 建議設定，見下方「為什麼要設 team_doc」
  command_chain: "使用者 → admin → leader → worker"
  workflow: "leader(規格) → worker(實作) → qa(驗收) → leader(結案)"

instances:
  admin-agent:
    working_directory: .
    private_chat: 123456789        # 私訊入口（意圖分類 + 路由）
    role: admin
    description: "👑 管家 — 服務管理、團隊指揮"

  leader-agent:
    working_directory: agents/leader-agent
    role: leader
    topic_id: 2
    description: "🧠 軍師 — 需求分析、派工、驗收"

  coder-agent:
    working_directory: agents/coder-agent
    role: worker
    group: leader-agent            # 輸出發到 leader 的 topic
    topic_id: 3
    persistent: false              # lazy spawn：需要才啟動
    idle_timeout_minutes: 30       # 閒置回收
    description: "💻 工匠 — 全端開發"
```

---

## 核心概念

### ① Instance 與四層角色

每個 agent 是一個 **instance** —— 一個獨立的 `kiro-cli` 子進程 + 一個工作目錄 + 一份人格檔（`SOUL.md`）。

| Role | 定位 | 通訊權限 |
|------|------|---------|
| `admin` | 你的入口與最高管理 | 可發給**所有人** |
| `manager` | 跨組協調、方針 | 可發給所有 group 成員 |
| `leader` | 拆解需求、派工、驗收 | 可發給所有人（除 admin） |
| `worker` | 執行專業職能 | 可發給 leader 與其他 worker |

**支援多 admin / 多 leader**。角色只決定**通訊權限與預設行為**，不限制數量。

### ② Channel：Telegram 就是操作介面

- **私訊** → 入口 agent（`private_chat` 那個）做意圖分類，決定自己回還是轉派
- **群組 Topic** → 每個 agent 一個 Topic，訊息天然歸類，不會混在一起
- **`group` 欄位** → worker 的輸出發到所屬 leader 的 Topic（而不是自己的），維持組內對話連續
- **`@mention`** → `@coder 幫我改這個 API` 直送指定 agent，不經路由

### ③ MCP：agent 之間怎麼傳話

套件內建一個 **MCP server（stdio JSON-RPC，手寫、不依賴 mcp SDK）**，掛給每個 agent。**17 個工具**：

| 分類 | 工具 |
|------|------|
| 對你回話 | `reply` · `reply_file` · `reply_task_image` |
| 對內回報 | `log_to_leader` |
| 跨 agent | `send_to_instance` · `delegate_task` · `smart_delegate` · `broadcast_all` |
| 團隊狀態 | `query_team_status` |
| 任務板 | `create_task` · `update_task` · `list_tasks` |
| 知識庫 | `wiki_query` · `wiki_ingest` |
| 決策 | `decision_digest` · `decision_overturn` |
| 成本 | `record_spend` |

> **`reply()` 是對使用者的唯一出口** —— 中間過程走 `log_to_leader()`。
> 這條規則寫在產生的 steering 裡，agent 會遵守。

你也可以在 `team.yaml` 宣告**外部 MCP server**（`github` / `bigquery` / `fetch`…）並逐 agent 掛載：

```yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env: { GITHUB_TOKEN: "${GITHUB_TOKEN}" }

defaults:
  mcp: [fetch]            # 全體預設

instances:
  coder-agent:
    mcp: [github]         # 實際掛載 = (defaults.mcp ∪ mcp) − mcp_exclude
```

> 🔴 **agent 完全不繼承你 `~/.kiro/settings/mcp.json` 的設定** —— 那份只服務你自己的 kiro session。
> agent 要用什麼工具，必須在 `team.yaml` 明確宣告。

### ④ 生命週期：daemon 在背後做的事

| 機制 | 行為 |
|------|------|
| **7 種狀態** | `stopped` · `starting` · `running` · `crashed` · `paused` · `awaiting_reply` · `idle` |
| **崩潰重啟** | 時間窗計數 + 遞增冷卻，**會停手**（不無限重啟） |
| **hang 偵測** | `running` 太久沒動靜 → 通報；`awaiting_reply`／`idle` 不誤報 |
| **MCP 預檢** | 啟動前驗證 server 可執行、env 內的路徑存在 → 單一壞 server 不再擊倒整個 agent |
| **lazy spawn** | `persistent: false` 的 agent 需要才啟動，閒置 `idle_timeout_minutes` 後回收 |
| **就緒判定** | 認 `All tools are now trusted`，**不認** MCP 初始化進度行（那行在 server 都沒起來時就出現） |

> 💡 **`idle` 為什麼要獨立一個狀態**：`awaiting_reply`（有人在等我）與 `idle`（做完了沒待辦）
> 對 hang 偵測的意義相反。原本兩者混用，導致 agent 一回覆就關掉偵測 —— 真掛死幾小時沒人知道。

---

## 功能詳解

### 🎯 派工與任務板

```
你：「幫我把登入 API 加上 rate limit」
 ↓ 入口 agent 意圖分類 → delegate_task("leader-agent", …)
leader：拆解 → create_task(…) → delegate_task("coder-agent", 規格)
coder：實作 → update_task(status=review) → send_to_instance("qa-agent", …)
qa：驗收 → update_task(status=done) → log_to_leader(結果)
leader：reply("✅ 已完成，QA 通過")
```

任務板是 `tasks/board.json` + `tasks/items/`，可用 `list_tasks` 查、Telegram `/tasks` 看。

**`smart_delegate`** 會依 `description` 自動挑人，不用你記誰負責什麼。

### 🤝 P2P 與逾時處理

worker 之間可直接通訊（`p2p.enabled`，**預設關閉**）。可設 `max_rounds`（同一對 agent 最多 N 輪，
超過就升級到 General topic）、`daily_limit_per_agent`、`cc_leader`（每輪 CC 給 leader —— **預設關**，多 leader 團隊會變純噪音，要稽核軌跡再 opt-in）。

**`peer_reply_timeout_minutes`（預設 15）** —— A 請 B 做事後 N 分鐘沒回應，系統會主動 nudge A「B 沒回你，要重試還是改派」。**不會靜默卡住。**

### ⚖️ 拍板迴路（Decision Loop）

`config/authority-matrix.yml` 定義誰能決定什麼：

| 層級 | 行為 |
|------|------|
| **L1** | agent 自決 |
| **L2** | 依 domain 找到的決策者拍板（如 `tech→admin`、`product→leader`） |
| **L3** | 超過門檻（例如日預算的 20%）→ 升級給你，Telegram 送**拍板卡** |

- 決策者由矩陣**推導**，不寫死角色名
- 逾時未拍板 → 自動轉呈
- **每日日報 + 翻案按鈕**（24 小時窗口）—— `decision_digest` / `decision_overturn`

### ⏰ 排程引擎

`scheduler.yaml`，完整 cron（含日/月/星期）：

```yaml
jobs:
  - name: hourly-check
    cron: "0 9-21 * * *"
    target: admin-agent
    prompt: '回報團隊狀態，style="report"'

  - name: memory-housekeeping
    cron: "30 3 * * 0"
    target: _builtin:memory-consolidate
```

**3 個內建 job（`_builtin:` 前綴，確定式、無 LLM 成本）**：

| target | 做什麼 |
|--------|--------|
| `_builtin:spec-validator` | Code ↔ Spec 一致性驗證，產 drift report |
| `_builtin:memory-consolidate` | `memory/daily/*.md` 超過 14 天搬進 `archive/`、`steering/MEMORY.md` 的舊日期分節歸檔、`memory.md` 超標記 `degraded` |
| `_builtin:output-ttl` | 依分類 TTL 掃 `output/`，超期記 `degraded`（**只提醒不刪**） |

### 📚 知識庫

三層，權限不同：

| 層 | 路徑 | 規則 |
|----|------|------|
| 私有結構化 | `knowledge/{agent}/wiki/` | **唯讀** —— 只由 ingest 產出，禁止手寫 |
| 私有原始素材 | `knowledge/{agent}/raw/` | 只新增，不改既有檔 |
| 跨 agent 共用 | 不是檔案，是 `wiki_query` | 多櫃子掃描，查無會回診斷（不是空白） |

查詢順序：私有 → 共用 → 外部搜尋，**不可跳層**；查無就說不知道。
`knowledge_search_order` 可指定櫃子優先序。

> **架構前提：每個 agent 獨立運作與發展，共用的只有知識庫本身** ——
> 不靠 symlink、不複製檔案，靠 `wiki_query`。

### 💰 成本控制

```yaml
cost_guard:
  daily_limit_usd: 30.0
  warn_at_percentage: 80
  timezone: Asia/Taipei
  per_instance_limits:
    coder-agent: 10.0      # 超限只暫停這一個，不影響其他人
```

搭配**模型分工**（零程式，用既有的 per-instance `model`）：

```yaml
instances:
  entry-agent:
    model: auto                            # 入口每則訊息都醒 → 用快的
  architect-agent:
    model: claude-opus-4.6                 # 重推理才用強的
    model_failover: [claude-sonnet-4.6]    # 主模型不可用時降級
```

### 👥 多使用者隔離

```yaml
instances:
  admin-agent:
    multi_user: true
    max_user_sessions: 5
    session_idle_timeout_minutes: 30
```

per-user 的 session 與工作目錄（`<wd>/sessions/<uid>/`）、per-user 記憶檔。
LRU 回收，但**不會回收正在等人回話的 session**。

### 📱 Telegram UX

| 指令 | 用途 |
|------|------|
| `/start` | 系統介紹 + **你的 Telegram ID** + 授權狀態（未授權者也能查 ID 來申請） |
| `/status` | agent 分層狀態 + 主程式健康行（版本 / 運行數 / degraded） |
| `/tasks` | 任務派工狀況 |
| `/help` | 指令清單 + 完整成員（職責導向，取自 `description`） |
| `/restart` | 重啟 —— **先跳確認鍵**，按了才執行（限私聊） |
| `/allow` `/deny` | 動態授權（admin only，免重啟） |

還有：長訊息自動分段（4000 字）、ANSI 殘留清洗、ToolTracker 進度單行摘要、
`reply(template=...)` 具名模板（`news-daily` / `ops-report` / `status-report` / `task-dispatch` / `error-alert`）。

**HTML 安全**：agent 輸出一律經過處理才送出 ——
`style="chat"` 是 escape-first（Markdown → 平衡的 HTML），
`style="report"` 走**標籤白名單**（agent 可自己寫 `<b>`，但 `<260>`、`<=5%`、`a<b`
這類非標籤文字會被跳脫，且保證標籤平衡）。

### 🧩 內建 Skills（11 個）

`init` 會把這些放進每個 agent 的 `.kiro/skills/`：

`ark-grill-me`（拷問設計） · `ark-superpowers`（產 spec/design/plan） ·
`ark-spec-executor`（執行 plan） · `ark-code-spec-validator`（一致性驗證） ·
`ark-skill-creator` · `ark-wiki-engine` · `ark-md-report` · `ark-html-report` ·
`ark-news-daily` · `ark-agent-cli` · `ark-telegram-sender`

`kiro_files.skills.policy: sync` 可讓 skill 更新下發到既有 agent
（預設 `once` 只在目錄不存在時複製 —— 那會讓 skill 永久停在首次部署的版本）。

### 🔭 觀測

```bash
curl -s localhost:13030/api/health
```

```json
{"ok": true, "version": "1.3.2",
 "instances": {"running": 4, "alive": 11, "total": 11},
 "degraded": []}
```

- `alive` = `running` + `awaiting_reply` + `idle`；`running` 只算正在跑的
  → **`4/11` 通常是 lazy spawn 的正常狀態，不是故障**（看日誌的 `Idle eviction` 行確認）
- **`degraded[]` 是事件簿**：`mcp_startup_failed:*`、`restart_interrupted:*`、
  `deferred_message:*`、`memory_oversized:*`、`slow_startup:*`…
  出現不代表壞掉，代表**有事情你該知道**

另有 22 個 HTTP 端點與一個 Web Dashboard（`health_port + 300`）。

---

## 產出結構

分兩個時機：`init` 一次性鷹架、`team start` 每次啟動同步。

### `ark-team-agent init` —— 一次性（`if not exists`，可安全重跑）

```
{home}/
├── team.yaml · scheduler.yaml · README.md · .env.example · .gitignore
├── start-team.sh / .bat            # 含 restart.flag 自動重啟迴圈
├── .kiro/steering/                 # SOUL · BRAIN · CODE · MEMORY · TEAM · USER.md
├── config/authority-matrix.yml     # 決策權限矩陣 L1/L2/L3
├── knowledge/                      # schema/index/log + wiki + raw
├── docs/                           # specs designs plans reports one-pagers references
├── prompts/                        # 5 個派工模板
└── tasks/                          # board.json + items/

{working_directory}/                # 每個 agent 各一份
├── AGENTS.md                       # 給非 Kiro 工具（Claude Code / Codex）的入口導覽
├── knowledge/ · docs/ · memory/ · scripts/ · artifacts/ · output/
└── .kiro/  steering + agents/{name}.json + skills/(11)
```

> `--entry-name` 可改入口 agent 名（預設 `ark-agent`）。

### `team start` —— 每次啟動

| 產出 | policy | 說明 |
|------|--------|------|
| `.kiro/steering/TEAM.md` | **always** | 成員表以 `team.yaml` 為唯一真相 |
| `.kiro/settings/mcp.json` | always | MCP 掛載（+ wrapper 腳本讓 `env` 生效） |
| `SOUL·BRAIN·CODE·MEMORY·USER.md` | `once` | 刻意的 —— agent 可自行補充而不被覆寫 |
| `.kiro/agents/{name}.json` | `once` | |
| `state/*.db` · `topics.json` | 自動 | topic 持久化，重啟不產生孤兒 topic |

### ⚠️ 手改無效的檔案

- **`TEAM.md`** —— 改成員請改 `team.yaml` 後重啟
- **`mcp.json`** —— 改 MCP 請改 `team.yaml` 的 `mcp_servers`

> 🔴 **`policy: once` 的檔案不會因為升級套件而修好。** 凡是「升級後行為沒變」的怪事，
> 先查該檔的 policy 是不是 `once`。實測踩過：一份 `TEAM.md` 凍結三個月，
> 裡面有個早就被移除的幽靈成員，而 14 個 agent 每次對話都吃到它。

### 為什麼要設 `team_doc`

`TEAM.md` 的「指揮鏈」與「協作流程」兩段由 `team_doc` 產生。**未設就整段省略**（不是留空標題）。

> 🔴 **agent 讀不到 `team.yaml`** —— 它只讀 `TEAM.md` 的最終內容。
> 所以「缺了指揮鏈」這件事它**既不會察覺、也無法自己補**。
> 實測稽核六個部署，只有一個設了 —— 其餘團隊的 agent 從來不知道自己在鏈上的位置。
>
> 1.3.1 起產生器會在圖下附一行**實際編制**（如「admin×2、worker×7」，因為鏈是角色層級的簡化），
> 並在鏈與 `instances` 不一致時發 warning（幽靈角色 / 隱形角色）。

---

## 設定參考

### `team.yaml` 頂層

| 欄位 | 說明 |
|------|------|
| `instances` | 成員定義（見下表） |
| `channel` | Telegram：`bot_token_env` · `group_id` · `general_topic_id` |
| `access` | `mode`（`locked` / `group`）· `allowed_users` |
| `defaults` | 全體預設：`backend` · `model` · `mcp` · `skip_resume`… |
| `team_doc` | 指揮鏈與協作流程（**建議設定**） |
| `mcp_servers` | 外部 MCP server 宣告 |
| `communication` | `peer_reply_timeout_minutes` · `dispatch_context_ttl_minutes` · `tool_detail` · `p2p.*` |
| `cost_guard` | `daily_limit_usd` · `warn_at_percentage` · `per_instance_limits` |
| `hang_detector` | 掛起偵測 |
| `startup` | 啟動策略 |
| `kiro_files` | 各產出檔的 policy（`always` / `once` / `skip`）與 `skills` 下發 |
| `knowledge_search_order` | 知識庫櫃子優先序 |
| `private_chat_routing` | 私訊路由模式（`ark-agent` / `keyword` / `ceo-direct`） |
| `project_roots` · `health_port` | 專案根、API port |

### `instances.<name>`

| 欄位 | 說明 |
|------|------|
| `working_directory` | **必填** —— agent 的家 |
| `role` | `admin` / `manager` / `leader` / `worker` |
| `description` · `display_name` · `tags` | 職責描述（`smart_delegate` 與 `/help` 會用） |
| `topic_id` · `general_topic` · `private_chat` | Telegram 路由 |
| `group` | worker 指向所屬 leader |
| `backend` · `model` · `model_failover` | 後端與模型 |
| `skip_resume` | `true`=開新對話 / `false`=`--resume` 接續；未設依 role |
| `persistent` · `idle_timeout_minutes` · `auto_start` | lazy spawn 與回收 |
| `mcp` · `mcp_exclude` | MCP 掛載 |
| `task_prefix` · `task_suffix` | 自動包夾訊息（如「只輸出結論、≤3000 字」） |
| `system_prompt` · `template` | 人格與範本 |
| `restart_policy` · `startup_timeout_ms` · `cost_guard` · `log_level` | 可靠性與成本 |
| `multi_user` · `max_user_sessions` · `session_idle_timeout_minutes` | 多使用者隔離 |

> 💡 **未知欄位會發 warning 並給建議**（1.2.24+）。實測有個部署把 `skip_resume` 寫成
> 不存在的 `resume_session`，**9 個 instance 全都寫錯、7 個行為與意圖相反、零訊號** ——
> 因為未知欄位原本被靜默丟棄。建議排序用詞素交集而非字面相似度
> （`difflib` 對 `resume_session` 的首選是 `max_user_sessions`，那是錯的）。

---

## CLI

```bash
ark-team-agent init [--entry-name NAME]   # 產生鷹架
ark-team-agent team start | stop | restart | status
ark-team-agent ls                          # 活躍進程
ark-team-agent sessions | attach <name>    # 對話 session
ark-team-agent send <name> <msg>           # 直接發訊息給某個 agent
ark-team-agent kiro status | sync | diff | refresh   # .kiro 產出檢查
ark-team-agent api | webbot | scheduler    # 單獨啟動子系統
```

---

## 運維

### 升級

```bash
pip install --force-reinstall --no-deps <wheel 或 Release URL>
python -c "import ark_team_agent as m; print(m.__version__, m.__file__)"
systemctl --user restart <service>
curl -s localhost:13030/api/health          # 🔴 看版本號
```

### 🔴 三條驗證鐵律（每條都是實測踩出來的）

| # | 規則 | 為什麼 |
|---|------|--------|
| 1 | **重啟後看版本號，不是看 `systemctl is-active`** | 實測發生過 `active` 但跑的是舊碼（uptime 17 小時、版本停在上一版）。**版本號是唯一可靠的證據** |
| 2 | **editable 與 wheel 不能混** | 先 `pip uninstall` 並確認 `import` 失敗，再裝 wheel。`.pth` 會把舊 `src/` 插在 `sys.path` 最前 → 「裝了 wheel 但讀的還是 src」**而驗證會通過** |
| 3 | **`degraded` 空 ≠ 沒問題** | 它記的是「系統知道的異常」。有些缺陷不產生任何事件（例如訊息送達但內容被刪、agent 讀到過時的成員表） |

---

## 測試與品質

```
1417 passed · 6 skipped
```

- 純 `pytest`（`asyncio_mode=auto`），無外部服務依賴
- **掃描式守門測試** —— 例如「所有 `team.yaml` 不得有未知欄位」、「`scheduler` 不得寫死 instance 名」。
  用 `ast` 排除 docstring 與註解（說明文件常示範「不該這樣寫」）
- 測試分兩層：套件邏輯測試（隨套件走）vs 部署驗證測試（留在各部署）

---

## 設計原則

這個套件修過的缺陷有**固定形狀**。這些原則就是從那些缺陷長出來的：

**① 「宣告了但沒接上」是最常見的缺陷類型。**
設定欄位存在、文件寫著、實作卻是空的或早被移除。比「沒有這個設定」更糟 ——
讀設定的人會在錯誤假設上做決定。所以現在**未知欄位會警告、死設定會被測試擋下**。

**② 假綠燈比壞掉更危險。**
壞掉你會發現；掃不到東西卻回報 ✅ 的檢查，你會信它。
所以守門腳本要驗**實際值**，不只驗寫法。

**③ 靜默失效要當成缺陷本體，不是副作用。**
不拋例外、不寫 log、只讓行為安靜地錯 —— 這類問題可以存在好幾個月。
`degraded` 事件簿、未知欄位警告、MCP 預檢，都是為了把靜默的東西講出來。

**④ agent 只讀最終 context，不知道設定檔存在。**
「機制存在但 agent 不知道」等於沒有。所以該讓人設定的事，位置是**範本註解與專案文件**，
不是 agent 的 always-on context —— 對它提示無法行動的事只是雜訊。

**⑤ 部分結果比沒結果好，但要講明白。**
逾時就用已完成的部分收斂，並在答覆裡說清楚誰沒回。不要讓人等到硬超時。

---
## 部署案例

| 專案 | 編制 | 說明 |
|------|------|------|
| [paddy-team-agent](https://github.com/igs-paddyyang-tw/paddy-team-agent) | 5 agents | **本套件的開發源**（editable）+ Paddy 個人 AI 團隊 |
| [nana-team-agent](https://github.com/igs-paddyyang-tw/nana-team-agent) | 11 agents | 小娜 AI Team（扁平 topic 架構，每 agent 一個 Topic）|
| aiops-team-agent | 9 agents | 維運團隊（事故指揮 / SLO / 觀測 / 資安 / 自動化 / MLOps）|
| director-team-agent | 7 agents | 情報蒸餾團隊（產品 / 市場 / 技術 / 知識管家）|
| slot-team-agent | 13 agents | 老虎機 QA 諮詢（**三層 group 架構**：總機 → 3 組 leader → 組內職人）|
| fish-team-agent | 14 agents | 捕魚機團隊（slot 的前身）|

> **三種架構模型都在生產跑**：扁平 topic（nana）、純私訊無群組（paddy）、三層 group（slot / fish）。
> 差異只在 `team.yaml` —— 套件本身不假設你的組織形狀。

---

## 版本歷史

完整變更見 [CHANGELOG.md](CHANGELOG.md)。

| 版本 | 日期 | 摘要 |
|------|------|------|
| **1.4.3** | 2026-08-18 | **群組也解析 `@mention`** —— 在此之前只有私訊路徑會（`ChatRouter`），群組走 `planner` 而它**完全不看 `@`**。所以在 General 打 `@qa` 是靠入口 agent 的 LLM 讀懂那個 `@` 再派工，代價是順序倒置（agent 的 ack 產生於派工之後，被派工者可能秒答）、多一次 LLM、可能判錯人、多喚醒一個 agent。`@all`／解析失敗／多目標**刻意交回 planner**（不靜默丟訊息）。守門測試 10 個 |
| **1.4.2** | 2026-08-18 | **私訊問 → 回私訊** —— `reply_to_origin` 現在涵蓋兩種場景。1.4.0/1.4.1 只處理群組 topic，而私訊 `@mention` 別人時答案會跑到群組的某個 topic（同一類問題：回覆去「agent 的家」而不是「你問的地方」）。`_origin_topic` 的值從 `int` 改成 `(kind, dest)` —— 一個概念一個資料結構，不開第二個 dict。⚠️ 私訊目的地記的是使用者那個 `chat_id`，不是 agent 自己的 `private_chat` 設定（被派工的 worker 通常沒有）|
| **1.4.1** | 2026-08-18 | 🔴 **訂正 1.4.0：origin 回覆路由改為 opt-in**（`communication.reply_to_origin`，**預設 false**）。1.4.0 讓它無條件生效是錯的 —— 它改變「訊息出現在哪裡」，屬於各團隊的動線習慣，不該由套件替它們決定；而實測 aiops(6)／director(4)／slot(8) 都用 `group` 把 worker 輸出收攏到組 leader 的 topic，1.4.0 在互動情境下把它拿掉且**沒有開關可關**。建議：扁平 topic／純私訊架構開，三層 group 架構維持關 |
| **1.4.0** | 2026-08-18 | 🎯 **回覆回到「使用者發問的地方」** —— 原本一律回 agent 的歸屬 topic（自己的或 leader 的），使用者在 General 問完卻要切 topic 找答案。`_reply_channel` 加 `origin` 態，並讓 origin **沿派工鏈傳遞**（少了這步解不了問題：被派工的 worker 從沒收到過使用者訊息）；排程／系統來源則清掉 origin，避免日報污染對話 topic。另新增 **`_builtin:ingest-queue`**：機械掃出「檔名尚未出現在 index.md」的原始檔，把清單當成任務內容發出去 —— 把「請沉澱知識」這種開放式任務變成有明確輸入的工作。守門測試 25 個 |
| **1.3.3** | 2026-08-18 | 🔴 **重啟有四套機制，看不出哪套在生效** —— 回報說「TG 重啟壞了，flag 沒人讀」；實測重啟是好的（靠 SIGTERM + systemd，PID 會換），但程式碼裡看得到三個 flag 讀取者、而它們在現行部署都沒在跑，磁碟還躺著幾天前的殘留 flag —— 任何人都會誤判。新增 `restart.py` 單一入口：偵測 supervisor（`INVOCATION_ID` / `ARK_TEAM_AGENT_WRAPPER`）、**flag 只在真的有消費者時才寫**、啟動與收尾清殘留、**沒有 supervisor 時不再假裝會重啟**。順帶消除 `telegram.py` / `api.py` 兩份不一致的重複實作。守門測試 15 個 |
| **1.3.2** | 2026-08-18 | 🔴 **`style="report"` 是 raw passthrough** —— agent 回覆裡的 `<260>` 被 TG 當未知標籤 → 整則 400；而 fallback 的 `_strip_html()` 會把那段**直接刪掉**，於是訊息送達、數值卻從報告裡無聲消失。改用白名單 `_sanitize_tg_html()`（TG 支援的 16 標籤留、其他 `<>&` 跳脫），並驗屬性（`if x <b then y> 0` 的 `b` 是合法標籤名，只驗名會把句子吃成標籤）＋保證結構平衡。`style="chat"` 未動。守門測試 38 個 |
| **1.3.1** | 2026-08-17 | 🔴 **三個部署共 21 個 agent 全程沒有日誌** —— `run_team()`（README 教大家用的公開入口）從不初始化 logging，只有走 CLI 的 nana 有 INFO；實測近一小時 nana 238 行、其餘三個 **0 行**。另含 `TEAM.md` 指揮鏈附註實際編制（nana 有兩個 admin，圖上看不出來）、`scheduler` drift 目標不再寫死 `cto-agent`、`init` 新增 `--entry-name`，以及修掉 `init` 在**全新環境必定失敗**的 `UnboundLocalError`（1.3.0 與更早皆有，因既有部署不再跑 init 而未被發現）|
| **1.3.0** | 2026-08-17 | 🚚 **開發環境遷移：nana-team-agent → paddy-team-agent**。功能與 1.2.26 完全相同（wheel 187/187 逐檔一致，唯一差異是 `__version__` 那行）—— minor 跳號唯一的意義是標記 build 來源已遷移。Release repo 與 `pip install` URL 機制不變 |
| **1.2.26** | 2026-08-17 | 🔴 **aiops / director 的緊急通知與 L3 拍板卡從來沒送出過** —— `notify_paddy()` 寫死 `_private_chat_map.get("ark-agent")`，而它們的入口是 `admin-agent` / `tech-agent`。改為依 role 推導。另含 CI 補 `ark-agent` 盲區（噴 24 處）與測試分兩層（`tests/deployment/` 不隨套件搬移）|
| **1.2.25** | 2026-08-17 | MCP 進度行不再被當成「就緒」—— `ctrl-c to start chatting now` 是`⠋ 0 of 1 mcp servers initialized…` 的尾巴，實測它在 2.4s、一個 server 都沒起來時就印出，於是「2.0s 就緒」其實是「剛開始初始化」。改認 `All tools are now trusted`（實機 2.0s→4.0s）。**本版為套件開發搬移至 paddy 的基準線** |
| **1.2.24** | 2026-08-17 | 未知 instance 欄位不再被靜默丟棄 —— aiops 9 個 instance 寫了不存在的 `resume_session`，7 個 agent 的行為與設定意圖相反且毫無訊號。建議排序依詞素交集而非字面相似度（`difflib` 會給錯答案）|
| **1.2.23** | 2026-08-17 | 「宣告了沒接上」四連修：`private_chat_routing` 整段是死設定（`TeamConfig` 沒此欄位，keyword 路由從未生效）、`route_by_keyword` 的寫死預設、決策路徑用名字比對而非 `working_directory`、`build_release --release` 從未成功執行過 |
| **1.2.22** | 2026-08-17 | 清掉三個假警報並修一個實質缺陷：`mcp_undeclared_moved` 殘骸誤判、`slow_startup` 閾值 0.5→0.75、TEAM.md 不再謊報自己的 policy；**MCP 啟動失敗不再被認列為就緒**（原本 status=RUNNING 後 4–30 秒才 rc=3 死掉，期間訊息全丟）|
| **1.2.21** | 2026-08-14 | MCP 宣告機制修正：宣告的 `fetch` 不再被硬編預設靜默覆蓋、兩份 fetch 並存修正、預檢擴充到 env 內的路徑（憑證不存在 → server 起得來但所有呼叫失敗）|
| **1.2.20** | 2026-08-14 | 多使用者 session 隔離治理（分隔符 `#`→`~`、`max_user_sessions` 空殼補實作、per-user 工作區隔離）+ N5 記憶治理啟用（`steering/MEMORY.md` 日期分節歸檔）|
| **1.2.19** | 2026-08-14 | 各 agent 根目錄產生 `AGENTS.md` 導覽檔（非 Kiro 工具不讀 `.kiro/steering/`，進目錄後沒入口）+ 工作區缺漏目錄自動補齊 |
| **1.2.18** | 2026-08-14 | 移除 `steering/AGENTS.md` symlink 共用與失效的知識庫掛載 —— 每個 agent 獨立運作，共用的只有知識庫本身 |
| **1.2.17** | 2026-08-14 | 補 1.2.16 漏掉的四處 `idle`（`alive` 會隨 agent 完工而下降）|
| **1.2.16** | 2026-08-14 | 新增 `IDLE` 狀態：區分「有人在等」與「做完了沒待辦」——原本 agent 一回覆就關掉 hang 偵測，真掛死 4-6 小時無人知 |
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
