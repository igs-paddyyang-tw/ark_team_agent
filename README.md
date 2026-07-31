# ark-team-agent

> Multi AI Agent 團隊管理框架，基於 [Kiro CLI](https://kiro.dev) 後端，支援 Telegram Bot 互動、跨 Agent 協作與自主拍板決策。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-orange)](https://github.com/igs-paddyyang-tw/ark_team_agent/releases)

**作者**：paddyyang（[@igs-paddyyang-tw](https://github.com/igs-paddyyang-tw)）

---

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.0.0/ark_team_agent-1.0.0-py3-none-any.whl
```

**需求**：Python ≥ 3.11、[Kiro CLI](https://kiro.dev) 已安裝

---

## 功能特色

- **多 Agent 協調**：`team.yaml` 定義 agent，4 層角色（admin / manager / leader / worker）
- **Telegram Bot**：Forum Topic 路由、HTML 通報、ToolTracker、InlineKeyboard
- **拍板迴路**：L1 自決 / L2 CEO‑CTO 拍板 / L3 升級 Paddy，ark-agent fallback，每日日報 + 翻案
- **MCP 通訊**：15+ tools，跨 Agent P2P / broadcast / wiki
- **生命週期**：崩潰重啟、掛起偵測、成本控制、排程引擎

---

## 快速開始

```bash
# 產生預設設定
ark-team-agent init

# 設定環境變數
export TELEGRAM_BOT_TOKEN="your-bot-token"

# 啟動
ark-team-agent team start
```

---

## 版本歷史

完整變更見 [CHANGELOG.md](CHANGELOG.md)。

| 版本 | 日期 | 摘要 |
|------|------|------|
| **1.0.0** | 2026-07-31 | 初始版本：Decision Loop、TG UX、ark-agent 意圖分類（8 種）、10 agents |

---

## 版權

Copyright (c) 2026 paddyyang — [MIT License](LICENSE)
