# ark_team_agent — 發布 Repo 導覽

> 給進到這個目錄的 agent（Claude Code、Codex、Kiro、其他 CLI）看。
>
> ⚠️ **這裡不是開發源。** 這個 repo 只放**發布產物與文件**。
> 改程式碼請去開發源（見下方）。在這裡改 `src/` 是無效的 —— 這裡沒有 `src/`。

## 這個 repo 是什麼

`ark_team_agent` 的**發布通道**：wheel、CHANGELOG、README。
子專案（aiops / director / fish / paddy / game 等）都是 `pip install` 這裡的 Release URL。

| 路徑 | 內容 |
|------|------|
| `dist/*.whl` | 各版 wheel（`gh release` 的 asset 來源）|
| `CHANGELOG.md` | 版本變更記錄（**寫「為什麼」，不只寫「改了什麼」**）|
| `README.md` | 安裝指令、版本歷史、badge |
| `LICENSE` | MIT |

## 開發源在哪

```
kiro-cli/projects/nana-team-agent/
├── src/ark_team_agent/     ← 真正的原始碼
├── tests/                  ← pytest（1000+ 測試）
└── AGENTS.md               ← 那邊的導覽（含模組地圖與紅線）
```

**要改任何行為，去 nana-team-agent 改，然後依 SOP 發版到這裡。**

## 發版 SOP（完整版在開發源的 `.kiro/steering/BRAIN.md`）

```
1. 在 nana：全量 pytest + CI 掃描 + 版號兩處一致（__init__.py / pyproject.toml）
2. 在 nana：**先清 build/** → python -m build --wheel → verify_wheel_skills()
3. cp wheel → 這個 repo 的 dist/
4. 更新這裡的 README（badge + 兩處安裝指令 + 版本表）與 CHANGELOG
5. git commit + push（這個 repo）
6. GH_TOKEN=<組織 token> gh release create v{ver} dist/*.whl \
     --repo igs-paddyyang-tw/ark_team_agent
7. 子專案：pip install --force-reinstall <Release URL> → 重啟 → **看版本號驗證**
```

### 三條驗證鐵律（每條都踩過）

| # | 規則 | 為什麼 |
|---|------|--------|
| 1 | **push 後驗遠端**（`git ls-remote`）| 背景執行輸出會截斷，push 被拒你不會知道 |
| 2 | **重啟後看版本號**，不是看 `systemctl is-active` | 實測 `active` 但跑的是舊碼 |
| 3 | **`build/` 必清** | setuptools 只複製新檔、不刪已消失的檔 —— 會夾帶已刪除的 skill |

### 組織 token

`gh` 預設登入個人帳號，對 `igs-paddyyang-tw/*` **無權限**，
症狀是 `Repository not found`（看起來像 repo 不存在，實際只是看不到）。
組織 token 在 `kiro-cli/.env` 的 `GITHUB_TOKEN`。

## 版號規則

| 版號 | 觸發 |
|------|------|
| Patch `x.x.+1` | bug fix、設定調整、文件、行為修正 |
| Minor `x.+1.0` | 新功能、API 擴充 |
| Major `+1.0.0` | breaking change |

> 實務上幾乎都走 Patch。**計畫文件裡的「波次代號」不是發佈版號** ——
> 曾有 26 處程式碼標記了一個從未發佈的版本，事後全部要校正。

## 你不該做的事

- 🚫 **不要在這裡改程式碼**（沒有 `src/`，改了也不會進 wheel）
- 🚫 不要手動編輯 `dist/` 內的 wheel
- 🚫 不要跳過 `verify_wheel_skills()` 直接發 release
- 🚫 不要用個人 `gh` 帳號發 release（會 `Repository not found`）
