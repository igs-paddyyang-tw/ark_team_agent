# ark-team-agent

> Multi AI Agent 團隊管理框架，基於 [Kiro CLI](https://kiro.dev) 後端，支援 Telegram Bot 互動、跨 Agent 協作與自主拍板決策。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.4-orange)](https://github.com/igs-paddyyang-tw/ark_team_agent/releases)

**作者**：paddyyang（[@igs-paddyyang-tw](https://github.com/igs-paddyyang-tw)）

---

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.0.4/ark_team_agent-1.0.4-py3-none-any.whl
```

**需求**：Python ≥ 3.11、[Kiro CLI](https://kiro.dev) 已安裝

---

## 快速開始

```bash
# 1. 安裝套件
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.0.4/ark_team_agent-1.0.4-py3-none-any.whl

# 2. 準備 team.yaml + .env
cp team.yaml.example team.yaml   # 或自行建立
echo "TELEGRAM_BOT_TOKEN=your-token" > .env

# 3. 啟動
python start.py
```

**最簡 start.py：**

```python
import asyncio
from dotenv import load_dotenv
from ark_team_agent.team import run_team

load_dotenv()

if __name__ == "__main__":
    asyncio.run(run_team())
```

---

## 功能特色

- **多 Agent 協調**：`team.yaml` 定義 agent，4 層角色（admin / manager / leader / worker）
- **Telegram Bot**：Forum Topic 路由、HTML 通報、ToolTracker、InlineKeyboard
- **拍板迴路**：L1 自決 / L2 CEO‑CTO 拍板 / L3 升級，ark-agent fallback，每日日報 + 翻案
- **MCP 通訊**：15+ tools，跨 Agent P2P / broadcast / wiki
- **生命週期管理**：崩潰重啟、掛起偵測、成本控制、排程引擎

---

## 部署案例

| 專案 | 說明 |
|------|------|
| [paddy-team-agent](https://github.com/igs-paddyyang-tw/paddy-team-agent) | Paddy 個人 AI 團隊（5 agents） |

---

## 版本歷史

完整變更見 [CHANGELOG.md](CHANGELOG.md)。

| 版本 | 日期 | 摘要 |
|------|------|------|
| **1.0.4** | 2026-08-05 | 安全性：`reply_file` 路徑白名單、`wiki_ingest` 封閉檢查；修正 cron 日/月/星期欄位被忽略 |
| 1.0.3 | 2026-07-31 | startup 通報去重 |
| 1.0.2 | 2026-07-31 | `chat_router` 動態找 leader/admin；`api.py` 補 import |
| 1.0.1 | 2026-07-31 | wiki_ingest 路徑、scheduler race、LLM retry 等 7 項修正 |
| 1.0.0 | 2026-07-31 | 初始版本：Decision Loop、TG UX、8 種意圖分類、10 agents |

---

## 版權

Copyright (c) 2026 paddyyang — [MIT License](LICENSE)
