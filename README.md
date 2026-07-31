# ark-team-agent

> Multi AI Agent 團隊管理框架，基於 [Kiro CLI](https://kiro.dev) 後端，支援 Telegram Bot 互動、跨 Agent 協作與自主拍板決策。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-orange)](https://github.com/igs-paddyyang-tw/ark_team_agent/releases)
[![Tests](https://img.shields.io/badge/tests-21%20passed-brightgreen)](https://github.com/igs-paddyyang-tw/nana-team-agent)

---

## 作者

| 項目 | 說明 |
|------|------|
| **作者** | paddyyang |
| **GitHub** | [@igs-paddyyang-tw](https://github.com/igs-paddyyang-tw) |
| **開發環境** | Windows 11 + Kiro CLI + VS Code |
| **原始碼倉庫** | [nana-team-agent](https://github.com/igs-paddyyang-tw/nana-team-agent)（私有，實際開發在此） |
| **Release 倉庫** | [ark_team_agent](https://github.com/igs-paddyyang-tw/ark_team_agent)（本倉庫，公開發版用） |

---

## 驗證環境

| 項目 | 版本 / 說明 |
|------|------------|
| **Python** | 3.11 / 3.12（CI 矩陣） |
| **Kiro CLI** | 最新版（後端必須已安裝且可執行 `kiro-cli chat`） |
| **作業系統** | Windows 11（主開發）、Ubuntu 22.04（Linux 部署驗證） |
| **原始碼版本** | [nana-team-agent](https://github.com/igs-paddyyang-tw/nana-team-agent) `main` 分支 |
| **單元測試** | 21 個測試通過（含 Decision Loop 全鏈） |
| **Telegram Bot API** | python-telegram-bot 21.10 |

---

## 安裝

### 從 GitHub Releases 安裝（推薦）

```bash
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.0.0/ark_team_agent-1.0.0-py3-none-any.whl
```

### 核心依賴

```
Python >= 3.11
pyyaml >= 6.0
python-telegram-bot == 21.10
fastapi >= 0.115.0
uvicorn >= 0.30.0
apscheduler >= 3.10.0
python-dotenv >= 1.0.0
pydantic >= 2.0.0
```

> **注意**：需要預先安裝 [Kiro CLI](https://kiro.dev) 並完成登入。

---

## 快速開始

```bash
# 1. 安裝套件
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.0.0/ark_team_agent-1.0.0-py3-none-any.whl

# 2. 產生預設設定
ark-team-agent init

# 3. 設定環境變數
export TELEGRAM_BOT_TOKEN="your-bot-token"

# 4. 編輯 team.yaml（設定 Telegram group_id、agents）
vim team.yaml

# 5. 啟動 AI Team
ark-team-agent team start
```

---

## 功能特色

### 多 Agent 協調
透過 `team.yaml` 定義多個 Kiro CLI agent instance，支援 4 層角色（admin / manager / leader / worker）。

### Telegram Bot 互動
- Forum Topic 路由、私聊路由
- HTML 通報（啟動 / 掛起 / 成本 / rate-limit）
- ToolTracker 工具呼叫即時顯示
- InlineKeyboard 互動按鈕

### 拍板迴路（Decision Loop）
- 權限矩陣（L1 自決 / L2 CEO‑CTO 拍板 / L3 升級人類）
- ark-agent fallback 拍板（60 分鐘無人拍板 → ark 自主決策或升 Paddy）
- 每日 09:00 拍板日報 + 24h 翻案窗口
- 決策知識自動沉澱至 agent 記憶與 Wiki

### 跨 Agent 通訊（MCP）
- 15+ MCP tools（P2P / broadcast / wiki / task 管理）
- Coordinator A2A（DAG 依賴管理 + FeedbackLoop）

### 生命週期管理
- 崩潰指數退避重啟
- 掛起偵測（30 分鐘 / 60 分鐘升級）
- 成本控制（每日上限 + 80% 警告）
- 排程引擎（cron 任務）

---

## team.yaml 最小範例

```yaml
channel:
  bot_token_env: TELEGRAM_BOT_TOKEN
  group_id: -100xxxxxxxxxx        # 你的 Telegram 群組 ID
  general_topic_id: 1

defaults:
  backend: kiro-cli
  model: auto

instances:
  ark-agent:
    working_directory: .
    description: "🎯 小娜組長 — 私訊入口、意圖分類、拍板 fallback"
    private_chat: 123456789       # 你的 Telegram user ID
    role: manager
    skip_resume: true
    auto_start: true

  cto-agent:
    working_directory: agents/cto-agent
    description: "🏴‍☠️ 技術長 — 架構設計、維運"
    role: admin
    topic_id: 2

  ceo-agent:
    working_directory: agents/ceo-agent
    description: "🗺️ 執行長 — 產品策略、決策"
    role: manager
    topic_id: 3

  pm-agent:
    working_directory: agents/pm-agent
    description: "📋 專案經理 — 需求分析、派工"
    role: leader
    topic_id: 4
```

---

## CLI 命令

| 命令 | 說明 |
|------|------|
| `ark-team-agent init` | 產生預設 `team.yaml` |
| `ark-team-agent team start` | 啟動全隊 |
| `ark-team-agent team stop [name]` | 停止全隊或單一 agent |
| `ark-team-agent team restart [name]` | 重啟 |
| `ark-team-agent team status` | 查看狀態 |
| `ark-team-agent send <name> <msg>` | 傳訊息給指定 agent |
| `ark-team-agent webbot start` | 啟動監控面板（port 13030） |

---

## API

FastAPI 預設 port 13030（可透過 `TEAM_AGENT_PORT` 覆寫）：

| 端點 | 方法 | 說明 |
|------|------|------|
| `/api/status` | GET | 所有 instance 狀態 |
| `/api/instances` | GET | 詳細資訊（含 role、restart_policy）|
| `/api/send` | POST | 傳訊息給 instance |
| `/api/reply` | POST | 回覆 Telegram 使用者 |
| `/ws/output/{name}` | WS | 即時輸出串流 |

---

## 版本歷史

詳細變更見 [CHANGELOG.md](CHANGELOG.md)。

| 版本 | 日期 | 摘要 |
|------|------|------|
| **1.0.0** | 2026-07-30 | 初始版本：Decision Loop、TG UX 優化、ark-agent 意圖分類（8 種）、10 agents 架構 |

---

## 版權

Copyright (c) 2026 paddyyang  
授權條款：[MIT License](LICENSE)

本套件基於 [Kiro CLI](https://kiro.dev) 後端，需遵守 Kiro 的使用條款。

---

## 相關資源

- [Kiro CLI](https://kiro.dev) — AI coding agent 後端
- [nana-team-agent](https://github.com/igs-paddyyang-tw/nana-team-agent) — 原始碼開發倉庫（私有）
- [發版 SOP](RELEASE_SOP.md) — 版控與發版標準流程
