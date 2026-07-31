# 版控與發版 SOP

> 作者：paddyyang  
> 適用倉庫：`nana-team-agent`（開發）→ `ark_team_agent`（發版）  
> 最後更新：2026-07-31

---

## 倉庫架構

```
nana-team-agent（私有，GitHub: igs-paddyyang-tw/nana-team-agent）
    ↓ python -m build
    dist/ark_team_agent-x.y.z-py3-none-any.whl
    ↓ gh release create
ark_team_agent（公開，GitHub: igs-paddyyang-tw/ark_team_agent）
    ↓ pip install <whl URL>
使用者環境
```

- **開發倉庫**（`nana-team-agent`）：實際程式碼、測試、agents、文件。私有，不對外。
- **發版倉庫**（`ark_team_agent`）：README、CHANGELOG、RELEASE_SOP、GitHub Releases。公開，只放可安裝的 `.whl`。

---

## 版本號規則（Semantic Versioning）

```
MAJOR.MINOR.PATCH
```

| 變更類型 | 升版哪位 | 範例 |
|---------|---------|------|
| 破壞性 API 變更、重大架構重構 | MAJOR | 1.x.x → 2.0.0 |
| 新功能、新模組（向後相容） | MINOR | 1.0.x → 1.1.0 |
| Bug fix、文件修正、小優化 | PATCH | 1.0.0 → 1.0.1 |

版本號定義在兩個地方，**必須同步**：
1. `src/ark_team_agent/__init__.py` 的 `__version__`
2. `pyproject.toml` 的 `version`

---

## 發版 SOP

### 前置條件

- [ ] `nana-team-agent` 所有測試通過：`pytest`（目前 21+ tests）
- [ ] `main` 分支最新，無未 commit 的改動：`git status --short`
- [ ] 版本號已在 `__init__.py` 和 `pyproject.toml` 更新
- [ ] `CHANGELOG.md` 已更新（在 `ark_team_agent` repo）

---

### Step 1：更新版本號（在 nana-team-agent）

```bash
cd D:\kiro-cli\projects\nana-team-agent

# 修改版本號
# 1. src/ark_team_agent/__init__.py
#    __version__ = "1.1.0"
#
# 2. pyproject.toml
#    version = "1.1.0"

# 確認一致
grep -n "version" src/ark_team_agent/__init__.py pyproject.toml
```

---

### Step 2：跑測試

```bash
cd D:\kiro-cli\projects\nana-team-agent
.venv\Scripts\python.exe -m pytest
# 全部通過後繼續
```

---

### Step 3：打包

```bash
cd D:\kiro-cli\projects\nana-team-agent

# 清理舊的 dist/
Remove-Item dist -Recurse -ErrorAction SilentlyContinue

# 打包
.venv\Scripts\python.exe -m build --wheel

# 確認產出
ls dist/
# ark_team_agent-1.1.0-py3-none-any.whl
```

---

### Step 4：更新 ark_team_agent repo（發版 repo）

```bash
cd D:\kiro-cli\projects\ark-team-agent-release

# 1. 更新 CHANGELOG.md（加新版本段落）
# 2. 確認 README.md 版本號已更新

git pull
git add CHANGELOG.md README.md
git commit -m "docs: release v1.1.0"
git push origin main
```

---

### Step 5：建立 GitHub Release

```bash
cd D:\kiro-cli\projects\ark-team-agent-release

gh release create v1.1.0 \
  --repo igs-paddyyang-tw/ark_team_agent \
  --title "ark-team-agent v1.1.0" \
  --notes "$(cat << 'EOF'
## ark-team-agent v1.1.0

### 安裝

pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.1.0/ark_team_agent-1.1.0-py3-none-any.whl

### 變更摘要
- 新增 XXX 功能
- 修正 YYY bug

完整 CHANGELOG: https://github.com/igs-paddyyang-tw/ark_team_agent/blob/main/CHANGELOG.md
EOF
)" \
  D:\kiro-cli\projects\nana-team-agent\dist\ark_team_agent-1.1.0-py3-none-any.whl
```

---

### Step 6：驗證

```bash
# 確認 Release 頁面正確
gh release view v1.1.0 --repo igs-paddyyang-tw/ark_team_agent

# 驗證安裝可用
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/v1.1.0/ark_team_agent-1.1.0-py3-none-any.whl --dry-run
```

---

### Step 7：在 nana-team-agent 打版本 tag

```bash
cd D:\kiro-cli\projects\nana-team-agent

git tag v1.1.0
git push origin v1.1.0
```

---

## 快速 Patch 發版（緊急修正）

```bash
# 1. 修 bug，改版本號 x.y.0 → x.y.1
# 2. pytest 通過
# 3. python -m build --wheel
# 4. gh release create vX.Y.1 ...（同 Step 5）
# 5. git tag + push tag
```

---

## 版本命名慣例

| 類型 | 範例 | 說明 |
|------|------|------|
| 正式版 | `v1.0.0` | 穩定，可生產使用 |
| 預覽版 | `v1.1.0-beta.1` | 功能完整但待驗收 |
| 候選版 | `v1.1.0-rc.1` | 最終測試中 |

---

## 發版 Checklist

```
□ pytest 全部通過
□ __init__.py 版本號已更新
□ pyproject.toml 版本號已更新（與 __init__.py 一致）
□ CHANGELOG.md 已更新
□ nana-team-agent main 分支 commit + push
□ dist/*.whl 已打包
□ ark_team_agent README.md 版本號更新
□ gh release create 成功，whl 附件可下載
□ nana-team-agent git tag + push
```

---

## 驗證環境資訊

測試通過的環境（官方支援）：

| 環境 | 版本 |
|------|------|
| Python | 3.11, 3.12 |
| OS | Windows 11 22H2+, Ubuntu 22.04 LTS |
| Kiro CLI | latest |
| python-telegram-bot | 21.10 |
| FastAPI | 0.115+ |

---

*SOP 由 paddyyang 制定，2026-07-31*
