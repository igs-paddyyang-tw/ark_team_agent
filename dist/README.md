# dist/ — 這裡沒有 whl，發布產物在 GitHub Releases

`ark_team_agent` 的 wheel **發布通道是 GitHub Release assets**，不是這個 git 樹的 `dist/`。

## 要下載某版 whl

```bash
gh release download <tag> --repo igs-paddyyang-tw/ark_team_agent -p '*.whl'
# 或直接 pip 安裝
pip install https://github.com/igs-paddyyang-tw/ark_team_agent/releases/download/<tag>/ark_team_agent-<version>-py3-none-any.whl
```

查有哪些版本：`gh release list --repo igs-paddyyang-tw/ark_team_agent`
或 https://github.com/igs-paddyyang-tw/ark_team_agent/releases

## 為什麼 dist/ 不放 whl

`scripts/build_release.py` 只把 whl 上傳成 Release asset（`uploads.github.com`），
**不回寫進發版倉的 `dist/`**。過去 `dist/` 裡殘留的 1.0.0~1.5.0 舊 whl 是早期手動提交的化石，
會讓查版的人誤以為「發版停在 1.5.0」——已於 2026-09-17 清除（issue: dist-stale-1.5.0）。

> 🔴 判準：查某版有沒有發布 → `gh release view <tag>` 看 assets，**不要**看 `dist/`。
