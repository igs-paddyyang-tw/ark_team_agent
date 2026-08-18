# CHANGELOG

所有版本的重要變更記錄於此，格式依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)。

版本號遵循 [Semantic Versioning](https://semver.org/lang/zh-TW/)：`MAJOR.MINOR.PATCH`

---

## [1.3.3] — 2026-08-18

### 🔴 重啟這件事有四套機制，而看不出哪套在生效

**回報**：「TG 觸發重啟的功能壞了 —— admin-agent 會 touch `restart.flag`，但沒人讀它。」

**結論是錯的** —— 實測 `/api/restart-service`（與 TG `/restart` 同一條路徑）：
PID `405204 → 426704`、uptime 歸零、5/5 alive。**重啟一直是好的**，因為
`_do_restart()` 除了寫 flag 還會 `os.kill(getpid(), SIGTERM)`，
Linux 上由 systemd `Restart=always` 拉起。**flag 不是機制，SIGTERM 才是。**

**但推論完全合理，所以這是套件的問題**：

| 環境 | 實際靠什麼 | `restart.flag` |
|------|-----------|---------------|
| Linux + systemd（六個生產部署全是這個） | **SIGTERM → `Restart=always`** | 沒人讀，寫了只是垃圾 |
| `start-team.sh` / `.bat` | wrapper 的 while loop | **必要** |
| Windows 前景 | `team.py` 的 win32 輪詢分支 | **必要** |
| `watchdog.py` | 它自己會讀 | **必要** —— 但**沒有任何部署在跑它** |

程式碼裡看得到三個 flag 讀取者，而它們在現行部署裡都沒在執行；
磁碟上還躺著 **2026-08-14 的殘留 flag**（nana / aiops / director 各一個）。
任何人看到都會得出「壞了」的結論。

> 🔴 **治本不是補一個讀取者，是讓「flag 存在」重新變成有意義的訊號。**

### 修法

**新增 `restart.py` —— 重啟的單一入口。**

1. **`detect_supervisor()`** 判斷「退出後誰會拉起我」，依據都是外部事實：
   `ARK_TEAM_AGENT_WRAPPER`（wrapper 自己設）、`INVOCATION_ID`（**systemd 對每個
   unit 都會設**，比找 `systemctl` 執行檔可靠 —— 後者只證明工具在，
   不證明我被它管）、`sys.platform`。
2. **flag 只在真的有消費者時才寫。** systemd 環境不寫，並順手清掉既有殘留。
3. **啟動時清殘留 + 收尾時清**（但 **wrapper 環境的收尾不清** —— 那份正是要給
   while loop 讀的）。啟動 log 會印一行 `重啟機制：systemd（…）`，
   讓下一個人不用猜。

**行為改變：沒有 supervisor 時不再假裝重啟。**
原本 TG 一律回「🔄 重啟中… watchdog 將在數秒內拉起服務」——
但本機**根本沒有 watchdog 進程**，而在前景執行的環境下 SIGTERM 只會讓服務
**關掉不再起來**。現在會明確講「服務即將停止，且不會自動起來」並給出解法。

**順帶消除一份重複實作**：`telegram.py` 與 `api.py` 各有一份「寫 flag + SIGTERM」，
延遲還不一致（1.0s vs 0.5s）。兩份必然漂移 → 收斂成一個，並用掃描測試守門。

**`watchdog.py` 標明是「可選的替代 supervisor」** —— 沒有任何地方 import 它，
只能手動 `python -m ark_team_agent.watchdog`。它 spawn 子進程時會設
`ARK_TEAM_AGENT_WRAPPER=1`（因為它自己就是 flag 的消費者）。

守門測試 `tests/test_restart_mechanism.py` **15 個**（含四種 supervisor 環境
各自的 flag 行為、殘留清理不得拋例外、不得再有第二份重啟邏輯）。
全量 **1440 passed**。

## [1.3.2] — 2026-08-18

### 🔴 `style="report"` 是 raw passthrough —— 回覆裡的 `<260>` 讓整則訊息被 TG 拒收

**回報來源**：slot-team-agent 的 `researcher-agent` 回覆含「樣本數 `<260>` 筆」，
Telegram 用 `parse_mode=HTML` 解析時把 `<260>` 當成未知標籤 → 400 Bad Request。

`_format_reply_html()` 在 `style="report"` 時原本就是 `return text` ——
**完全不跳脫**。而那個 raw passthrough 是刻意的：2026-08-10 為了讓 agent 能自己寫
`<b>`（escape-first 會把它變成 `&lt;b&gt;`），把排程 job 全部改成 `style="report"`
繞過 escape。**於是繞掉的不只是 `<b>`，是所有 `<`。**

> 🔴 **下游 fallback 讓它更難發現**：400 之後 `_rate_limited_send` 會用
> `_strip_html()` 重送純文字，而那是 `re.sub(r"<[^>]+>", "")` ——
> **`<260>` 被整段刪掉**。訊息送達了、沒有錯誤畫面，只是**報告裡的數值無聲消失**。
> 「降級成功」把一個資料正確性問題偽裝成了格式問題。

**修法不是回頭 escape 全部**（那會復發 2026-08-10 的問題），是**白名單**
`_sanitize_tg_html()`：TG 支援的 16 個標籤留、其他 `<` `>` 與裸 `&` 跳脫。

三個實作要點：

1. **只驗標籤名不夠，要驗屬性** —— `if x <b then y> 0` 的 `b` 是合法標籤名，
   於是整句 `<b then y>` 會被當標籤吃掉。現在每個標籤有允許的屬性樣式
   （`a href` / `code class` / `span class` / `blockquote expandable` / `tg-emoji emoji-id`），
   不符者當文字。
2. **順帶保證結構平衡**（TG 對不平衡標籤同樣 400）：跳脫掉某個開標籤時，
   它的 closer 也一併跳脫（否則變成落單的 `</a>`）；無主 closer 跳脫；
   收尾補關未關閉的標籤。輸出永遠是合法嵌套。
3. **既有 entity 不重複跳脫** —— `A&amp;B` 不能變成 `A&amp;amp;B`，
   但裸的 `A&B` 要變 `A&amp;B`。先把合法 entity 挖洞保護再跳脫。

`style="chat"` 路徑**完全未動**（它本來就是 escape-first，安全）。
report 模式「無 header、agent 自控格式」的語意也不變。

守門測試 `tests/test_tg_html_sanitize.py` **38 個**。全量 **1417 passed**。

### `team_doc` 的範本註解（純註解，零 runtime 影響）

`templates/team.yaml` 與 `examples/team.yaml` 的 `team_doc` 說明改寫 ——
原本只寫「未設則省略該段」，讀者不會意識到那代表什麼。

實測動機：2026-08-17 稽核六個部署，**只有一個設了 `team_doc`**。
其餘五個團隊的 agent 從來沒見過指揮鏈或協作流程，而且：

> **agent 讀不到 `team.yaml`，只讀 `TEAM.md` 的最終內容** —— 它不知道
> `team_doc` 這個設定存在，所以「缺了指揮鏈」既不會被它察覺、也無法自己補。

新註解明講這件事、建議直接設定（不要留註解狀態）、並說明 1.3.1 的實際編制註記
與幽靈／隱形角色 warning。**這類「機制存在但要人來開」的事，正確位置是範本註解
與專案文件，不是 agent 的 always-on context** —— 對 agent 提示它無法行動的事只是雜訊。

✅ 註解隨本版（1.3.2）發佈 —— 新部署 `init` 起就帶得到。

### `examples/` 隨套件搬入本 repo

`examples/`（10 檔）原本留在 `nana-team-agent` —— 它記述的是**套件的**設定格式，
而套件已於 2026-08-17 搬到這裡。留在 nana 會變成第二個真相來源
（實測 2026-08-17 就已經需要人工同步過一次兩份範本的差異）。

`templates/team.yaml`（`init` 用的最小範本）與 `examples/team.yaml`
（含 manager/leader/worker 的 worked example）是**兩份不同用途的檔案**，
不是重複 —— 逐行比對確認 167 行相異，兩份都保留。

## [1.3.1] — 2026-08-17

> **同一個缺陷家族的第五批**：宣告了、寫死了、或根本沒接上，而且**沒有任何錯誤訊息**。
> 這批裡影響最大的一項（`run_team()` 不設 logging）讓三個部署共 21 個 agent
> **持續在無日誌狀態下運行** —— 服務是活的、API 回 200、TG 有反應，
> 所以沒人發現看不到任何 INFO。**假綠燈比壞掉更難察覺。**

### 🔴 `run_team()` 從不初始化 logging（三個部署 21 agents 全程無日誌）

各部署的 `start.py` 直接 `from ark_team_agent.team import run_team`，**不走 CLI**。
而 logging 的 `setup_logging()` 只在 `cli.py` 裡呼叫 → 走 `start.py` 的部署
root logger 沒有任何 handler，level 停在 WARNING。

實測（2026-08-17，近一小時 journal）：

| 部署 | 入口 | INFO 行數 |
|------|------|-----------|
| nana | CLI | 238 |
| aiops / director / paddy | `run_team()` | **0**（總共 7 行，全是 stderr 漏出的）|

看不到的東西：scheduler 觸發、agent 就緒／崩潰、MCP 掛載結果、degraded 明細。
出事時完全無從追。

修法：`run_team()` 在 root logger **沒有 handler 時**補上 `setup_logging()`，
沿用 `ARK_LOG_LEVEL` / `ARK_JSON_LOG` 環境變數。已設過的（如 CLI）不覆蓋。

### `team_doc.command_chain` 與實際編制不一致時無訊號

指揮鏈圖是**角色層級**的簡化（`使用者 → admin → manager → leader → worker`），
但同一角色可以有多個 instance。nana 有**兩個 admin**（`ark-agent`、`cto-agent`），
讀圖的 agent 會誤判自己在鏈上的位置。

`TEAM.md` 現在會在圖下附一行實際分佈：

```
> 實際編制：admin×2、leader×1、manager×1、worker×7。上圖是角色層級的簡化 ——
> 同一角色可能有多個 instance，完整成員見上方表格。
```

並對兩種不一致發 warning：圖上有的角色實際不存在（**幽靈角色**）、
實際有的角色沒上圖（那些 agent 在指揮鏈上**隱形**）。

### `scheduler` 的 drift 日誌目標寫死 `cto-agent`

`_builtin:knowledge-digest` 的 drift 記錄目標是 nana 專屬的 instance 名，
其他部署一律落空。改為依 **role** 推導：找 `admin`（其次 `manager`）的
`working_directory` 下的 `knowledge/`。

⚠️ **不能猜 `agents/{name}` 路徑** —— aiops 的 agent 目錄直接在專案根下，
而它另有一個殘留的 `agents/itom-agent/`。猜路徑會把日誌寫進沒人在用的目錄
（第一版就是這樣）。四個部署實測皆正確解析。

### `cli.py` 的入口 agent 名寫死 `ark-agent`（13 處）

`init` 產出的 `.kiro/agents/<name>.json`、`knowledge/<name>/` 櫃子、
以及 `schema.md`／`index.md`／`log.md` 內容全部寫死 `ark-agent`。
新增 `--entry-name`，預設值保持 `ark-agent`（向後相容）。

### 🐛 `init` 在全新環境**必定失敗**（`UnboundLocalError`，1.3.0 與更早版本都有）

修上面那項時發現的既有 bug：`raw` 在第 350 行才賦值，卻在 319 行就被使用。
症狀是 `init` 走到寫 `mcp.json` 那步直接炸，rc=1。

沒被發現的原因：**既有部署都不會再跑 `init`**。也就是說這個指令對
README 的第一個讀者是壞的，對所有現存使用者是無感的 —— 典型的
「只有新人會踩到，而新人以為是自己弄錯」。

### 測試

`tests/test_v131_fixes.py` — 12 個守門測試，含正反兩向
（logging 要補、但**不得覆蓋**呼叫端已設的）。全量 **1345 passed / 7 skipped**。

## [1.3.0] — 2026-08-17

> **第一個從 paddy-team-agent build 的版本。** 功能與 1.2.26 **完全相同** ——
> 實測 wheel 內容除版號相關檔案外逐檔一致。這個 minor 跳號唯一的意義是標記
> **build 來源已遷移**。

### 開發環境遷移：nana-team-agent → paddy-team-agent

| | 遷移前 | 遷移後 |
|---|---|---|
| 套件開發源 | nana（同時是 11 agents 部署 + 2 子專案宿主） | **paddy**（5 agents，最精簡） |
| 發版 build | nana 的 `src/` | paddy 的 `packages/ark-team-agent/` |
| Release repo | `igs-paddyyang-tw/ark_team_agent` | **不變** |
| 消費者的 pip install | — | **URL 與機制照舊，無感** |

**為什麼搬**：nana 身兼三角（套件開發源 / 自身 11 agents 部署 / aiops+director 宿主），
導致版本混淆、驗證一個小 bug fix 要拉起整個生態系、協助子專案時可能誤觸套件原始碼。
paddy 只有 5 agents，改壞影響小、啟動快。

> ⚠️ **搬走套件不會讓 nana 變輕** —— nana 有 5,026 個追蹤檔，套件本體只佔 **57 檔（1.1%）**，
> `projects/`（aiops + director）佔 48.6%。搬移買到的是「**套件開發**這件事變輕」。

### 版號策略

以「誰 build 的」切分，**不預先宣告**：

| 階段 | 版號 |
|------|------|
| nana build | 接續 `1.2.x` —— 最後一版 **1.2.26**（＝搬移基準線） |
| **paddy build 的首版** | **1.3.0** |
| 之後 | 1.3.1+ |

計畫文件曾有互斥敘述（主文「從 v1.3.0 起算」vs 風險 3「版號接續不 reset」），
2026-08-17 定案以上表為準。

### 搬移的驗證方式

| 驗證 | 結果 |
|------|------|
| 原始碼一致性 | 182 檔逐檔 sha256 與 nana 來源相同 |
| **wheel 一致性** | paddy build vs nana build **187/187 逐檔相同**（唯一差異 `dist-info/METADATA`）|
| 測試 | 96 檔**原封不動**搬移 → **1259 passed, 0 failed** |
| skill 集合 | `build_release.py` 的 `verify_wheel_skills()` 通過（11 個）|

`tests/deployment/` **不隨套件搬移** —— 那層測「本機部署設定」而非套件邏輯。
分兩層的理由：測試是搬移的度量衡，**度量衡不能在搬移過程中被修改**，
否則無法區分「測試過了因為搬移成功」與「測試過了因為我把它改到會過」。

## [1.2.26] — 2026-08-17

### 🔴 aiops / director 的緊急通知與 L3 拍板卡，從來沒送出過

補 CI 盲區時逼出來的功能性缺陷。`notify_paddy()` 與 `send_l3_escalation_card()`
都寫死：

```python
paddy_chat_id = self._private_chat_map.get("ark-agent")
```

`ark-agent` 是 **nana** 的入口名。實測各部署的私訊入口：

| 部署 | 入口 instance | 結果 |
|------|--------------|------|
| nana | `ark-agent` | ✅ 有效 |
| **aiops** | `admin-agent` | ❌ 拿不到 → 直接 return，通知丟棄 |
| **director** | `tech-agent` | ❌ 同上 |

而且 log 還印「`ark-agent has no private_chat_id configured`」——
對它們來說**連這句訊息都是錯的**（它們沒有 ark-agent）。

修法：抽出 `owner_chat_id()`／`_owner_chat_id()`，**依 role 推導**
（優先 `admin`，再退回任何已綁 `private_chat` 的 instance）。
這個 pattern 套件裡本來就有 —— `/api/reply` 的 admin fallback 一直是這樣寫的，
只是這兩處沒用它。

### CI 補上 `ark-agent`，一次噴出 24 處

`check_hardcoded_names.py` 的 `BANNED` 原本只有 `ceo-agent` / `cto-agent` +
角色代號。補上 `ark-agent` 後噴 **24 處**，分類處理：

| 類型 | 數量 | 處置 |
|------|------|------|
| 功能性缺陷 | 3 | 上述 `owner_chat_id` 修正 |
| 與名字無關的路徑判斷 | 4 | `team_mcp` 的知識庫櫃子改看**目錄是否存在**；其中一處是**完全多餘**的（它加的路徑與下方 fallback 相同且有去重） |
| 訊息來源標記 | 3 | `"source": "ark-agent"` → `_instance`（實際發送者） |
| 升級預設對象 | 1 | `decision_manager` 的 `fallback` 預設改 `_fallback_owner()`（第一個 admin → 原 owner），原本會把升級發給不存在的 instance |
| `init` 範本內容 | 13 | 標 `# hardcode-ok: init 範本預設值` |

> 為什麼原本不在名單裡：推測是怕 `Author: ark-agent` 署名造成誤報。
> 但 `scan()` 早就跳過 docstring 與註解，**那個顧慮不成立** ——
> 代價是這個名字的硬編碼一個標記都沒有，不是因為它更乾淨，是沒人在看。

### 測試分兩層（為套件搬移做準備）

新增 `tests/deployment/` —— 測「**本機部署設定**」而非套件邏輯，**不隨套件搬移**。

| | `tests/*.py` | `tests/deployment/*.py` |
|---|---|---|
| 測什麼 | 套件邏輯 | 本機部署的實體設定 |
| 依賴 | 只用 `tmp_path` 與自建 fixture | 讀專案根的 `team.yaml`、`scripts/`、`projects/*/` |
| 搬移時 | **一起搬**（原封不動應綠燈） | **留在原地** |

**為什麼現在做**：套件開發要移交 paddy，而計畫的成功標準是「`pytest` 核心測試全過」。
若測試在搬移途中被改（改 fixture、改路徑、改斷言），那句話就失去證明力 ——
你無法區分「測試過了因為搬移成功」和「測試過了因為我把它改到會過」。
**測試是搬移的度量衡，度量衡不能在搬移過程中被修改。**

盤點結果：98 個測試檔裡只有 **3 個測試**有本機依賴，全部在當天新寫的
`test_dead_config_a_batch.py`。既有 97 檔的 fixture 早就自包含 ——
**耦合是持續產生的，不是歷史債**，所以判準寫進 `tests/deployment/README.md`。

搬移時順帶補了兩個部署守門：每個部署必須有 `private_chat` 入口
（否則上述通知無處可送）、`group` 必須指向存在且 role 為 leader 的 instance。

測試：1156 → **1158 passed**（+5 部署測試 −3 搬走）。

## [1.2.25] — 2026-08-17

> **這一版是「套件開發搬移至 paddy」的基準線。** 從 paddy build 的首版是 1.3.0，
> 行為應與本版完全一致，差異只有 build 來源。1.3.0 出問題時，對照點就是這裡。

### MCP 進度行不再被當成「就緒」

`READY_PATTERN` 含 `ctrl-c to start chatting now` 與 `start chatting` ——
而那正是 Kiro **MCP 初始化進度行**的尾巴：

```
⠋ 0 of 1 mcp servers initialized. ctrl-c to start chatting now
```

**兩個 pattern 都吃中它。** 2026-08-17 實測（復現 daemon 的 pipe spawn，非 pty）：
這行在 **2.4 秒** 就印出，而當下 **`k=0`，一個 MCP server 都還沒起來**。
於是 daemon 判定就緒、`status=RUNNING`、啟動訊息佇列 ——
**多數 agent 帳面上的「2.0s 就緒」其實是「MCP 剛開始初始化」。**

真就緒訊號是 `All tools are now trusted`（2.8–2.9s，位於 logo／`Model: auto` 之後，
`--resume` 與否都會印）。它本來就在 pattern 裡，所以移除進度行不會讓判定失去依據。

**實機驗證**（nana 11 agents）：

| | 啟動耗時 | 意義 |
|---|---|---|
| 改動前 | 2.0s | 假就緒（`0 of N`） |
| 改動後 | **4.0s** | 真就緒 |

11/11 alive、degraded 空、`MCP N 個` 觀測數字不變。代價是多一輪輪詢（2s）。

**為什麼不改成「等 `k == m`」**（原本的直覺方案）：實測 Kiro **不等 MCP 完成就進 chat**
—— 印 `⚠ 0 of 1 … Servers still loading: - fetch` 之後直接進入交談。
所以 `k == m` 的進度行**可能永遠不出現**，那種寫法會等到 timeout 才失敗。
這個假設是靠實測推翻的，不是靠讀程式碼。

### 與 1.2.22 的把關是兩道不同的防線

| 防線 | 攔什麼 | 時序 |
|------|--------|------|
| 1.2.22 `_wait_for_ready` 的 MCP 失敗把關 | buffer **已含**失敗訊息 | 失敗訊息先到 |
| 1.2.25 收緊 `READY_PATTERN` | 進度行被當就緒 | 進度行先到 |

兩者都需要 —— 單有把關擋不住「先看到進度行就 return True」，
單有收緊擋不住「已經失敗但還沒退出」。

> ⚠️ **仍未完全解決**：Kiro 進 chat 後 MCP 才在背景失敗的情況，daemon 已回報就緒，
> 只能靠 `_on_exit` 的事後歸因（`rc=3` + `mcp_startup_failed:` degraded）。
> 要根治需要 Kiro 提供「MCP 全部就緒」的明確訊號，那不在套件可控範圍。

測試：1155 → **1156 passed**（原本記錄「進度行會被誤判」的兩個測試反轉為守門）。

## [1.2.24] — 2026-08-17

### 未知 instance 欄位不再被靜默丟棄

`_merge_instance` 的 `if hasattr(ic, k): setattr(...)` —— 條件為假時**什麼都不做**。
設定檔寫了、載入時被丟掉、**沒有任何訊息**。

實測代價（aiops）：9 個 instance 全部寫 `resume_session: true`，
而套件的欄位名是 `skip_resume`，**且語意相反**（要接續 ＝ `skip_resume: false`）。
於是：

| instance | 設定意圖 | 實際（欄位被丟 → 依 role 預設）|
|---|---|---|
| admin-agent / itom-agent | 接續 | 接續 ✅ 碰巧一致 |
| **ic-agent + 6 worker** | 接續 | **開新對話** ❌ 相反 |

7 個 agent 的行為與設定意圖相反，而且從設定檔完全看不出來。
這是本專案反覆修過的「宣告了沒接上」家族中最隱蔽的一種 ——
前幾例是「欄位對但沒讀」，這是「**欄位名拼錯**」。

修法是一道**通用警告**（攔下所有拼錯），而不是逐個支援錯誤的別名：

```
WARNING team.yaml instance 'ic-agent' 的欄位 'resume_session' 不存在
        → **已忽略**（設定不會生效）。你是不是想寫 'skip_resume' 或 'max_user_sessions'？
```

**建議排序刻意不用字面相似度。** `difflib.get_close_matches("resume_session", …)`
的首選是 `max_user_sessions`（共同子串多），而正解是 `skip_resume`。
改成依「詞素交集 ÷ 候選詞素數」排序（`_suggest_field`）：

```
skip_resume       ∩ {resume, session} = {resume}  → 1/2 = 0.50  ✅
max_user_sessions ∩ {resume, session} = {session} → 1/3 = 0.33
```

詞素少而命中的欄位更可能是使用者想寫的那個。完全無共同詞素時才退回字面相似度。

**噪音控制**：加警告前先掃過全部六個部署 —— 只有 aiops 的 `resume_session` × 9
會觸發，其餘五個乾淨。常駐假警報會讓人習慣性忽略整個警告機制，
所以「加了會噴多少」必須先量過再決定。

新增守門測試：掃本機所有 `team.yaml`，有未知欄位就紅燈 ——
下次有人再發明一個欄位名會立刻被抓到。

測試：1151 → **1155 passed**。

## [1.2.23] — 2026-08-17

### 「宣告了沒接上」四連修 —— 同一個病灶家族

四項的共同病徵：**設定檔看起來很完整，但套件根本沒在讀**，且沒有任何錯誤訊息。

**① `private_chat_routing` 整段是死設定**

`TeamConfig` **沒有這個欄位**，而 `telegram.py` 用
`getattr(config, "private_chat_routing", {})` 讀 —— 於是永遠拿到 `{}`。
nana 宣告了 `mode` / `fallback` / `confidence_threshold` 與 3 群 20 個關鍵字，
**keyword 路由從未生效過一次**。

沒被發現的原因值得記下來：**主路徑是好的** —— 入口 agent 用 `SOUL.md` /
`intent-classify.md` 的提詞做 LLM 意圖分類，那不經過套件。keyword 只是備援。
**備援壞了而主機制正常，所以零症狀。**

修法不只是補欄位。接上之後**必須依 `mode` 分流**（新增 `keyword_routes_for()`）：

| `mode` | 套件行為 |
|--------|---------|
| `keyword` | 用 `keyword_routes` 決定目標（空的話發警告 —— 那等於沒設定） |
| 其他 / 未設 | 回 `{}`，不介入，一律送 `bound_instance` 讓入口 agent 自己判斷 |

> ⚠️ 若無條件套用 keyword，nana（`mode: ark-agent`）的行為會**改變** ——
> 「技術」「bug」會在入口 agent 之前被攔去 cto-agent，**繞過它的 7 種意圖分類**。
> 那是把一個沉睡的備援變成會改變既有行為的攔截器。已用真實 `team.yaml` 測試固定此行為。

`fallback` 與 `confidence_threshold` **套件不解讀** —— confidence 是 agent 的 LLM
判斷結果，不會回傳給套件，套件無從得知該不該降級。那兩個值由 agent 的 steering 遵循。
這點已寫進 `config.py` 的欄位註解，避免下一個人再以為它們沒接上。

**② `route_by_keyword(fallback="ark-agent")` 死預設**

`ark-agent` 是 nana 的 instance 名，對其他部署毫無意義。它一直沒出事是因為
唯一呼叫端明確傳了 `bound_instance` —— 那行是**死代碼**，留著只會讓人以為
套件有「預設路由目標」這回事。改回 `None`（回傳型別 `str | None`）：
沒有目標就明說沒有，由呼叫端處理。呼叫端加 `or bound_instance` 兜底。

**③ 決策知識回寫用名字比對，而非 `working_directory`**

`write_knowledge_feedback` 裡 `if original_owner == "ark-agent"` 走
`knowledge/ark-agent/raw/decisions/`，其他走 `agents/{name}/…`。
其他部署沒有這個名字，**永遠走 else，是死分支**。

真正的判準不是「叫什麼名字」，而是「`working_directory` 是否就是專案根」：
根目錄的 agent 不能用 `base/knowledge/`（那層是共用圖書館，會污染其他櫃子），
要用自己的櫃子 `base/knowledge/{owner}/`。抽出 `_knowledge_raw_dir()`，
任何部署的「wd=專案根」agent 都正確，不只 ark-agent。查不到設定時退回舊佈局。

**④ `build_release.py --release` 從未成功執行過**

`RELEASE_REPO` 指向 `ark-team-agent-release`（**不存在**，實際是 `ark_team_agent`），
所以 `--release` 一律在存在性檢查就 `sys.exit(1)` —— 歷來發版都是手動走 BRAIN 的 SOP。

修路徑之外**加了發版前置檢查**：發版 repo 的 README 與 CHANGELOG 必須已含本版號。
少了這道，`--release` 會發出「GitHub 有 tag、但 repo 文件還停在上一版」的半套發佈
—— 使用者照 README 的安裝指令抓到舊版 URL，而且沒有任何錯誤。

### 已知盲區（本版未處理）

`scripts/check_hardcoded_names.py` 的 `BANNED` 只有 `ceo-agent` / `cto-agent` +
角色代號，**`ark-agent` 不在名單**。所以上面 ②③ 這類硬編碼 CI 抓不到 ——
`scheduler.py` 的 4 處 `cto-agent` 全都被迫加 `# hardcode-ok` 標記，而
`decision_manager.py` 的 `ark-agent` 一個標記都沒有，不是因為它更乾淨，是沒人在看。
加入名單前要先排除 `Author: ark-agent` 署名，否則大量誤報。

測試：1136 → **1151 passed**（+15）。

### 範本補上 `team_doc` 註解（純註解，零 runtime 變更）

`team_doc` 從 v1.1.3 就存在，但**沒有任何範本提到它** —— 沒讀過
`write_team_context` 原始碼的人不會知道可以設。它控制 TEAM.md 的兩段：

```yaml
team_doc:
  command_chain: "使用者 → admin → manager → leader → worker"
  workflow: "leader(分析) → worker(執行) → leader(驗收) → manager(彙報)"
```

**未設 → TEAM.md 直接省略這兩段**（不是留空標題，見
`test_omitted_when_unset`）。所以 slot / fish / paddy 的 agent 從來只看到
成員表與通訊規則，沒有指揮鏈；nana 有設，所以它的 TEAM.md 兩段都在。

已在 `templates/team.yaml` 與 `examples/team.yaml` 補上註解形式的範例
（維持預設不啟用 —— 內容是團隊自訂的，套件不該硬給一套）。

> ⚠️ **這批註解不在 1.2.22 的 wheel 裡**（發版在前）。新部署用
> `ark-team-agent init` 產生的 `team.yaml` 要等下次發版才會帶上這段。
> 純註解，不影響任何行為。

### agent 看不到的機制，要寫在人看的地方

順帶記錄一個判斷：**沒有讓產生器在未設 `team_doc` 時提示 agent「可以去設」**。
agent 只讀 TEAM.md 的最終內容，且通常沒有修改自己設定檔的權限與 skill ——
對它提示等於製造一段它無法行動的雜訊。這種「機制存在但要人來開」的事，
正確位置是範本註解與專案文件，不是 agent 的 always-on context。

## [1.2.22] — 2026-08-17

### 清掉兩個常駐假警報 —— 都是判定邏輯的問題，不是服務有問題

`/api/health` 的 `degraded` 長期掛著兩筆，兩筆都不代表任何真實故障。
常駐假警報的代價不是雜訊本身，是**維運會開始習慣性忽略 degraded** ——
那等於把這個欄位廢掉。

**① `mcp_undeclared_moved` —— 宣告移除後的殘骸被當成「使用者忘了宣告」**

1.2.14 修過一次同類假警報，但只修了「舊手寫 `{server}` + 新宣告
`{server}-{instance}` 並存」。漏掉第三種情況：**宣告曾存在、後來被移除**。

判斷式 `name in cfg.mcp_servers` 拿 `bigquery-data-agent` 去比對宣告名
`bigquery`，永遠是 `False` —— 於是**套件自己產生**的殘骸被歸類成
「使用者手寫但忘了宣告」，永久計入 degraded，還附上「補進 team.yaml 即回歸」的
指引。而該宣告是**刻意移除**的（`GOOGLE_APPLICATION_CREDENTIALS` 指向的
憑證檔不存在，宣告了只會產生新的 degraded）——
照著指引做只會把剛清掉的問題加回來。

修法：`{server}-{instance}` 形式的 key **必然是本套件寫的**（wrapper script
逐 instance 產生），使用者手寫的一律是裸名。據此把殘骸單獨歸類，
`_reason` 標「可安全刪除」，不計入提醒。條目仍保留在 `_disabled`（ADR-005
不變 —— 套件不自動刪使用者設定的最後副本）。

**② `slow_startup` 閾值 0.5 → 0.75**

2026-08-17 實測 nana 11 個 agent 的啟動耗時，呈**雙峰**且中間無值：

| 群 | 耗時 | 例 |
|---|---|---|
| 快 | 2.0–10.0s | architect(mcp=3) 2.0s、pm(mcp=2) 4.0s |
| 慢 | 25.7–30.3s | cto(mcp=4) 30.3s、data(mcp=2) 28.3s |

**與 MCP 數無關** —— 這推翻了 1.2.14 標記時的假設（「慢啟動 MCP 疊加就是
下一次 rc=3」）。60s × 0.5 = 30s 正好切在慢群頂端，所以報出來的不是
「這個 agent 有問題」，而是「這次剛好越線 0.3s」：同一批裡 28.4s 的三個都不報。

提到 0.75（45s）後仍留 15s 告警餘裕，真的逼近 timeout 才報。
各次耗時未被丟棄 —— `state.last_startup_seconds` 一直有記錄可查。

> 慢群的成因（固定卡 25–30s）尚未查明，只確定不是 MCP 疊加。
> 這是獨立的觀測項，不該由一個會誤報的閾值代勞。

### `_wait_for_ready` 對 MCP 啟動失敗把關 —— 失敗不再被認列為就緒

追查慢啟動時挖出來的，比原問題嚴重：**MCP 啟動失敗的 agent 會被回報為就緒**。

誤判機制比預期更根本 —— `READY_PATTERN` 含 `ctrl-c to start chatting now`
與 `start chatting`，而 Kiro 的 MCP **進度行**正是：

```
⠸ 6 of 7 mcp servers initialized. ctrl-c to start chatting now
```

兩個 pattern 都被它吃中。於是失敗當下的 buffer 仍讓 `is_ready()` 回 `True`，
daemon 把 status 設成 `RUNNING`、啟動訊息佇列，**4–30 秒後**進程才以 `rc=3` 死掉。
這段期間送進去的訊息全丟，而狀態看起來一切正常。
`has_mcp_startup_failure` 原本只用在 `_on_exit` 的**事後**歸因。

修法：`is_ready` 分支前先檢查，命中就記 `mcp_startup_failed:{name}:{k}/{m}`
並 `return False` 走啟動失敗路徑（`status=CRASHED`，重啟邏輯即刻接手）。
`--require-mcp-startup` 保證 Kiro 必然退出，所以不會誤殺還能用的 agent ——
只是把「必死」提早認列。

> ⚠️ **未修的部分**：進度行被當成就緒這件事本身還在。若 Kiro 先印進度行
> （我們認列就緒）**才**印失敗訊息，這道把關攔不到，仍得靠事後歸因。
> 要根治得讓 `is_ready` 不把 `k < m` 的進度行當就緒 —— 那會改變**所有** agent
> 的啟動語意（現在多數 agent 的「2.0s 就緒」其實是「MCP 剛開始初始化」），
> 風險級別不同，另案處理。

### 開機錯開（`scripts/stagger_boot_start.py`）

2026-08-17 09:02 實測：**七個** user service 在同一秒被 systemd 拉起，
合計 50+ 個 kiro-cli 併發 spawn（原以為只有三個）。

在 `ExecStart` 前插一道**只在開機 600 秒內生效**的 `ExecStartPre` sleep，
依序 aiops 60s / director 120s / slot 180s / paddy 240s / ninja-team 300s
（nana 不延遲，它是主控）。判斷 uptime 是必要的 —— `Restart=always` 會讓
`ExecStartPre` 在每次崩潰重啟時也執行，無條件 sleep 等於拉長故障恢復時間。

> systemd 會展開 `%`，所以 `${up%.*}` 在 unit 檔裡必須寫成 `${up%%.*}`；
> `|| true` 也不可省 —— `ExecStartPre` 回非 0 會讓整個服務啟動失敗。

### TEAM.md 不再謊報自己的 policy

`write_team_context` 產出的內容有 **4 處**寫死「本文件由系統自動產生
（policy=always）」，但 `team_md` 的**預設是 `once`**（只在檔案不存在時寫入）。
對 `once` 的部署來說那句話是反的 —— 它不但不會每次重啟產生，而且**永遠不會再產生**。

實測代價（slot-team-agent）：該專案從 fish fork 而來
（`3317693 chore: fork from fish-team-agent`），TEAM.md 原封帶過來 ——
內容是 **v1.0.5 時代**產生的，含當年 `backend.py` 硬塞的 `cto-agent`
（v1.2.0 已移除硬塞），而 slot 的 `team.yaml` **從未宣告過**這個 instance
（`git log -S` 全歷史 0 次）。`policy=once` 讓 14 份 TEAM.md 全部凍在 fork 當天，
**升級套件到 1.2.21 也修不好**。TEAM.md 是 always-on steering，
13 個 agent 每次對話都吃到這張含幽靈成員的表。

而檔案上「自動產生」四個字，讓診斷往「產生器壞了」的方向去 ——
真相是它根本沒再產生過。

修法：`tc_policy` 本來就在手上，改印**實際值**，並抽出 `_team_md_policy_note()`
依 policy 給對應說明。`once` 模式明說「重啟不會更新」並附重新產生的方法。

測試：1123 → **1136 passed**（+3 殘骸判定、+7 ready 把關、+3 policy 誠實性）。

## [1.2.21] — 2026-08-14

### MCP 宣告機制首度啟用 —— 並修掉三個讓它「宣告了沒用」的缺陷

`mcp_servers` 宣告機制 1.2.13 就做好了。實況比「都沒用」複雜一些：
**aiops 早就宣告了**（github / kibana，1.2.13 升級時一併搬入），
**nana 與 director 從沒宣告過任何 server** —— director 的 team.yaml 甚至
明確寫了「本團隊不宣告」的註解。本次補的是 nana。
實測 agents **完全沒有**繼承 `~/.kiro/settings/mcp.json` 的 7 個 server ——
cgroup 內只有 `team_mcp` × 11。那些 server 只服務使用者自己的 kiro session，
所以 agent 的 `openai` / `github` 等能力一直是缺的。

啟用時撞到三個缺陷：

#### ① 宣告的 `fetch` 被硬編預設**靜默覆蓋**

`_write_mcp_config` 的 elif 鏈會無條件覆蓋 `servers["fetch"]`，
所以 `mcp_servers.fetch` 的宣告寫了完全沒用，也沒有任何警示。

修法：宣告存在時跳過硬編分支。判斷條件要同時認 `cfg.mcp_servers`
與 **per-instance key**（`fetch-<instance>`）—— 宣告的 server 是用後者存的
（wrapper script 逐 instance 產生），只判斷 `"fetch"` 永遠是 False。

#### ② 兩份 fetch 並存 —— 11 個 agent 跑出 22 個 fetch 進程

`servers` 是從既有檔案讀進來的，光「不覆蓋宣告」不夠，得主動清掉舊的硬編條目。
沒清之前一半的進程是壞的那份，還會繼續觸發 `rc=3`。

#### ③ 預檢漏掉「env 內的路徑」

原本只檢查 `command` 與 `args` 裡的絕對路徑。實測 bigquery 的
`GOOGLE_APPLICATION_CREDENTIALS` 指向不存在的檔案 —— server 起得來、
Kiro 回報成功，但**所有呼叫都會失敗**。這正是 ADR-004 要消滅的靜默失敗
（當時只防空 token，沒防壞路徑）。現在 env 值若是不存在的絕對路徑就不部署。

### 順帶修好 `fetch`（rc=3 的真正原因）

`uvx mcp-server-fetch` 目前起不來：上游把 `McpError` 改名 `MCPError`，
與快取的 `mcp` SDK 版本歪斜。`uvx --refresh` 無效，必須釘住舊 SDK：
`--with 'mcp<1.15'`。這就是 architect-agent 出現
`rc=3 mcp-startup-failed（初始化到第 1/2 個 server 就失敗）` 的來源。

⚠️ **aiops／director 尚未宣告 fetch，它們的 fetch 仍是壞的那份。**

### nana 的宣告內容

| server | 授予 | 說明 |
|--------|------|------|
| `fetch` | 全員（`defaults.mcp`）| 釘住 SDK 的版本；原本就是全員給，只是壞的 |
| `openai` | ark-agent · ai-dev-agent | LLM 整合 |
| `github` | cto · coder · architect | 套件維護、實作、審核 |
| `kibana` | cto-agent | 服務監控 |

**未宣告**（前置條件不存在，宣告了只會產生 degraded 雜訊）：
`grafana`（腳本目錄整個不在）、`bigquery`（憑證檔不存在）。
兩者在全域設定裡都還留著，但那份設定早已過時。

密鑰一律 `${VAR}` + 值放 `.env`（`team.yaml` 有版控，不可寫明文）。

**成本**：MCP 進程 130 → 162 個、11.7 → 13.2 GB（+1.5 GB）。
用 per-instance 白名單控制，不是每個 agent 都給。

+7 測試。全量 1116 → 1122 passed。

## [1.2.20] — 2026-08-14

### 多使用者 Session 隔離治理（M1–M3）

盤點 `multi_user` 機制時發現四類問題。**功能至今未啟用** ——
三個部署的 `team.yaml` 都沒有 `multi_user: true`，磁碟上 0 個 session 目錄。
所以缺陷是潛在而非正在發生，但也因此**現在修的成本最低**（零遷移）。

文件：`docs/specs`／`designs`／`plans` 的 `2026-08-14-multi-user-session-isolation-*`

#### M1 表示法地基 —— 分隔符 `#` → `~`

`#` 是 URL fragment 分隔符，而 `api.py` 有兩個端點以路徑參數收 instance 名
（`/api/output/{name}`、`/api/instances/{name}/restart`）。實測：

```
/api/output/admin-agent#937896656
  → server 收到 path='/api/output/admin-agent'  fragment='937896656'
```

`user_id` 整段丟失，**且不報錯** —— 操作靜默打到 base instance。

改用 `~`（RFC 3986 unreserved）。它是唯一 URL／POSIX／Windows **三邊都安全**的
選擇：`#` 在 Windows 檔名合法但 URL 危險，`:` 恰好相反。

新增 `SessionKey`（frozen dataclass），分隔符只在該模組出現一次。
刻意**不繼承 `str`** —— 繼承後任何字串操作都不報錯，等於保留了要消除的誤用空間。

新增 `Daemon.entity_instances()` / `session_instances()` 單一出口，
9 處執行期列舉改用它（TEAM.md 成員表、可派工對象、manager 通知、TG 清單、
`broadcast_all`、`general_topic` 掃描）。原本 28 處列舉只有 2 處記得過濾 ——
與 1.2.16 的 IDLE 事故同型（用 enum grep 盤點，漏掉 4 處字串字面值比對）。
所以這次改用**掃描測試**守門，不靠 grep。

#### M2 上限與驗證 —— 修掉一個「看起來有保護」的設定

`config.py` 宣告 `max_user_sessions: int = 5`，`resolve_session()` 也如實呼叫
`_enforce_session_limit()`，但那個函式的實作是 `return`。
每個 session 是一個獨立 kiro-cli 子進程 → 啟用後使用者數就是進程數。
**比「沒有上限」更糟**：讀設定的人會以為安全。

`AWAITING_REPLY` **永不回收** —— 那代表有使用者正在等這個 agent 回話。
這是 1.2.16 引入 `IDLE` 的價值兌現處：在那之前「回覆完」與「有人在等」
都是 `AWAITING_REPLY`，**無法區分，也就無法安全回收**。
受保護但仍佔額度；全部都在等回覆時放行超額並留 warning。

新增 `ark-team-agent sessions [--purge]` 盤點無主 session 目錄，
**預設只報告不刪除** —— 誤刪的代價是使用者的對話歷史。

#### M3 per-user 工作區隔離 —— 隔離的是資料，不只是對話

`resolve_session()` 原本只做 `replace(ic, name=key, multi_user=False)`，
**`working_directory` 與本體完全相同**：

```
base  wd=agents/admin-agent
user  wd=agents/admin-agent      ← 相同
```

所有使用者的 `memory/journal.md`、`memory/daily/` 都寫進同一個目錄。

現在每個 session 有 `<base_wd>/sessions/<uid>/`：`memory/` 獨立、
`.kiro/steering/*.md` 首次**複製**繼承（不是 symlink）、**不建立** `knowledge/`。

不建 `knowledge/` 不只是省磁碟 —— 空目錄會**誘導 agent 往裡面寫**，
而 `wiki_query` 查的是本體的知識庫，寫進去的知識誰也查不到。

> 注意範圍：**套件層**的 per-user 記憶（`memory.py` → `memory/user_<uid>.md`）
> 本來就是隔離的，本次不動。

### 修正 1.2.19 留下的 2 個紅測試

1.2.19 把 `agents_md` policy 加回來（語意不同：為非 Kiro agent 產生根目錄導覽檔），
卻沒更新 1.2.18 寫的「死設定已移除」守門測試 → `test_dead_policies_removed` 與
`test_legacy_yaml_with_removed_keys_still_parses` 兩個測試在 HEAD 上是紅的。

### 修正本版自己造成的回退

M2 的 commit 把另一個 session 剛加的 `telegram:` → `channel:` 別名回退掉了
（`config.py` 一行）。根因是**共用 working tree**：對方的 commit 已進遠端，
但磁碟檔案落後於它，我 `git add` 磁碟版就把功能刷掉了 ——
BRAIN.md 第一鐵律記載的失效模式，這次犯的人是我。已復原別名與對應的 2 個測試。

> 順帶：`cli.py` 被我從 CRLF 正規化成 LF（`read_text`/`write_text` 的
> universal newlines 副作用）。內容無損，但那次 diff 多出 2000+ 行雜訊。

### N5 記憶治理補完 —— 機制早就有，但從沒啟用過

`_builtin:memory-consolidate` 在 **1.2.4** 就實作了，`examples/scheduler.yaml`
也有範例，但**三個部署的 `scheduler.yaml` 都沒有它** —— 從沒執行過一次。
（與 `mcp_servers` 同型：機制做好但沒啟用。）

而且它管的東西不對：處理 `memory/daily/*.md`（實測只有 3 個檔）與
`memory/memory.md`（45 行），但**沒管 `.kiro/steering/MEMORY.md`** ——
那才是 **always-on 載入**、每次對話都吃 context 的檔案（ark-agent 636 行 / 36 KB、
pm-agent 626 行 / 66 KB），而它自己開頭就寫著「> 2 週的段落移到 knowledge/」。

新增 `memory_archive.py`：把 `## YYYY-MM-DD` 分節中超過 14 天的搬到
`knowledge/raw/memory-archive/YYYY-MM.md`，原處留指標。

**確定式，不用 LLM** —— 這是搬移不是摘要，分節有結構、日期比對是機械的。
LLM 會引入「摘要漏掉重要事實」的風險，搬移只要「歸檔寫成功才動原檔」
就零資訊損失（已用 439 行逐行比對驗證）。

安全設計：只動日期分節（`## 專案快照`／`## 待辦` 這類當前狀態絕不碰）、
先寫歸檔再改原檔、append 不覆寫、`min_bytes=8000` 門檻
（10 個 agent 只有 2 個超標，對 9 行的檔案硬歸檔只會變吵）。

已啟用於三邊 `scheduler.yaml`（每週日 03:30）。
實跑 ark-agent：636 → **456 行**（−31%），12 段歸檔。

+21 測試（重點在「不該動什麼」）。

### 測試

1024 → **1115 passed** / 6 skipped（+91：SessionKey 26、上限 22、隔離 18、
記憶歸檔 21，另修復 2 個紅測試）。

> ⚠️ **M4 實機驗證尚未執行**（`docs/plans/…-plan.md` 的 4.2／4.3）。
> 需臨時開啟 `multi_user` 驗三件事：衍生 wd 下 `.kiro/` 能否完整產生、
> 複製 steering 後 kiro-cli 能否載入人格、回收後 `--resume` 能否接回對話。
> 1.2.19 剛踩過「16 個單元測試全綠但實機全錯」，這步不可省。

## [1.2.19] — 2026-08-14

### `AGENTS.md` 成為規範主文件，放在**工作區根目錄**

> **問題**：Kiro 自動載入 `.kiro/steering/*.md`，其他工具（Claude Code、Codex）不會 ——
> 它們進到 agent 目錄後**沒有任何入口**，只能自己亂翻。
> 實測 `AGENTS.md` 在 agent 根目錄、nana 根目錄、套件 repo **全都不存在**。
>
> **定調**：`AGENTS.md` 為規範**主文件**，其他工具的專屬檔（如 `CLAUDE.md`）直接參考它。

- **新增 `_write_root_agents_md()`**：在每個 agent 的 **`working_directory` 根目錄**
  （**不是** `.kiro/` 底下）產生導覽檔 —— 工作區結構、記憶怎麼用、知識庫怎麼查、
  產出放哪、回報格式、Output Marker、紅線。
  **套件原本完全沒有寫 agent 根目錄檔案的機制**，這是新能力。
- 內容**依實際結構產生** —— 不存在的目錄不列入（導覽檔不該指向不存在的路徑）；
  標題用代號（從 `description` 前綴推導，如「🏴‍☠️ 魯夫」）而非 raw instance 名。
- **新增 `_ensure_workspace_dirs()`**：補齊工作區目錄（實測 10 個 agent 有 **4 個**
  缺 `artifacts/` 與 `scripts/`）。只建目錄與 `.gitkeep`，不動既有內容。
- **`agents_md` policy 回歸**（`once` / `always` / `skip`，預設 **`once`** ——
  agent 可自行補充工作區筆記而不被覆寫）。

> ⚠️ **`agents_md` 在 1.2.18 才被移除、1.2.19 回歸，語意不同**：
> 舊＝把根目錄 AGENTS.md「同步／複製」到各 agent 的 `steering/`（實作早已拿掉）；
> 新＝在各 agent 的**工作區根目錄**產生導覽檔。
> 本意相同（讓每個 agent 都有 AGENTS.md），這次是把實作做對。

### 整併並刪除 `.kiro/steering/AGENTS.md`（11 份）

實測那份 126 行的「AI Team 共用規範」**8 節裡有 6 節在別處已有** ——
所以這是**去重**不是搬遷：

| 章節 | 處置 |
|------|------|
| 工具使用規則（`fs_write`）／訊息分流／Output Marker | **獨有** → 移入根 `AGENTS.md`（`fs_write` 改寫成「Kiro ↔ 你」的工具對照表）|
| 協作流程 | 刪除（`TEAM.md` 已有）|
| AI 開發流程 SDD | 刪除（`dev-workflow.md` 已有）|
| 回覆風格 | 移入根 `AGENTS.md`（含 **❌／⚠️ 的語意分野**：❌ 是「你的操作不成立」、⚠️ 是「系統狀況」）|
| 產出路徑規則／禁止事項 | 移入根 `AGENTS.md`（`BRAIN.md` 的 Output 表回答的是「知識 vs 交付物」，兩者互補不重複）|

**Kiro 端零損失**：`BRAIN.md` 用 Kiro 既有的檔案引入語法 `#[[file:AGENTS.md]]`
把主文件拉進 context —— 單一真相，Kiro 與非 Kiro 兩邊吃到同一份。

### 文件

- nana 根 `AGENTS.md`：產生的工作區導覽 + **手寫的 repo 面**（模組地圖、常用指令、
  共用 working tree 的 git 紅線）。`once` policy 保護手寫部分不被覆寫。
- `CLAUDE.md`：開頭指向 `AGENTS.md` 並說明分工（只保留 Claude Code 專屬的
  `@` 匯入與工具對照）；`@.kiro/steering/AGENTS.md` 匯入改為 `@AGENTS.md`。
- **套件 repo 新增 `AGENTS.md`** —— 內容不同（那是發布通道不是開發源）：
  明寫「這裡沒有 `src/`，改程式碼要去 nana」+ 發版 SOP + 組織 token 陷阱。

測試：`tests/test_root_agents_md.py`（16）。

### 實機驗證抓到的缺陷（同版修掉）

首次重啟後 11 份都產出了，但**標題全是 raw instance 名**（`cto-agent（cto-agent）`
而非「🏴‍☠️ 魯夫」）。根因：代號從 `cfg.instructions` 推導，而 `daemon` 傳的是
`instructions=ic.system_prompt` —— 實測**多半是 `None`**，代號其實在 `ic.description`。

單元測試沒抓到，因為測試 fixture 直接傳 `instructions=DESC`（把「輸入從哪來」也一起
假設掉了）。修法：`KiroBackendConfig` 加 `description` 欄位、`daemon` 傳 `ic.description`、
推導改以 description 優先，並補兩個迴歸測試鎖住來源。

> 這是「單元測試全綠但實機錯」的典型。**實機重啟驗證不能省。**

## [1.2.18] — 2026-08-14

### 架構定調：**每個 agent 獨立運作與發展，只有知識庫共用**

依此清掉三個「設定還在、實作早就拿掉」的死 policy：

| 死設定 | 宣稱行為 | 實際 |
|--------|---------|------|
| `kiro_files.steering.agents_md`（預設 `symlink`） | AGENTS.md 從根目錄同步到各 agent | `backend.py` 明寫 `AGENTS.md — REMOVED: no longer synced`，**零引用**。實測 12 份 AGENTS.md 是內容相同的複本（md5 一致），**沒有任何同步機制** |
| `kiro_files.knowledge.shared_wiki`（預設 `symlink`） | 共用知識庫掛進各 agent | `_mount_shared_knowledge` 是 **DISABLED 空殼**，卻仍被呼叫 |
| `kiro_files.knowledge.private_wiki` | 私有 wiki 初始化 | 零引用 |

- 三個欄位與其 YAML 解析全部移除；`_mount_shared_knowledge` 的呼叫一併拿掉。
- 舊 `team.yaml` 若仍留著 `kiro_files.knowledge` 區塊，**解析時直接忽略**（不報錯、不影響啟動）。
- 知識庫共用**維持不變** —— 靠 `wiki_query` MCP 工具，本來就不是靠 symlink 或複製檔案。

> 這與 1.2.13 的 `mcp_servers` 死程式碼是同一類問題：**設定存在、實作已移除**，
> 讀設定的人會以為它有效，然後在錯誤的假設上做決定。

### 清除孤兒 agent 目錄 + CI 守門

- 刪除 `agents/report-agent/`：team.yaml **零提及**，卻有 77 個檔案
  （6 個 skill，全是別處也有的副本；非 skill 檔 **0**），且完全沒有
  steering／mcp.json／agent.json —— 那些檔只在 `start_instance` 時寫，而它從不啟動。
  是 2026-07-29「5 → 8 agent」的遺留。
- 新增 `scripts/check_agent_dirs.py`：`agents/` 目錄必須與 team.yaml 成員一一對應。
  **孤兒目錄（有目錄無成員）→ exit 1**；缺目錄只提示（首次啟動會建立）。
  `working_directory` 不在 `agents/` 底下者自動排除
  —— orchestrator 常設 `.` 指向專案根，不排除會永遠報一個假缺口。

### 文件更正

`CLAUDE.md` 原寫「`TEAM.md` 由服務啟動時自動產生（policy=always），手動修改會被覆寫」，
**實際預設是 `once`** —— 只在檔案不存在時寫，重啟不覆寫（實測三份 mtime 都停在 2026-07-30）。
目前內容正確只是因為成員沒變過；**改了成員後 TEAM.md 不會自動更新**。

測試：`tests/test_agent_dirs_check.py`（7）。

## [1.2.17] — 2026-08-14

### 修 1.2.16 漏掉的四處 —— `alive` 會隨 agent 完工而下降

> 盤點 IDLE 影響面時用 **enum** grep（`InstanceStatus.RUNNING`），但有四處是用
> **字串字面值**比對（`i["status"] == "running"`）—— 兩種寫法互相掃不到。
> 後果：agent 完工進 `IDLE` 後不計入 alive → `/api/health` 的 `alive` 隨工作完成而下降，
> 外部監控誤判成「agent 掉了」。

- `api.py` `/api/health`、`team.py` 團隊就緒統計 ×2、`team_mcp.py` `query_team_status`
- 新增守門測試：掃全部 `src/*.py`，同時出現 `"running"` 與 `"awaiting_reply"`
  卻沒有 `"idle"` 的行一律失敗（用字串掃字串，補 enum grep 的盲區）

## [1.2.16] — 2026-08-14

### 新增 `IDLE` 狀態 —— 區分「有人在等」與「做完了沒待辦」

> `reply()` 原本不論來源一律轉 `AWAITING_REPLY`，但該狀態語意是「**有人在等我回話**」。
> 排程跑完、agent 間任務做完都沒有人在等。
>
> **後果比誤報嚴重**：hang 偵測與升級兩處都寫 `!= AWAITING_REPLY`
> → **agent 一旦回覆過，hang 偵測就被關掉**，真掛死要等安全網（4-6 小時）才看得見。
> 實測 7 天：safety-net 33 次、decider-stuck 31 次、rule1 12 次。

| 狀態 | 語意 | hang | 決策規則 | crash |
|------|------|------|---------|-------|
| `AWAITING_REPLY` | 有人在等我回話 | 跳過 | 套用 | ✅ |
| `RUNNING` | 我在做事 | 超時要叫 | — | ✅ |
| **`IDLE`（新）** | 做完了、沒待辦 | 跳過 | 跳過 | **✅ 照常** |

- `send_message(..., source=)`（keyword-only、預設 `"user"`）記 inbound 來源；
  `reply()` 分流：`user` / `web*` → `AWAITING_REPLY`，其餘 → `IDLE`。
  **未標記的呼叫端維持既有行為。**
- 已標記非互動來源：`scheduler` / `peer`（`/api/send`）/ `system`
- 六處狀態判斷納入 `IDLE`（含**崩潰偵測**與送訊就緒閘門）
- 未知來源保守進 `IDLE`：誤判成 `IDLE` 只少一次「跳過 hang」，
  誤判成 `AWAITING_REPLY` 會關掉 hang 偵測
- TG `/status` 新增 `idle` → 🟢 待機

### Telegram 訊息文案

- 「對方沒在線上」四處統一（原本四種寫法、兩種符號，且都沒講「你能做什麼」）
- 代號與 raw instance 名混用 → 統一走代號
- **例外訊息不再丟給使用者**（原 `❌ 重啟失敗：{e}`）—— 原因寫 log
- 掛起通報移除三個工程師視角的「可能原因」（使用者一個都不能行動），
  改一句人話 + 明確代價
- 符號體系：❌ 只給「你的操作不成立」，⚠️ 給系統狀況

測試 **1002 passed / 0 failed**。

## [1.2.15] — 2026-08-13

### 修 1.2.14 沒修乾淨的假警報

> 1.2.14 想修掉 `mcp_undeclared_moved` 假警報，但**只分類「本次搬移」的條目**。
> 條目一旦進了 `_disabled` 就不會再被搬移 → 1.2.13 留下的舊條目永遠算在缺口裡。
> 實測：升到 1.2.14 重啟後假警報還在。

- 計數改為掃**整個** `_disabled`：`key in mcp_servers` → 已由宣告取代（不計入），否則才算缺口
- 順手補正舊條目的 `_reason`，讓 `mcp.json` 自己也說得清楚
- 新增兩條迴歸測試（既存條目要能重新分類 / 真缺口仍須回報）

> **教訓**：修「狀態會被持久化」的邏輯時，只處理本次寫入路徑是不夠的 ——
> 既有狀態不會自己重新流過那條路徑。

測試 **983 passed / 0 failed**。

## [1.2.14] — 2026-08-13

### 崩潰迴圈治理（P3）

> **`max_retries` 在有活動的迴圈裡永遠到不了。** 它是「**連續**失敗」上限，但 health loop
> 的活動偵測是「有新 output → `crash_count = 0`」——「崩潰 → 重啟 → 印啟動訊息
> （**就是 output**）→ 再崩潰」的迴圈中，計數每次被歸零。
>
> 實測 director 2026-07-31~08-12 累積 **2421 筆 crash**（峰值 846/日），
> **每一筆 detail 都是 `crash_count=1`** —— 計數被歸零的直接證據。

- **改以時間窗計數**（與活動偵測無關）：`RestartPolicy` 新增 `crash_window_minutes`(60) /
  `max_crashes_per_window`(5) / `max_cooldowns`(3)。窗內達門檻即判迴圈，**不等** `max_retries`。
- **冷卻遞增** `600s × 2^(n-1)`，上限 1 小時（原固定 10 分鐘）。
- **超過 `max_cooldowns` → 停止自動重啟**（`halted`）+ `degraded` + event + 通報。
- 冷卻結束只清 `crash_count`，**不清窗內記錄** —— 清了等於讓迴圈原地復活。
- 人工介入的唯一訊號是 `stop_instance`（自動重啟只呼叫 `start_instance`）→ 在此解除
  `halted`，否則被停手的 agent 連人工 restart 都起不來。
- `/api/status` 增 `lifetime_crashes` / `crashes_in_window` / `cooldown_count` / `halted`。

### MCP 啟動觀測（P4）

> 現場（2026-08-13 15:08）：`rc=3` 被歸因為「多為上游/API 錯誤」，實際是
> `Error: One or more MCP servers failed to start`。且 Kiro 說 **7 個** server，
> 套件寫進該 agent 的只有 4 個 —— 差額來自**全域** `~/.kiro/settings/mcp.json`，
> 不在套件管轄內，預檢管不到。

- 新增 `KiroBackend.mcp_progress()` / `has_mcp_startup_failure()`：解析 Kiro 自印的
  `k of m mcp servers initialized`（取最後一筆、先清 ANSI）。
- **`rc=3` 正確歸因**：有 MCP 證據時 `last_exit_cause` 改為
  `mcp-startup-failed（初始化到第 k/m 個 server 就失敗）`+ `degraded`；無證據則不動。
- **啟動耗時**：spawn→ready 秒數，超 `startup_timeout_ms` 50% 記 `slow_startup`。
  MCP server 由 Kiro 生，**無法逐一計時**，總時長是唯一可得的代理指標。
- **外部 server 落差**：載入數 > 宣告數+2 → `degraded["mcp_external_servers:{name}:{n}"]`。
- `/api/status` 增 `last_startup_seconds` / `mcp_loaded` / `mcp_declared`。

### 修 1.2.13 的假警報

- `mcp_undeclared_moved` 會把「已由宣告取代」的舊條目誤報為未宣告：宣告式 key 是
  `{server}-{instance}`、升級前手寫的是 `{server}`。現在 `_reason` 標「已由宣告取代」
  且**不計入**提醒，只報真缺口。

測試：+23（`test_crash_loop` 11 / `test_mcp_startup_observability` 10 / `test_mcp_prune` +2），
全量 **981 passed / 6 skipped / 0 failed**。

## [1.2.13] — 2026-08-13

> MCP 設定治理。規格／設計／計畫／驗收四份文件見 nana repo 的
> `docs/{specs,designs,plans,reports}/2026-08-13-mcp-*`。

**根因：`cfg.mcp_servers` 從來沒有人填值，是死程式碼。** 全套件只有宣告與消費兩處提及它，
`config.py` 零解析 → 自訂 MCP 只能手改 `.kiro/settings/mcp.json`，而該檔 `policy=always`
每次啟動被重寫。疊上 `--require-mcp-startup`（「**任一** server 失敗即 **exit 3**」），
一個壞條目就讓整個 agent 起不來。

### 可用性（P0）

- **啟動前預檢**：`command` 解析不到、或 `args` 內以 `/` 開頭的路徑不存在 → **剔除該 server
  並繼續啟動 agent**，記入 `degraded[]`。不移除 `--require-mcp-startup`（移除等於把
  all-or-nothing 換成 nothing-guaranteed —— MCP 全掛也「啟動成功」）。
- **`command` 絕對路徑解析**：venv/bin → `which` → **找不到就明確失敗**（systemd 的
  `PATH` 不含 venv/bin；fallback 只會把問題延後成難歸因的 exit 3）。
- **`mcp.json` 原子寫入**（tmp + `os.replace`）；寫入被中斷不再留半截 JSON。
- **壞檔備份** `mcp.json.corrupt-<ts>` 再重建（原本靜默覆蓋）。
- wrapper 改用 `shlex.quote`（原本 `command` **完全未引用**，含空格的路徑會壞）。

### 宣告式設定（P1）

```yaml
defaults:
  mcp: [kibana]                   # 全體預設掛載
mcp_servers:
  kibana:
    command: mcp-server-kibana    # 相對值 → 絕對路徑
    env: { KIBANA_URL: "${KIBANA_URL}" }   # 經 wrapper 生效（Kiro 忽略 mcp.json 的 env）
  fetch: { enabled: false }       # 內建 server 亦可停用（不記 degraded）
instances:
  sre-agent:
    mcp: [github]                 # 與 defaults 取聯集
    mcp_exclude: [kibana]         # 例外
```

- 掛載 = `(defaults.mcp ∪ instance.mcp) − mcp_exclude`（**union 非覆寫**）。
- `${VAR}` 缺值或空值 → **不部署該 server** + `mcp_unresolved_env`
  （空 token 會「啟動成功但所有呼叫失敗」，是最難查的靜默失敗）。
- `mcp` / `mcp_exclude` 引用未宣告名稱 → **啟動即擋**。

### 宣告即真相（P2）

- 未宣告的 server 移至 `mcp.json` 的 `_disabled`（**不刪除**，`autoApprove` 完整保留），
  **永久保留 + 常駐提醒**，補進 `team.yaml` 即回歸。
- 只在 `team.yaml` 真的有 `mcp_servers` 宣告時才修剪 —— 未宣告的既有部署零影響。

### 升級注意

- 既有部署**不加** `mcp_servers` → 行為僅多出預檢驗證，其餘不變。
- 要加 `mcp_servers` 時，**必須同時宣告該部署現有的所有手寫 server**，否則會落入
  `_disabled`（可完整還原，但當下少工具）。

測試：+36（AC-001~AC-018 全覆蓋），全量 958 passed / 6 skipped / 0 failed。

## [1.2.12] — 2026-08-13

> 內建 skill 的**版控、下發、對齊、打包**四個環節一次補齊。原定 1.2.11 的崩潰鑑識未單獨發佈，併入本版。

### 內建 skill（11 個）

- **範本全數進版控**：`templates/skills/` 原有 **29 / 95 檔未進 git**（含 `ark-agent-cli`、`ark-news-daily`、`ark-telegram-sender` 三個完整 skill，以及 `wiki_guard.py`、`wiki_taxonomy.py`、`loop-rules.md`、各 skill `evals.json`、superpowers hooks 與 13 份 fixtures）。未追蹤數 **29 → 0**，換機 clone 後建出的 wheel 內容自此一致。
- **新增 `skills_policy: sync`**（`once` 仍為預設，既有行為不變）：
  - 逐檔比對**內容**（`filecmp.cmp(shallow=False)`；`copytree` 會保留 mtime，比 stat 會漏）
  - 有差異才覆寫、缺檔補上；**不刪除** agent 自行新增的檔案（不套 `rsync --delete` 語意）
  - 未知 policy 值（如 `snyc`）**發 WARNING** 再退回 `once`，不靜默退化
  - bundled 與 `skills_source` 自訂來源皆適用
  - ⚠️ **既有部署要吃到 skill 內容更新，須在 `team.yaml` 加 `kiro_files.skills.policy: sync`**；維持 `once` 只會補「新增的 skill」，不會更新既有內容。
- **對齊 canonical 庫**：5 份 SKILL.md 更新，`metadata.schema_version` + `category` 齊備度 **8/11 → 11/11**。
- `templates/skills/README.md` 不再被當成 skill 部署。
- **audit empty-skill-dir 規則**：`audit_skills.py` 新增 P2 規則，攔截空殼目錄。

### 打包正確性

- **`build_release.py` 打包前清 `build/`**：setuptools 只把新檔**複製**進 `build/lib`，**不刪除**來源已消失的檔案。實測 `build/lib` 停在 2026-07-31 → 自那之後每個 wheel 都夾帶 v1.0.5 就已刪除的 `ark-daily-decision-digest` 與 `ark-policy-translate`。
- **新增 `verify_wheel_skills()`**：打包後比對 wheel 內 skill 集合與來源，**多一個或少一個就中止發版**。夾帶錯誤 skill 不會讓打包失敗，只會安靜地把廢止 skill 發到每個團隊。
- 本版 wheel 實測 **11 / 11 一致**。

### 崩潰鑑識（原定 1.2.11，未單獨發佈）

- 崩潰偵測處記 `proc.returncode` + `capture(200)`（截 1500 字）進 log 與 `EventType.CRASH` 的 detail；鑑識失敗不影響重啟。
- 新增 `Daemon.classify_exit(rc, intentional=)`：
  - asyncio subprocess 被訊號殺是**負數**（`-9`），非 shell 的 `128+N`（137）→ `137` 視為自行退出
  - `-9` 僅標「**疑似** OOM，請查 `dmesg`」不斷言（SIGKILL ≠ OOM）
  - `-11` SIGSEGV、`rc>0` 自行退出、`rc=0` 正常結束
- **主動停止旗標 `InstanceState._stopping`**（`stop` 設、`start` 清）：`terminate()/kill()` 是 `-15/-9`，不標記會讓每次 `systemctl restart` 產生**假 OOM**。
- **`/api/status` 暴露 `last_exit_code` / `last_exit_cause`**。

測試：**922 passed / 6 skipped / 0 failed**。

## [1.2.10] — 2026-08-12

- **templates/skills 更新**：
  - 新增 `ark-html-report`（單檔 HTML 報告，5 風格 token + 17 元件）+ `ark-md-report`（結構化 MD 報告，與 html-report 成對）。新建 agent 自動獲得。
  - 移除 `ark-daily-decision-digest` + `ark-policy-translate`（nana Decision Loop 專屬，子專案不需要）。根治「重啟後被重新部署」問題。
- 版本 1.2.9 → 1.2.10。

## [1.2.9] — 2026-08-11

- **修 v1.2.7 引入的 UnboundLocalError（生產回歸）**：`telegram.py::_on_message` 函式內 `from .event_log import get_event_log` shadow 模組層同名 → 整條 TG 訊息路徑爆掉。修：移除函式內 import + AST 回歸測試。
- **`log_to_leader` 顯示代號**：原印 `🔒 [sec-agent]`（raw instance 名），改走 `_CODENAMES` + `_md_to_html`，所有團隊受益。
- **補實作 4 個 API 端點**（drift 歸零）：`DELETE /api/dashboard/board/{task_id}`、`GET /api/dashboard/costs/trend`、`GET /api/dashboard/timeline`、`WS /ws/events`。

## [1.2.8] — 2026-08-11

- **code-spec-validator 假警報修正（57%→0）**：
  - 解析 `APIRouter(prefix=)`：22/54 drift 原因是同批端點被重複計成 missing+extra。
  - 排除 `templates/` 目錄掃描。
  - 註解行不計為路由/前綴。
  - 測試維度只取未勾選 `- [ ]` + 跳過 `docs/reports/` + CJK 依標點分段。
- **新增 `docs/references/api-endpoints.md`**：45 端點單一真相表（由 code 掃描產生）。
- Drift score 54→0，API 維度 0→100，總分 56→79。

## [1.2.7] — 2026-08-11

- **取經整合（借鏡 ninja / hoyeah）**：
  - CI 反模式掃描：`.parent` 層數推導、套件位置反推 `team.yaml`、錯誤分類裸 HTTP 數字、寫死 `host:port`（+ docstring 追蹤，不誤判說明文字）。
  - 新增 `_builtin:output-ttl`：依 BRAIN.md TTL 掃 `output/`，超期記 `degraded[output_ttl_overdue:N]`（只提醒不刪，無超期自癒）。
  - Skill 自薦：ToolTracker 步數 ≥ `SKILL_HINT_STEPS`(12) 時提示可沉澱為 Skill。
  - Chat trace：新增 `EventType.ROUTE`，記錄路由決策（action / 目標 agent / 來源 user）。
- **修 ToolTracker 步數低報**：原取 `len(_lines)`（受 `_MAX_LINES=8` 截斷）→ 長任務一律顯示 8；改用 `_total_steps` 真實累計。
- 文件：新增「模型分工」成本優化 pattern（快模型路由／強模型幹活，零程式）。

## [1.2.6] — 2026-08-10

- **修專案根定位缺陷（pip 安裝環境失效）**：原 `Path(__file__).parent.parent.parent` 在安裝後指到 `.venv/lib/python3.x` 且不拋例外。
  - 新增 `paths.py`：`find_project_root`（哨兵 `team.yaml`，找不到拋錯）、`resolve_project_root`（注入 > `ARK_TEAM_AGENT_HOME` > cwd 哨兵）、`package_asset`。
  - `reply_task_image` 原永遠失敗 → `task_screenshot.py` 搬進 `ark_team_agent/scripts/` + `package-data`（隨 wheel 發布）。
  - `run_spec_validator` 掃錯目錄 → `TeamScheduler(project_root=...)` 可注入，`team.py` 注入 `get_home()`。
  - `src/` 執行碼已無 `.parent.parent.parent`；spec-validator 無 drift 日誌目標時改明示 log + `degraded`。

## [1.2.5] — 2026-08-10

- TG 五指令目標明確化：/start（系統介紹+使用者 ID+授權狀態+如何對話，不 gate）、/status（+主程式健康行）、/help（新增：指令+職責導向成員）、/restart（先跳確認鍵，按確定才重啟）。set_my_commands 補 help。全動態、通用。

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
