# CHANGELOG

所有版本的重要變更記錄於此，格式依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)。

版本號遵循 [Semantic Versioning](https://semver.org/lang/zh-TW/)：`MAJOR.MINOR.PATCH`

---

## 1.7.12 (2026-09-04)

### 進度列孤兒收尾 —— 收尾點原本綁在兩個不保證發生的事件上

`ToolTracker.finalize()`（把「第 N 步 撰寫中…」收成完成狀態）只在兩處觸發：

| 觸發點 | 條件 | 保證發生？ |
|---|---|---|
| `telegram.py` 的輪詢 | **下一段 stdout 到來** | ❌ |
| `api.py` 的 reply | **送出 reply** 時 `tracker.reset()` | ❌ |

agent 中途停產出（逾時／中斷／hang）且不送 reply → 兩條路都不觸發
→ 進度訊息凍在原地永不收尾。

**症狀會誤導**：看起來像有東西還在跑、或像路由被搶
（原始回報者正是這樣誤判成「維運長對話被搶」）。

### 成因就在那個 `continue`

```python
# 沒有新輸出 — 檢查是否該 flush
if not buffer:
    continue          # ← buffer 清空後直接跳過，tracker 永遠不被檢查
```

### 修法：加第三條路（不取代原本兩條）

`ToolTracker.is_stale(threshold)` + 輪詢迴圈在 `continue` **之前**檢查：

```python
if tool_tracker.is_stale(_ORPHAN_FINALIZE_SECS):
    await tool_tracker.finalize()
    tool_tracker.reset()
```

- 用既有的 `_last_edit` 當基準，**不另記時間戳**（多一份狀態就多一個會漂移的地方）
- `_ORPHAN_FINALIZE_SECS = 180`，刻意遠大於 `idle_threshold`（2 秒，那是 buffer
  flush 用的）—— agent 思考、等上游 API、跑長工具的正常間隔可能有數十秒，
  **收尾太快會把「還在跑」的任務誤標成完成**
- 同時比 hang 偵測（預設 60 分）快，否則使用者早就看到凍住的進度列了

> 💡 **它的放大因子已在 1.7.10 消失**：問題單原本說「reply 通道故障使排程 agent
> 永遠送不出 reply → 孤兒必然增生」。1.7.10 之後 headless 的 reply 走 log 出口
> 並呼叫 `_rollback_reply_status`，那條放大路徑斷了。所以它從「必然」降回「偶發」。

### 守門

12 條，反證 5 項各紅 1-3 條。

> 🔴 其中一項反證第一次**沒有變紅** —— `test_no_activity_is_not_stale` 用新建的
> tracker（`_last_edit` 是 0），第二個條件先短路，`has_activity` 那半邊從沒被驗到。
> 補上 `_last_edit` 的前置條件後才真的守住。
> **「反證抓到空守門」正是它存在的理由。**

## 1.7.11 (2026-09-04)

### 🔴 `allowed_targets = []` 的註解與行為相反（不改行為，只讓語意誠實）

TG 問題單提「manager 派工無目標白名單」。**查證後那個需求不成立** ——
`/api/send` 的 role 路由規則（`api.py:709`）一直在擋，實測：

```
qa-agent(worker) → admin-agent
403「qa-agent（worker）不能直接發給 admin —— 請改用 send_to_instance
    發給 leader-agent（你的 leader）由其轉呈，或用 log_to_leader 私下回報」
```

| src role | 可發給 |
|---|---|
| admin | 任何人 |
| **manager** | **只能** admin / manager / leader |
| worker | leader 永遠可以；worker↔worker 看 p2p；**admin 403** |

> 問題單描述的死鎖（manager 派給 admin → 跟著卡死）成立，但那條路徑是
> **刻意允許**的 —— manager 是私訊入口，需要能把維運請求轉給 admin。
> 死鎖的正解是 hang 偵測 + 逾時（已有），不是移除合法的通訊路徑。

### 但查證挖到一個真缺陷

```python
allowed_targets = []   # workers can only reply, no send targets     ← 註解
allowed_targets = []   # Private manager: only sees itself           ← 註解
                       ↓
if not _allowed_targets:
    return None        # no restriction                              ← 實際
```

`None` 與 `[]` 都走「無限制」，於是 daemon 想表達「不能發給任何人」時
**得到的是相反的效果**。實測 5 個 agent 的 `mcp.json`：只有 leader 有
`--allowed-targets`（`backend.py` 的 `if cfg.allowed_targets:` 對空 list
是 falsy → 不傳）。

**兩層防護裡有一層是假的，而註解讓讀的人以為它在保護。**

### 修法：三種狀態分開

| 值 | 意思 |
|---|---|
| `None` | 無限制 |
| `[]` | **不得發給任何人** |
| `[...]` | 只能發給清單內的 |

⚠️ **不改「部署」的行為**：原本傳 `[]`（實際＝無限制）的地方改傳 `None`（無限制），
效果完全相同。`[]` 從此保留給「全禁」，目前無人使用。

> 🔴 **但「模組預設值」變了，而測試依賴它** —— 實測 9 個測試紅：
> 8 個用 `_allowed_targets = []` 表達「無限制」（其中一個還在 `teardown_method`
> 裡設，污染了後續測試），1 個斷言 `KiroBackendConfig.allowed_targets == []`。
> 全部改成 `None`，並在 `TestCheckTarget` 補一條專測 `[] = 全禁`。
>
> 💡 **宣稱「不改行為」時要說清楚是誰的行為** ——
> 部署的、API 的、還是模組預設的。這三者不同，
> 而測試通常釘的是最後一種。

同步四處：`team_mcp`（判斷 + argparse default + `tools/list` 過濾）、
`backend`（型別 + **`is not None` 而非 truthy** —— truthy 會把「全禁」的
空 list 當成沒設定而漏傳）、`daemon`、`cli`（第二份實作）。

### 守門

10 條，其中一組專門證明**行為沒變**（daemon 不得傳 `[]`、cli 與 daemon 一致、
backend 用 `is not None`）。反證 5 項各紅 1 條，包含一條釘住
「真正在擋的是 `/api/send`」—— 若日後有人拿掉那段，這條會紅，
而那時 team_mcp 這層（設成無限制）擋不住任何東西。

實機驗證行為不變：worker→admin 仍 403、worker→leader 仍通、
`mcp.json` 與 1.7.10 一模一樣。

## 1.7.10 (2026-09-04)

### 🔴 headless 部署的 reply 從 1.7.0 起每天落空 —— 而症狀是「看不見」不是報錯

`mode: headless` 的部署（`team.py:584` 跳過整個 TG 區塊）**永遠不會有** TG adapter。
1.7.0 之前那不是問題：排程啟動綁在同一個 TG 分支裡，headless 部署的排程
**根本不跑**，所以沒有人會叫 agent `reply()`。

1.7.0 M2 把排程搬出 TG 分支（那本身是對的），於是排程開始叫醒 agent，
而它們呼叫 `reply()` 在沒有 TG 出口的部署上必然落空：

```
no telegram adapter 次數（paddy，唯一的 headless 部署）
  08-25 ~ 08-30    5 次
  08-31（1.7.0）  11 次   ← 排程開始跑
  09-01 / 09-02   11 / 10
  09-03           27 次
```

4 個 job 叫 agent reply（`hourly-check` / `daily-summary` /
`daily-ops-report` / `daily-news`），**連續五天產出全部消失而沒有人發現**。

> 🔴 TG 回報的問題單把根因判成「`team_mcp` 子進程的 api 實例 `self.tg=None`
> （架構層實例隔離）」。**三個實測都推翻它**：
> ① `team_mcp` 的 reply 是走 HTTP 打主進程（`/api/reply`），不是自建實例
> ② `no telegram adapter` 只發生在 paddy（64 次），其他四個部署 **0 次**
> ③ paddy 是唯一 `mode: headless` 的
>
> 它建議的修法（team_mcp 加 TG fallback）對真根因無效 —— 主進程本來就沒有 tg。

### 修法一：headless 的 reply 走 log 出口

```python
if self.headless and not self.tg:
    log.info("📝 REPLY(headless) %s「%s」", instance, text[:200])
    if len(text) > 200:
        log.info("📝 REPLY(headless) %s 全文：\n%s", instance, text)
    return {"ok": True, "channel": "log", "note": "headless 部署無 TG 出口…"}
```

**回 `ok: True` 是刻意的** —— 對 agent 而言它確實完成了回報，
失敗的是「這個部署沒有對外通道」，那不是它能處理的事。
回 error 會讓它以為自己做錯了。而 `channel` 與 `note` 讓它知道實情。

分支排在「等 3 秒重試」**之前** —— headless 永遠不會有 adapter，
等待是白費，而且會讓每次 reply 多花 3 秒。

### 修法二：「headless + 有 job 叫 reply」進 degraded

接到已經有人在看的地方（同 1.7.4 / 1.7.5 的做法），而不是只寫一行啟動 log。
警告同時指出可行的替代（`mode: full`，或讓那些 job 改走檔案／知識庫產出）。

```
degraded: ['headless_no_reply_channel:4 jobs(hourly-check,daily-summary,daily-ops-report)']
```

### 修法三：狀態回退的訊息不再說謊

`_rollback_reply_status` 原本寫死「⚠️ reply 失敗，狀態回退」——
而 headless 走的是**成功路徑**，看 log 的人會以為訊息遺失了。

改成帶 `reason` 參數；真正的失敗仍說「失敗」，不把所有訊息軟化掉。

```
舊：⚠️ reply 失敗，狀態回退：qa-agent awaiting_reply → RUNNING
新：⚠️ headless 無 TG 出口（內容已入 log），狀態回退：qa-agent awaiting_reply → RUNNING
```

> 這正是 2026-09-04 問題單缺陷 #3 在問的那條（「訊息是否遺失？」）。

### 守門

17 條，5 項反證（移除 headless 分支 / 回 error 而非 ok+log / 不印內容 /
用不存在的變數 / 旗標設在 agents 啟動之後）各紅 1-5 條。

> ⚠️ 實作時我第一版寫了不存在的變數 `scheduler_cfg`，
> 而外層的 `except Exception` 會把 `NameError` **靜默吃掉** —— 整段檢查等於不存在。
> 守門因此有一條專門釘「必須用真實的 `load_scheduler_config()`」。

## 1.7.9 (2026-09-03)

### 🔴 回退 1.7.5 的 hang 判準 —— 它誤判了所有正常工作的 agent

1.7.5 把判準從 `last_output` 改成 `last_reply`，理由是
「有輸出不等於有回應」。**那句話沒錯，但拿它當 hang 判準是錯的。**

一個 agent 收到訊息、做完事、判斷不需要回覆（寫檔案、更新知識庫、
或單純沒有要回報的東西）—— 那是**正當行為**，不是掛住。

而 `last_inbound` 在**任何訊息投遞時**都會更新，包括 hang 自己發的喚醒訊息：

```
收到訊息 → 做完不回覆 → inbound > reply 成立
  → 30 分後判 hang → 發喚醒 → inbound 又更新 → 仍不回覆 → 回到開頭
```

**實測（2026-09-03，24 小時）**：

| 部署 | hang 通報 | 實際重啟 |
|---|---:|---:|
| paddy | **44** | **13**（6 auto-restart + 7 wake-retry） |
| aiops | 34 | — |
| director | 7 | — |

而同時查 `/api/instances`：**7 個 agent 的 `last_output` 全是 2-3 分鐘前**
—— 它們一直在產出、活得好好的。

### 🔴 更根本的錯誤是歸因

1.7.5 想抓的案例（aiops/ic-agent 沉默 5 天）**根因不在 hang 偵測**：
是 obs 的告警門檻設成 50 而正常基線是 1500，每 30 分鐘送一次噪音；
ic 判斷「無需行動、不回覆」是**對的**。那個已在設定層修掉（門檻 → 3000 + 相對變化）。

> 💡 **「這個指標抓不到某個案例」不代表該換指標** ——
> 先確認那個案例是不是該由這個指標負責。
> 為了抓一個「其實不是 hang」的案例把判準改嚴，結果誤判所有正常 agent。

**修法**：判準與 idle 參考點**兩處一起**退回 `last_output`
（用不同基準會得出互相矛盾的結論：判準說「還在等」，
idle 計算卻說「它幾秒前才動過」）。

`last_reply` 欄位**保留**（`/api/status` 可見）——
「長期沒有對外回應」是有用的**觀測**訊號，但**不該觸發重啟**。

### 守門

反轉 1.7.5 加的 5 條（它們釘住的是錯的行為），並補一條釘住實測病例：

```python
def test_the_real_paddy_case_is_not_hung():
    st = _FakeState(inbound=now-60,       # 剛被 hang 喚醒訊息更新
                    output=now-180,       # 3 分鐘前還在產出
                    reply=now-6*3600)     # 6 小時沒對外回應
    assert _has_unanswered(st) is True    # 第一層：確實有未回應的請求
    assert now - (st.last_output) < 1800  # 第二層：idle 未超標 → 不判 hang
    assert now - (st.last_reply) > 1800   # 對照：1.7.5 的算法會誤觸發
```

> ⚠️ 第一版測試斷言也錯了 —— 把「有未回應的請求」與「判定 hang」混為一談。
> 實際擋住誤判的是 `idle_seconds` 那層。**兩層一起看才是 hang 的定義**，
> 而 1.7.5 只換了一層。

反證：把判準改回 `last_reply` → 5 條紅。

## 1.7.8 (2026-09-02)

### 🔴 1.7.7 的撞港檢查造成常駐假警報

1.7.7 把 port 檢查放在最前面，於是**明確設了
`dashboard.builtin_page: false` 的部署照樣被警告**：

```
ninja-team 實測（16:01）：
  WARNING website port 28333 已被佔用 —— 跳過網站啟動
```

而它關掉了內建頁、也沒有 `apps/team-website/` —— **根本不需要那個 port**，
卻每次啟動都被警告一次（28333 是它自己的 web dashboard）。

> 🔴 本檔記過：**常駐假警報的代價不是雜訊，是維運開始習慣性忽略該檢查。**
> 而這次的假警報是 1.7.7 自己造成的 —— 那個檢查本身是對的，只是問錯了順序。

**修法**：先問「要不要網站」，才問「port 綁得上嗎」。

```python
_wants_site = _has_custom or config.dashboard.builtin_page
if not _wants_site:
    log.debug(...)                       # 不是異常 → debug 不 warning
elif not _port_bindable(...):
    log.warning(...)                     # 真的要綁卻綁不上 → warning
elif _has_custom:  ...                   # 自訂網站
else:              ...                   # 內建頁
```

守門 4 條（「要不要」必須排在 port 檢查之前 / 不需要網站時只 debug /
`_wants_site` 必須涵蓋兩個來源 / 檢查仍排在兩條啟動路徑之前），
反證 1 項紅 3 條。

> 💡 判準：**一個檢查該在「確定要做那件事」之後才跑。**
> 提前檢查會對「不做那件事的人」也發出警告 ——
> 而他們無法（也不需要）處理它。

## 1.7.7 (2026-09-02)

### 🔴 撞港會讓整個 daemon 死掉 —— 而 1.7.6 讓每個部署都開始綁那個 port

uvicorn 綁不到 port 時呼叫 `sys.exit(STARTUP_FAILURE)`，拋的是
**`SystemExit`** —— 它繼承 `BaseException` 而**不是** `Exception`：

```python
issubclass(SystemExit, Exception)   # False
```

所以 `team.py` 包住整段的 `except Exception` **絕對抓不到它**。
而 `serve()` 跑在 `asyncio.create_task()` 裡，`SystemExit` 會傳播到
event loop → `asyncio.run()` 整個中止 → **daemon 起不來**（實測 exit code 3）。

**1.7.6 把這個從低機率變成真實風險**：在那之前，這條路只在有
`apps/team-website/` 時才走（本機四個部署只有一個）；加了內建頁之後
**每個部署都會綁 `health_port + 5000`**。

而 ninja-team 的 health 是 `23333` → website `28333` ——
**那正是它自己 web dashboard 的 port**（`.env` 的 `WEB_PORT`）。
差一步就會讓它的 daemon 起不來。

> 該注意的是 `assert_no_port_collision()`（1.6.2）擋不住這個 ——
> 它只檢查 `reserved_ports` 裡**宣告過**的 port，沒宣告的撞港它看不到。
> aiops / director / paddy 沒炸是**運氣**（那三個 port 剛好空著）。

**修法**：綁之前先試綁（`_port_bindable()`）。不可用就 `log.warning` 並
**兩條路徑都跳過** —— 自訂網站那條同樣會 `SystemExit`（那是 1.7.6 之前
就存在的缺陷，只是機率低）。

```
website port 38333 已被佔用 —— 跳過網站啟動（daemon 與 agents 不受影響）。
若要換 port 請改 health_port，或設 dashboard.builtin_page: false
```

端到端驗證：故意佔住 38333 再重啟 paddy → **daemon 7/7、degraded 空**，
只留那一條可行動的 warning。

> 用「試綁」而不是「連線看看有沒有回應」：佔用者可能不說 HTTP
> （也可能是別的協定），而我們要問的問題就是「我綁得上去嗎」——
> 直接試綁才是那個問題的答案。

守門 5 條（helper 行為 / 檢查必須排在兩條路徑之前 / 不可用時兩條都跳過 /
必須 log 不得靜默 / `SystemExit` 那個事實要留在註解裡不被當成多餘檢查刪掉），
反證 3 項各紅 1-3 條。

## 1.7.6 (2026-09-02)

### 🔴 dashboard 後端閒置三個月 —— 因為它的前端在一個 daily commit 裡被無聲刪掉

`dashboard_api.py`（298 行、10 個端點：stats / agents / board / timeline /
costs.trend / autopilots）從 2026-05 起就完整可用。掃過全機 —— **零消費者**。
連自己寫了 37 個檔網站的 nana-team 都沒用它。

追下去發現原因：它的前端 `ark_team_webbot` 在 `a39f22cb`
（2026-05-17，**1649 檔 / +325,906 行**的 daily commit）裡被刪除，
而那個 commit：

| | |
|---|---|
| 訊息 | `chore(daily): 2026-05-16 Skills expansion (48 skills) + ToolTracker...` |
| 提到 webbot | **一個字都沒有** |
| CHANGELOG | **沒有記載** |
| 同時做的事 | 給 5 個測試檔各加一行 **`pytest.importorskip("ark_team_webbot")`** |

> 🔴 **刪除實作的同一個 commit，順手讓測試不會變紅。**
> 這比單純刪掉更難發現 —— 測試從此永遠是綠的，
> 在全量 2547 passed 裡看起來完全正常。而 CLI 的 `webbot` 子命令留著，
> 執行即 `ModuleNotFoundError`，三個月沒有人碰到。

### 新增：內建唯讀儀表板（零設定）

`apps/team-website/src/main.py` **不存在時**，套件在 website port
（`health_port + WEBSITE_PORT_OFFSET`）起一個單檔唯讀頁，
打自己的 `/api/dashboard/*`。

- **有自訂網站的部署完全不受影響** —— 內建頁只在「本來什麼都不會起」時接管
- **單檔內嵌**：無 npm、無 build、無 CDN。消費端是 daemon，
  不該因為一個狀態頁而長出前端工具鏈，且部署環境不保證能連外
- **綁 `127.0.0.1`**，與既有 team-website 一致

**唯讀是設計邊界，不是偷懶**：`dashboard_api` 有 `POST /board`、
`DELETE /board/{id}`、`POST /autopilots/{name}/trigger`，而 daemon API
**目前沒有任何認證**（靠綁 127.0.0.1）。把無認證的寫入操作放上瀏覽器
是另一個層級的決定。守門會檢查頁面實際打的每個端點都在唯讀白名單裡。

`dashboard.builtin_page`（預設 `true`）是回退出口。
**預設開是刻意的** —— 它不改變任何既有部署的行為（有自訂網站的照舊，
沒有的那個 port 原本就是空的），而 opt-in 的下場本檔記過太多次：沒有人會去開它。

### 移除：webbot 殘留

- `cli.py` 的 `webbot` 子命令 + `cmd_webbot_api` / `cmd_webbot_scheduler` /
  `cmd_webbot_start` + 對 `ark_team_webbot` 的 import（約 2000 bytes）
- `tests/test_webbot_*.py` ×4（測的是不存在的模組）
- `DEFAULT_WEBBOT_PORT` **保留但標註已無使用者** —— 3030 是這個生態系用過的
  port，刪掉會讓日後有人重新分配它而不知道歷史

### 🔴 兩個「守門全綠而實機是壞的」錯（同版修掉）

實機起來之後才看到的：

| # | 錯 | 為什麼守門看不到 |
|:-:|---|---|
| ① | `team.py` 傳 `config.name`，而 **`TeamConfig` 沒有這個欄位** | 守門測 `render()` 本身（直接傳字串），沒測 team.py 的呼叫。而 `except Exception` 把 AttributeError 降級成 warning → 服務照常起 7/7，只有看 log 才發現頁面沒起來 |
| ② | 頁面讀的欄位名是**憑印象猜的**（`last_task` / `crash_count` / `st.cost.today` / `d.usd`） | 頁面照樣渲染、HTTP 200，那幾欄一律顯示「—」。**沒有錯誤、沒有 log，只是資料是空的** |

逐一 `curl` 取回實際結構後對齊：

```
/agents      → name status role display_name last_activity
/stats       → running_agents total_agents cost_today_usd tasks{total,...}
/board       → id title assignee status priority created_at completed_at
/timeline    → timestamp instance type detail source
/costs/trend → date total_usd by_instance（+ note）
```

補三類守門（共 23 條）：
- 呼叫點的每個 `config.X` 必須是 `TeamConfig` 真有的欄位
- 頁面讀的欄位名必須在 `dashboard_api.py` 出現過
  —— **端點改名時會紅，而不是靜默變成一排「—」**
- `EXPECTED` 清單裡的欄位頁面真的要用到（避免它變成過時的裝飾品）

> 🔴 兩個錯的共同形狀：**測試 fixture 把「輸入從哪來」也一起假設掉了。**
> 本檔記過同型（1.2.19 的代號推導：單元測試全綠但實機全錯）。
> 判準：**測「產生器」時，要連「呼叫它的人傳什麼」一起測。**

守門 18 條，反證 6 項（加寫入呼叫 / 打白名單外端點 / 引用 CDN /
內建頁蓋掉自訂網站 / 預設改 opt-in / webbot import 復活）各紅 1-2 條。

## 1.7.5 (2026-09-02)

### 🔴 hang 偵測看的是「有沒有輸出」，而那不等於「有沒有回應」

1.6.9 把 hang 的定義從「它很久沒說話」改成「我問了它沒答」，
判準是 `last_inbound > last_output`。**但 `last_output` 記的是
process 的 stdout 行數有沒有增加** —— agent 思考、讀檔、呼叫工具
全都會產生 stdout。

於是一個「一直在忙、但從不回應任何請求」的 agent：

```
last_output  一直前進  →  inbound > output 永遠不成立  →  hang 偵測完全看不到它
```

實測病例（aiops / ic-agent）：

| 觀測 | 值 |
|---|---|
| 最後一次對外回應 | 2026-08-28 18:44 |
| 期間 `peer-reply timeout` | **142 次**（obs → ic） |
| `/api/health` | **9/9 running、degraded 空** |
| hang detector 命中 | **0** |
| 它真正的排程任務（daily-summary） | 同期 **4 次靜默失敗** |

**修法**：新增 `InstanceState.last_reply`（真正呼叫過對外通訊工具的時間），
`_has_unanswered_request` 與 hang 的 idle 參考點 `_hang_ref` **兩處一起**
改用它。只換其中一處無效 —— 判準會成立，但 idle 秒數仍由 stdout 決定。

`Daemon.mark_reply()` 是唯一寫入點，由 `api.py` 的**五個**對外通訊端點呼叫
（`/api/reply`、`/api/send`、`/api/reply-photo`、`/api/reply-file`、`/api/log`）。
第一次實作只接了前兩個 —— 守門因此明確斷言呼叫點數量。

> ⚠️ **行為變更**：一個收到請求後長時間不回應的 agent，現在會在
> `hang_detector.timeout_minutes` 後被通報（先前不會）。
> 這正是本版要修的東西，但若某個部署有「刻意長時間沉默工作」的 agent，
> 升級後會看到新的 hang 通報 —— 那是正確的訊號，不是回歸。
> 有 `auto_restart: true` 的部署請先確認門檻夠長。

### 🟠 同一對 agent 重複逾時進 degraded

`peer-reply timeout` 只 nudge**等待方**叫它重試，於是形成迴圈：

```
A 發問 → B 不回 → 逾時 → nudge A「請重試」→ A 重發 → 回到開頭
```

7 天 142 次，而沒有任何一層認為這是異常。現在同一對 `(sender → target)`
累積 `PEER_TIMEOUT_DEGRADE_AT`（3 次 ≈ 45 分鐘）就進 `degraded`，
接到已經有人在看的地方（同 1.7.4 的做法）。

對方一旦回應，`mark_reply()` 會清掉等它的計數與標記 —— **自癒是刻意的**：
degraded 反映當下狀態，不是歷史事件的累積。

## 1.6.9 (2026-08-28)

### 🔴 hang 的定義錯了 —— 「很久沒說話」不等於「掛住」

1.6.8 修掉緊迫迴圈（單一 agent 每 35 秒）之後，剩下**每 30 分鐘一次的整批重啟**：

```
18:18:05  leader-agent  idle for 1811s → auto-restarting
18:18:12  ai-dev-agent  idle for 1818s → auto-restarting
18:18:17  coder-agent   idle for 1823s → auto-restarting
18:18:22  qa-agent      idle for 1828s → auto-restarting
18:18:27  data-agent    idle for 1833s → auto-restarting
```

**7 個 agent 全部 `status=running`、同時觸發，而沒有一個是壞的** —— 只是沒工作。

#### 根因

hang 偵測只問「`last_output` 多久沒前進」。於是**從沒被派過工的 agent 必然中招**：

```
啟動 → 產出橫幅 → 沒人找它 → last_output 不再前進
     → 30 分鐘後判定掛住 → auto_restart 重啟
     → 又停在新的啟動時刻 → 每 30 分鐘一次，永遠
```

⚠️ **`IDLE` / `AWAITING_REPLY` 擋不住這個** —— 那兩個狀態只在
**完成過任務之後**才會被設，而受害者是**從沒收過任務**的 agent，永遠停在 `RUNNING`。

實測 paddy：7 天累計 **3285 次** auto-restart。而每次重啟都帶 `--resume`，
kiro-cli 在 resume 時自己會發一則
「In a few words, summarize our conversation so far.」並取得完整回覆
→ 一來一回追加進 history。**這就是 context 持續膨脹的主幹。**

#### 修法：補上第三個時間戳

1.4.7 把 `last_activity`（inbound ∪ output）拆出 `last_output`（只有 output），
解掉「一直收訊息但從不回覆的 agent 永遠不被判 hang」。
**但那次只做了一半 —— 反過來的情況沒解。**

新增 `InstanceState.last_inbound`（只由「我們真的寫進 stdin」更新），
於是問得出真正的問題：

```python
def _has_unanswered_request(state) -> bool:
    """hang 的定義是「我問了，它沒答」，不是「它很久沒說話」。"""
    inbound = state.last_inbound or 0
    if not inbound:
        return False          # 從沒送過東西給它 → 它沒有義務說話
    return inbound > (state.last_output or 0)
```

三處判定（hang 通知／escalation／wake-retry restart）各加這一道，
漏一處就會從那裡漏出去（守門用 `ast` 數呼叫節點，要求恰好 3 處）。

#### 效果

| | auto-restart / 小時 |
|---|---:|
| 1.6.7（修正前） | 26–81（典型 ~50） |
| 1.6.8（修掉緊迫迴圈） | ~14 |
| **1.6.9（本版）** | **預期 ~0**（閒置不再觸發） |

### 守門

`tests/test_restart_loop.py` 12 條。反證：判準恆真 → 3 紅；
`inbound == 0` 當成掛住 → 2 紅；只加一處 → 1 紅；欄位不記錄 → 1 紅。

> 💡 最後一條特別重要：**只加欄位不記錄的話，判準永遠回 False → hang 偵測整條失效**
> —— 那比誤判更糟（真的掛住也沒人管）。守門要求 `send_input` 與
> `last_inbound` 更新一一對應。

## 1.6.7 (2026-08-28)

收斂通報的三個不一致（1.6.6 盤點時列出的）。

### 🔴 ① `notify_recovery` 完全不受 `notifications` 控制

設 `notifications.startup: private` 只擋得住啟動橫幅，
**恢復通知照樣無條件進 General** —— 在有人聊天的群裡，
agent 每次崩潰重啟都跳一次「已恢復」。

> 🔴 `NotificationsConfig` 的 docstring 寫
> 「這些是服務自己發的訊息：**啟動橫幅、重啟通知**」——
> **「重啟通知」寫在說明裡，但實作只涵蓋了啟動橫幅。**
> 說明比實作寬，而讀設定的人會照說明去設。

新增 `notifications.recovery`（`both`／`group`／`private`／`off`），
**預設 `group` = 維持 1.6.6 行為**，既有部署零影響。
順帶讓它支援私訊（原本只有 General 一條路）。

### 🔴 ② 收件人有三套推導

| 通報 | 原本問誰 |
|---|---|
| L3 拍板卡 · `notify_paddy` | `_owner_chat_id()`（admin 優先） |
| 掛起升級 | 掃 `private_chat_map` 找 admin **或 manager** |
| **新版本通知** | **`allowed_users[0]`** |

第三套與前兩套的**對象型別都不同**（TG user id vs 綁了 private_chat 的
instance）—— 同一個部署裡它們可能指向不同的人。

`notify_new_version` 改用 `_owner_chat_id()`，
**保留 `allowed_users` 當最後備援**（純私訊部署可能沒有任何 instance
綁 private_chat，那時它是唯一線索）。

掛起升級的 admin+manager 是**刻意的**（升級要給更多人），維持不變。

### 🔴 ③ `owner_chat_id` 有兩份實作 + 兩份狀態

`api.DaemonAPI.owner_chat_id()` 與 `TelegramAdapter._owner_chat_id()`
邏輯一字不差。更麻煩的是狀態：

| 誰 | `_private_chat_map` 從哪來 |
|---|---|
| `DaemonAPI` | `team.py` 從 `config.instances` 建的 dict |
| `TelegramAdapter` | 逐一 `bind_private_chat()` 綁進去 |

內容目前一致（同一來源），但**執行期動態綁定只會更新 adapter 那份**。

`api` 改為委派給 adapter；headless（沒有 TG）時才用自己那份。

### 守門

`tests/test_notify_consistency.py`（15 個）。
**反證**：`notify_recovery` 拿掉開關 → **3 紅**；
`notify_new_version` 改回 `allowed_users[0]` → **2 紅**；
`api.owner_chat_id` 不委派 → **1 紅**。

## 1.6.6 (2026-08-27)

系統通報的第三種去向：**哪裡都不到**。

### 🔴 `_notify_to_instance_channel` 的第三段原本什麼都不做

```
① private_chat → 私訊
② 自己的 topic  → topic
③ 兩者都沒有   → 什麼都不做（不寫 log、不退回）
```

呼叫者全是**成本通報**（`notify_cost_warn` / `notify_cost_pause` /
`notify_cost_resume`），而實測兩個部署都命中第 ③ 條：

| 部署 | instances | 有 `private_chat` | 有 `topic_id` |
|---|--:|--:|--:|
| paddy | 7 | **1** | **0** |
| hoyeah | 15 | **1** | **0** |

**其餘 6／14 個 agent 的成本警告從來沒送出去過。**

> 🔴 「成本已達上限、agent 被暫停」是最不該無聲的一類通報。
> 而 `notify_hang` 另有一條「掃 admin/manager 的 private_chat」的保險，
> **成本通報沒有** —— 這個不對稱就是補 fallback 的理由。

### 🔴 同場加映：`group` 的判準原本只有一半在用

`api._resolve_output_topic()`（`reply()` 用的）**會看 worker 的 `group`**
→ 回覆進所屬 leader 的 topic。而 `_notify_to_instance_channel` **不看** ——
於是同一個 worker 的**回覆進組 topic、成本通報卻落到 General**。

實測 hoyeah：15 個 instance 裡 **9 個 worker 設了 `group`**，全部受影響。

> 🔴 **同一件事兩套實作必然漂移**（本套件記過多次）。
> 這裡沿用 `_resolve_output_topic` 的判準：`role == "worker"` 且有 `group`。

### 修法：五段出口

```
private_chat → 自己的 topic → 組長的 topic（worker）→ General（標明來源）→ log.error
```

兩個實作細節：

- **送失敗時不 `return`** —— 原本私訊送失敗只 `log.warning` 然後 `return`，
  通報一樣消失。現在往下退到 topic／General。
- **退回 General 要標明來源**（`⚠️ [📊 數據 無專屬出口]`）——
  否則群組裡會出現一則不知道在講誰的成本警告。
- 連 `group_id` 都沒有（純私訊部署）→ **`log.error`**，不再靜默。
  那是設定問題，要看得見。

### 守門

`tests/test_notify_fallback.py`（13 個）：四段出口各一、退回要標來源、
私訊失敗要往下退、全失敗要 log、三個成本通報都走同一條路（`ast` 驗）。

**反證**：拿掉 General 段 → **5 紅**；私訊失敗後 `return` → **3 紅**；
退回不標來源 → **1 紅**；
拿掉 `group` 解析 → **1 紅**；非 worker 也借用組長 topic → **1 紅**。

## 1.6.5 (2026-08-27)

`reply(topic_id=N)` —— 顯式指定送到哪個 Telegram topic。

### 🔴 起點：參數被靜默忽略，訊息去了看起來合理但不對的地方

排程 prompt 寫 `reply(text, topic_id=4)` 想推公告到手動建的 topic，
訊息卻沒出現在那裡。查證後三層都對不上：

| 層 | 實況（1.6.4 及之前） |
|---|---|
| `reply` 的 schema | 只有 `text`／`kind`／`style`／`template`，**沒有 `topic_id`** |
| 會被 schema 擋嗎 | ❌ 沒宣告 `additionalProperties: false` → **放行，但 handler 不讀** |
| 訊息去向 | `_resolve_output_topic()`：worker 有 `group` → **送到所屬 leader 的 topic** |

> 🔴 **最糟的組合：schema 不擋、handler 不讀、而訊息確實送達了某個地方。**
> 使用者看不到訊息，也不知道它去了哪。

### 設計

**① 顯式 `topic_id` 優先序最高**（蓋得過 origin）

顯式指定是呼叫端的決定，套件不該猜。但互動情境下它會蓋掉
「回使用者問的地方」（那多半是誤用），所以覆寫時留 `log.info` 可追。

**② 🔴 白名單擋在送出之前**

```
允許 = _topic_map.values() ∪ channel.announce_topics ∪ {general_topic_id}
```

**不能只用 `_topic_map`** —— 手動在 TG 建的公告 topic 根本不在裡面，
而那正是這個功能的主要用途。

> 💡 這讓 1.6.3 的 `announce_topics` 語意更完整：
> 「只出不進」→「只出不進，**且可被顯式指定為出口**」。
> **一份設定同時表達「不收訊息」與「可以送過去」，語意一致。**

不在集合裡 → **拒絕並回可行動的錯誤**（列出可用的 + 告訴人去哪設定），
而不是送出去之後失敗。**打錯數字時 agent 當場看得到。**

**③ 不傳就完全維持舊行為** —— `topic_id: int | None = None`。

### 相容性：零影響

| 情境 | 結果 |
|---|---|
| 不傳 `topic_id` | 行為完全不變 |
| 舊版 daemon 收到帶 `topic_id` 的請求 | ✅ Pydantic **忽略未知欄位，不拋例外**（已實測） |
| 新版 daemon + 舊版 MCP | 不傳就是舊行為 |

`team_mcp` 只在**有值時**才把 `topic_id` 放進 payload，
舊 daemon 收不到多餘欄位。

### 守門

`tests/test_reply_topic_id.py`（14 個）：schema 有宣告、**handler 真的傳下去**
（1.6.4 就是漏在這）、白名單三個來源、顯式排在 origin 之後、
檢查在送出之前、錯誤訊息可行動、覆寫留 log、`group_leader` 被清掉、舊版相容。

**反證**：拿掉優先序段落 → **5 紅**；白名單漏掉 `announce_topics` → **2 紅**；
handler 不傳（回到 1.6.4 的缺陷）→ **2 紅**。

## 1.6.4 (2026-08-26)

三個獨立回報，一起收進本版：進度條顯示半句系統指令、啟動訊息漏版本號、
decider 回完查詢卡 AWAITING_REPLY 觸發假的「決策者卡住」告警。

### 🔴 修：系統注入的提詞污染進度條摘要（`last_task`）

使用者在 TG 看到進度條的 `📋` 行顯示「【系統掛起偵測】你已閒置…2. 有任」後被硬切。
根因鏈三環：① hang 提詞用 `send_message` 發給 agent，預設 `source="user"`；
② daemon 記成 `state.last_task[:60]`，斷在「2. 有任」（剛好 60 字元）；
③ ToolTracker 拿 `last_task` 當進度條摘要顯示。

`last_task` 的語意是「使用者交辦的任務」，但系統自動注入的提詞（hang 喚醒、
掛起偵測）也走同一條 `send_message` → 被誤當任務摘要。

修法：`source in {"system","scheduler"}` 時**不更新 `last_task``；顯示上限
`60 → 100`，超出補「…」讓截斷處有視覺訊號。兩個 hang 提詞標成 `source="system"`。

### 修：啟動訊息「部分失敗」分支漏版本號

「全員到位」那則啟動訊息早有 `📦 ark-team-agent v…`，但**「部分失敗」那則漏掉**
（AIOps 在更舊的 1.4.11 兩則都沒有，才會回報成整則缺）。把版本計算提到分支之前
（hoist），兩則共用同一 `_ver_line`，與 `/status` 的「主程式 v{ver}」對齊。

### 🔴 修：decider 回完一次性查詢卡 AWAITING_REPLY → 假的「決策者卡住」

decider（leader/admin）回答完使用者的一次性查詢後，`reply()` 因來源是 `user`
而標成 AWAITING_REPLY 等接話。但查詢類對話使用者常不會再回 → 停到 30 分鐘被
decision rule 4（decider-stuck）誤報「決策者卡住」。

它**沒有卡** —— 只是這輪對話結束、沒有待辦。修法（根治）：新增
`hang_detector.awaiting_idle_minutes`（預設 20m，**必須 < Rule 4 的 30m**），
AWAITING_REPLY 無 inbound 超過此值 → 降為 IDLE。IDLE 不被 Rule 4 觸發
（它只看 AWAITING_REPLY），且仍在 `send_message` 可投遞集合裡（新訊息拉回 RUNNING）。

與既有 `_awaiting_safety_net_due`（2× escalation、拉回 RUNNING、防死鎖）互補：
本層門檻近、降為 IDLE、治誤報。抽成 `_awaiting_idle_degrade_due()` 純判斷便於測試。

## 1.6.3 (2026-08-26)

公告頻道（只出不進）+ 修 `_last_source` 污染。

起點是一個問題：「topic 3／4 保持無主，使用者在那邊發訊息會到 General
（admin 處理）—— 現在的架構可以不處理對話訊息嗎？」
實測推翻了原判斷的一半，並挖出一個更嚴重的缺陷。

### 🔴 修：`_last_source` 只寫不清 → 排程輸出被使用者的提問綁架

出口有兩條，而先前**只有一條有防護**：

| 路徑 | 定址依據 | scheduler/system 清除 | 受 `reply_to_origin` 控制 |
|---|---|---|---|
| `reply()` 工具（`api.py`） | `_origin_topic` | ✅ 1.4.1 起 | ✅ |
| **agent stdout（`_poll_output`）** | **`_last_source`** | ❌ **沒有** | ❌ **無條件優先** |

後果：使用者在某個 topic 問一次，`_last_source[instance]` 就被寫死，
**之後該 agent 的每一次 stdout 輸出（含排程日報）都跑到那個 topic**。

這正是 1.4.x 設計 origin 傳遞時明講要防的「上午問過一次，之後每天日報
都塞進那個 topic = 污染」—— **當時只做了一半**。
本套件記過：**收落點要寫入端與讀取端一起收。**

修法沿用同一條判準（`source in ("scheduler", "system")` = 自主工作、沒有人在等），
但**刻意放在 `_origin_routing_on()` 之外** —— 讀取端本來就不看那個開關，
照抄旁邊的寫法擺進 `elif` 分支的話，`reply_to_origin` 預設 `False` → **永遠不執行**。

### 新增：`channel.announce_topics` —— 公告頻道（只出不進）

```yaml
channel:
  announce_topics: [3, 4]     # 排程可推送；使用者發訊不派工、不回覆，但仍寫進 session
```

語意：**入口關閉、出口照常**。

- 使用者在裡面發訊 → 走既有的 `_record_group_only()`：不回覆但**仍寫記憶**
  （對齊 1.4.0：群組是資訊來源，不回話是禮貌，不聽是浪費）。
  **刻意不另寫一條路** —— 同一件事兩套實作必然漂移。
- 排程／agent 的 `reply(topic_id=N)` **完全不受影響**
- 攔截點排在**指令之後**（在公告頻道打 `/status` 仍有效）、
  **`group_policy` 之前**（「這個 topic 不收訊息」比「這則值不值得回」更早）

#### 🔴 為什麼是白名單，不是 `unowned: ignore` 這種 policy

「所有無主 topic 一律忽略」看起來更省事，但它會把
**「設定漏綁」偽裝成「刻意忽略」** —— 新增 agent 忘了綁 topic 時，
那個 topic 的訊息被靜默吞掉，沒有任何訊號。本套件記過太多次這種形狀。

沒列進白名單的無主 topic 仍照舊 fallback，只是 `_resolve_instance()`
**現在會寫 log**（先前完全沒有訊號）：公告頻道走 `debug`（預期內），
其餘走 `info`（值得注意 —— 那多半是漏綁）。

### 相容性：對既有部署零影響

`announce_topics` 預設空 list；`_last_source` 的清除只在 `source` 是
`scheduler`／`system` 時觸發，使用者互動不受影響（回覆仍跟隨提問的 topic）。

### 守門

`tests/test_announce_topics.py`（17 個）。反證：`is_announce_topic` 永遠回
`False` → 2 紅；清除移回 origin gate 之內 → 1 紅；`team.py` 不傳設定 → 1 紅。

順帶修 `test_gate_has_single_decision_point`：它用 `src.count()` 純字串計數，
**任何在註解裡提到 `_origin_routing_on()` 的人都會讓它變紅**（本次就踩到）。
改用 `ast` 數 Call 節點。本套件記過：**常駐假警報的代價不是雜訊，
是維運開始習慣性忽略該檢查。**

## [1.6.2] — 2026-08-25 · ✅ Release

port 規則集中到 `constants.py`，衍生規則給唯一計算出口，並**接上啟動撞港檢查**。

> 📌 開發過程中一度標為 1.6.1（只做常數收斂），但沒有 build 也沒有發版。
> 接上啟動檢查後併成單一 **1.6.2** —— **留一個從未存在的版號比跳號更誤導**。

**Patch** —— 修的是寫死與打架的預設值，沒有新行為（新增的兩個 helper 是把
既有規則搬到單一出口，不是新功能）。

### 🔴 `website = health_port + 300` 是寫死在使用點的魔術數字

`team.py` 直接寫 `config.health_port + 300`，只留一行註解。
**改 `health_port` 的人看不到這個隱形連動。**

實際踩到：本機把 PC 團隊服務遷到 `2xxxx`，director 的 health_port 改成 23033
→ 衍生的 website 變成 **23333**，正好等於 ninja-team 的 port。

沒真撞只是因為 director 沒有 `apps/team-website/src/main.py`，website 根本
不啟動。**那是運氣，不是設計保證** —— 哪天 director 補上 website 就會撞，
而撞港的症狀是**執行期靜默搶不到 port**，不是啟動報錯。

### 🔴 `health_port` 預設有兩個打架的值

| 位置 | 值 |
|---|---|
| `constants.DEFAULT_TEAM_AGENT_PORT` | 13030 |
| `cli.py` 產 mcp.json 時 | **33300** |
| `team_mcp.py` 的 `except ImportError` 分支 | 又一次 13030 |

`team.yaml` 沒寫 `health_port` 時，產出的 mcp.json 會指到 33300 而 daemon
實際綁 13030 → **mcp 連不上，而且不報錯**。

`team_mcp` 那個 fallback 是死碼 —— 三個啟動點（`mcp.json` / `cli.py` /
`backend.py`）全都是 `python -m ark_team_agent.team_mcp`，相對 import 不會失敗。

### 收斂後

```python
# constants.py —— port 規則的唯一定義處
WEBSITE_PORT_OFFSET: int = 300
CROSS_TEAM_PORT_DEFAULT: int = 23031

def website_port_for(health_port: int) -> int: ...
def assert_no_port_collision(health_port, *, reserved=frozenset()) -> None: ...
```

`reserved` 由**消費端**傳入 —— 套件不硬編別部署的 port（23333 是 ninja 的
事實，不是套件的）。有一條測試專門釘住這點。

使用點六處改接出口：`team.py` 的 `+300`、`cli.py` 的 `33300`、
`api.py` 的 `"23031"`、`team_mcp.py` 的重複預設、`cli.py` 兩處把
webbot port 寫進 help 文字。

### 接上啟動檢查（`team.yaml` 新增 `reserved_ports`）

光有 `assert_no_port_collision()` 沒有用 —— 本套件記載過最常見的缺陷形狀
就是「建了但沒人呼叫」（`ModeRouter.route()` 有 213 個測試全綠卻零呼叫點）。
現在 `team.py` 在 **`daemon_api.start()` 之前**呼叫它：

```yaml
# team.yaml
health_port: 23033
reserved_ports: [23333]      # 同機其他服務（ninja-team）
```

- **由消費端提供，套件不硬編** —— 23333 是 ninja 的事實，不是套件的
- **留空 = 不檢查** —— 既有部署升級後行為完全不變，直到自己填
- 設定寫壞（非 list／含非數字）只警告不中止 —— 但**會留下 log**，
  靜默忽略正是這類設定長期失效的原因

### offset 從 300 改成 5000（🔴 會改變既有部署的 website port）

`+300` 讓 website 落在**與 health 相同的頻段**（23030 → 23330），
所以它可能撞到另一個服務的 health port —— director 的 23033 → 23333
正好是 ninja 就是這樣來的。**換 offset 是結構性解法，不是把那一格挪開。**

| 服務 | health | website（舊 +300）| website（新 +5000）|
|---|---|---|---|
| nana-team | 23030 | 23330 | **28030** |
| director | 23033 | **23333 ⚠️ = ninja** | **28033** |
| aiops | 23036 | 23336 | **28036** |

health 走 `23xxx`、website 走 `28xxx`，後三碼不變好對照。

⚠️ **nana-team 實測原本就在 23330 上聽** —— 升級重啟後會移到 28030，
反向代理／書籤／防火牆規則要一併更新。

### 守門 22 個 —— 兩條初版是空的，都是反證抓到的

**① 掃描器把字串常值一起剝掉**，而 port 預設常寫成字串
（`os.environ.get("CROSS_TEAM_PORT", "23031")`）→ `api.py` 那條**永遠不會紅**。
退回四處舊寫法時只失敗三條才發現。改成「剝註解與 docstring、保留一般字串」。

**② 驗接線用名字比對**（`"assert_no_port_collision" in code`）——
拿掉呼叫之後 `import` 那行還在，名字照樣命中。
改用 `ast` 找 **Call 節點**，並驗它的行號在 `daemon_api.start()` 之前。

> 🔴 **兩條都不是「測試沒寫」，是「測試寫了但驗不到東西」。**
> 而它們一路都是綠的 —— 只有反證會讓空的守門現形。
> 通則：**新增守門一定要退回舊行為確認它會紅**，綠燈本身不是證據。

另有一條驗**性質**而非數字的：`test_offset_moves_website_out_of_health_band`
—— 只釘 `offset == 5000` 擋不住有人改成 3000（23030 → 26030，還在 2xxxx）。

---

## [1.6.0] — 2026-08-24 · ✅ Release

啟動時檢查有沒有新版，有的話在 TG 給一張帶按鈕的卡片一鍵更新。

**這是新功能 → minor**（前面 1.5.x 全是 bug 修復與行為保持的 refactor）。

### 為什麼需要：pin 說明不了實況

實測本機九個部署的 `pyproject.toml` pin 對照實際跑的版本：

| 部署 | pin | 實跑 |
|---|---|---|
| game-team-agent | 1.1.13 | **1.5.2** |
| fish | 1.2.19 | — |
| slotverse | 1.2.21 | — |
| slot | 1.4.8 | — |
| paddy | 1.2.21 | **1.6.0** |

**pin 是下限，說明不了實況**，而實際版本只有去打 `/api/health` 才知道。
於是「我這台是不是舊的」沒有人答得出來 —— 直到出事才發現少了某個版本
才有的保護（1.5.1 的 wake-retry 與 context-reset，1.4.x 完全沒有）。

### 版本檢查（`version_check.py`）

啟動時查 Release API，結果進 log 與啟動橫幅：

```
📦 ark-team-agent v1.5.6 ⚠️ 有新版 v1.6.0
📦 ark-team-agent v1.6.0（已是最新）
📦 ark-team-agent v1.6.0                    ← 查不到時只顯示目前版本
```

三個設計約束，每一條都有測試釘住：

| 約束 | 為什麼 |
|---|---|
| **絕不阻塞啟動** | 網路慢、DNS 壞、repo 打不通都不該讓服務起不來。3 秒逾時 + 吞掉所有例外 |
| **絕不每次啟動都打 API** | 服務會頻繁重啟（改設定、hang 重啟、context-reset），版本不會。結果快取一天 |
| **沒 token 就安靜跳過** | 發版 repo 是 private，`GITHUB_TOKEN` 沒設不是錯誤，是這台沒有查詢權限 |

有新版時用 **WARNING**（維運通常只看 WARNING+），其餘走 INFO 不製造噪音。
`ARK_VERSION_CHECK=0` 可完全關閉 —— **關掉時不發出任何網路請求**
（不是只把結果藏起來；有些部署沒有對外出口，逾時 3 秒乘上重啟次數是實際成本）。

> 🔴 **查不到 ≠ 過期。** `latest` 是 `None` 時 `outdated` 回 `False` ——
> 否則網路不通的部署會每次重啟都跳一次假通知。

### 一鍵更新（`updater.py` + TG 卡片）

有新版 → 私訊管理者一張卡片 → 按鈕確認 → 下載、驗證、安裝、重啟。
**只送私訊不送群組**（升級是維運決策，不是團隊公告）。

這是**線上不可逆變更**，所以每一步都能擋下來：

| 防線 | 擋什麼 |
|---|---|
| **開發源偵測** | 見下 —— 最重要的一道 |
| 授權 | 只有 `access.allowed_users` 能按，且**權限檢查排在執行之前**（有測試驗順序） |
| 版號比對 | 下載回來的 wheel 內 `__version__` 必須等於預期 —— **驗內容不驗檔名** |
| zip 完整性 | 截斷的下載、下載到 HTML 錯誤頁 |
| 安裝結果 | `pip` 回非 0 就**不重啟**，舊版還在跑 |

> 💡 **安裝與重啟拆成兩步是關鍵設計。** 合在一起的話，
> 一次失敗的安裝會連帶把服務重啟成裝不起來的狀態。

其他細節都是踩過的：用 `sys.executable -m pip`（PATH 上的 `pip` 可能指向
系統 Python，會裝到錯的地方**而且看起來成功**）、下載帶
`Accept: application/octet-stream`（少了它拿到 JSON metadata）、
**存檔用原始檔名**（`uv`／`pip` 依檔名解析版本）。

### 🔴 開發源偵測：兩個判準，第二個是實測補上的

從源碼樹跑的部署**絕不能裝 wheel** —— site-packages 的實體副本會贏過
`.pth` 加的路徑，於是 `src/` 的修改再也不生效，而**沒有任何錯誤訊息**：
服務照跑，只是跑的是被凍住的舊碼。

| 判準 | 抓什麼 |
|---|---|
| ① PEP 610 `direct_url.json` → `dir_info.editable` | 標準的 `pip install -e .` |
| ② **實際載入位置不在 site-packages 內** | 純路徑 `.pth` 的開發源 |

**判準②是實測逼出來的**：本專案的 `ark_team_agent` 靠一個純路徑 `.pth`
把 `src/` 加進 `sys.path`，`direct_url.json` **沒有 editable 標記** ——
只有判準① 的話會回 `False`，而它確確實實是開發源。
**漏掉這條，一鍵更新會毀掉開發環境。**

> 💡 判準②直接看「實際載入的是哪個檔案」，比任何 metadata 可靠 ——
> metadata 描述的是安裝當下，而我們要問的是**現在跑的是誰**。

開發源上仍會收到通知（知道有新版有價值），但**不給按鈕**：
按了也會被擋，不如不給。

### 群組第一次說話會跳「想交給哪個 Agent？」選單

追下去是 `ConversationPlanner.plan()` 的第 7 條（前六條都沒命中就 clarify）。
而**沒命中第 4 條才是根因**：`session.instance` 來自 `_resolve_instance(topic_id)`，
查不到 topic 就 fallback 到 `_general_instance` —— **沒有任何 instance 帶
`general_topic: true` 時它是 `None`**，於是每一句話都掉到第 7 條。

四處一起修：

| | 修前 | 修後 |
|---|---|---|
| 按鈕 emoji | 硬編 `'🏗️' if 'backend' in name else '🎨'` | 從 `description` 取 |
| 按鈕文字 | `description` **全文** | `display_name` |
| 群組行為 | 跳選單 | 回一句 `@代號` 提示 |
| 設定檢查 | 無訊號 | 啟動時 warning |

**emoji 那條的判準是「instance 名字裡有沒有 `backend`」** —— 與 `team.yaml`
完全無關。實測本機七個部署**沒有任何 agent 叫 `backend-*`**，所以全部顯示 🎨。
（`check_hardcoded_names` 抓不到：它掃的是專有名詞，這是硬編的分類邏輯。）

按鈕文字原本是「🛠️ 維運管家 — 服務監控、套件維護、Bug 修復、部署」全文 ——
TG 會截斷，而那是給 TEAM.md 用的完整職責描述。修後是「🛠️ 維運管家」。

> 🔴 **群組不跳選單是行為變更，理由是那張選單本來就不適合多人環境**：
> 它**對所有人可見，而選擇只寫進按的那個人的 session**
> （`get_or_create(user.id, ...)`）。A 按了「已轉給 🧠 軍師」，
> B 看到會以為整個群都轉了。改成 `@代號` 提示 ——
> 那是明確、per-message、對誰都一致的指定方式。

另外選完會**寫回 `topic_map`**：原本只設 `session.instance`，而 session
**5 分鐘就過期**（`Session.timeout = 300`）—— 同一個 topic 五分鐘後會再問一次。

### 啟動檢查：General 沒有主人 / 有兩個主人

`validate_config` 新增兩條，**在此之前完全沒有訊號**（設定看起來很正常）：

- 有 `group_id` 但沒有任何 `general_topic: true` → 每句話都掉進 clarify
- **有兩個以上** → 後者覆蓋前者，而順序取決於 dict 迭代，**誰接 General 是不確定的**

實測抓到 `game-team-agent` 的第二種（`paddy-agent` + `ceo-agent` 都設了）。

### 守門

`tests/test_clarify_routing.py` 14 個：按鈕標籤三段 fallback、
emoji 不得回到硬編、群組回提示不回選單、`is_group` 有傳下去、
預設維持私聊行為、四種 General 設定組合（含**對本專案真實設定跑一次**）、
選完有綁定 topic。

`tests/test_version_check_and_update.py` 22 個：版本數字比較
（**`1.5.9` → `1.5.10` 這種只在 patch 跨過 9 時才出現的錯**）、
查不到不算過期、關閉時零網路請求、快取阻止重複呼叫、
開發源被拒（**直接在本專案上驗證**）、wheel 四種驗證、
安裝失敗不重啟、用 `sys.executable`、以及接線
（啟動有呼叫且包了 try、callback 有註冊、權限檢查在前、更新走 `to_thread`）。

## [1.5.6] — 2026-08-24 · ✅ Release

修一行**方向講反的註解** —— 它讓兩個不同部署各產出一份「故障回報」，
而那個故障不存在。

### 🔴 兩份獨立回報，同一個誤判

2026-08-24 收到兩份來自不同部署的回報（game-team-agent 的 `ceo-agent`、
另一部署的 `conductor-agent`），現象一致：

> agent 呼叫 `reply()` 後進 `AWAITING_REPLY`，超過 30 分鐘未降級、
> **無法接收新訊息**，手動 `send_to_instance()` 可觸發脫離。

兩份都判成「agent 永久卡住」，並要求加「5 分鐘自動降級 RUNNING」。
**兩個判斷都不成立**：

| 回報主張 | 實況 |
|---|---|
| 「有人在等 agent 回話，它卻沒動」 | 方向相反 —— 是 **agent 回覆完了在等使用者接話**（`api.py` 的 `/api/reply` 是進入這個狀態的唯一路徑） |
| 「30 分鐘未降級」 | 停在這裡很久**是正常的**。使用者一整天不說話，它就停一整天 |
| 「無法接收新訊息」 | 完全能接收（`daemon.py` 的可投遞狀態集合含它，收到就轉 `RUNNING`）—— **回報自己的 workaround「發任意訊息可脫離」就是反證** |

而誤判的來源是這一行：

```
舊：AWAITING_REPLY 有人在等我回話     ← 方向相反
新：AWAITING_REPLY 我回覆了，等你接話
```

照舊註解推論很自然：「有人在等 → 該做事卻沒做 → 卡住 → 要有超時降級」。

> 🔴 **這是「宣告了但沒接上」的近親：註解與實作語意相反。**
> 掃 import、跑測試、讀設定都看不到它 —— 它只在**有人照著推論**時才顯現，
> 而那時已經產出一份要求「復活刻意廢除的行為」的修復建議了
> （5 分鐘降級是 1.2.16 引入三態時刻意廢除的：它會讓正常閒置的 agent
> 在 60 分鐘的 hang 門檻內被反覆誤降級）。

已改寫 enum 的方向說明（含完整狀態轉移圖）與 `_enforce_session_limit`
的同源敘述。

### 安全網的時間基準沒跟上 1.4.7

`_awaiting_safety_net_due` 用 `state.last_activity`，而 hang 偵測
（`_hang_ref`）1.4.7 起已改用 `last_output or last_activity`。

`last_activity` **同時被 inbound 與 output 刷新** —— 那正是 1.4.7 要修的坑
（實測 aiops 的 ic-agent 每小時收巡檢訊息，`last_activity` 每次被推遲到
<60m，9 小時無輸出而帳面全綠）。當時只改了 hang 偵測，**安全網被漏掉**。

已對齊。**兩處用不同基準本身就是隱患** —— 同一個「多久沒動靜」的問題
有兩個答案，而兩邊都自認正確。

⚠️ 但要說清楚**這不是那兩份回報的成因**：安全網門檻是 `2× escalation`
（預設 3 小時），而回報觀察 30 分鐘就下了結論；且持續收到訊息的 agent
每次 inbound 都會轉回 `RUNNING`，根本進不了「安全網該救」的情境。
修它的理由是一致性，不是那個故障。

### 給消費端的守門範本：`examples/tests/test_docs_baseline_drift.py`

這次回報的**真正起點**不在套件，也不在提詞 —— 是**消費端專案自己的
`docs/` 與實際套件版本脫節**。回報者的規格文件寫於套件 1.1.x 時期，
而該部署跑 1.5.2；`1.2.16` 刻意廢除的行為，文件裡還記載著。

> 🔴 **agent 沒有做錯任何事。** 讀規格 → 比對實作 → 回報落差，
> 正是我們要它做的。它只是無法分辨「這份規格還有效嗎」——
> 文件不會標示自己過期，而升級套件不會動到消費端的 `docs/`。

新增可複製的範本：消費端在描述套件行為的文件加 `package_baseline`
frontmatter，守門比對它與實際安裝版本的 `major.minor`。
（patch 是 bug 修復不會讓設計描述失效；會讓文件過期的是 minor。
而 pip 的 pin 是**下限**不是基準 —— `>=1.1.13` 對 1.5.2 也成立，抓不到漂移。）

順帶收一條開發範本時**真的踩到**的：`test_no_duplicate_installation`
—— 同套件裝了兩份（`.venv` editable 1.5.6 + `~/.local` 的 1.4.11）時，
**哪一份被 import 取決於 cwd 與執行方式**，兩者都不報錯。
判準是「dist metadata 版本」對「實際 import 到的 `__version__`」。

### 守門

`tests/test_awaiting_reply_semantics.py` 14 個 —— 把**方向語意**釘成
可執行的斷言：進入路徑是 reply 不是 receive、inbound 會離開該狀態、
該狀態不擋投遞、註解必須講方向且不得回到反向敘述、
安全網基準與 hang 對齊、門檻不得縮短、**不得出現 5 分鐘降級的符號**
（`AWAITING_REPLY_TIMEOUT` / `status_changed_at`）。

核心那條做過反證 —— 改回 `last_activity` 確實變紅。

> 💡 寫測試時自己踩了一次同型的坑：新註解裡**原文引用**了舊敘述當說明，
> 於是守門測試被自己的說明誤報。這是本專案記載過的假警報型態
> （`check_hardcoded_names` 那次），改成描述而不引用。

## [1.5.5] — 2026-08-24 · ✅ Release

### TG 送檔白名單擋掉了 agent 的圖表與 PDF

`reply_file` 傳 `.png` 會被拒（白名單只有 `.html` / `.md`），而 agent
只看得到「檔案類型不允許傳送」—— **看不出是白名單的問題，也不知道該找誰改**。

改用明確的判準重排白名單：**交付物可以，程式碼與設定／機密不行。**

| | 放行 |
|---|---|
| 報告 | `.html` `.md` `.txt` `.pdf` |
| 資料 | `.csv` `.xlsx` |
| 圖片 | `.png` `.jpg` `.jpeg`（自動走 `send_photo`） |
| 文件 | `.docx` `.pptx` |

| | 仍然擋著 | 為什麼 |
|---|---|---|
| 程式碼 | `.py` `.sh` `.js` … | 送出去等於外流實作 |
| 設定 | `.json` `.yaml` `.env` `.ini` | 見下 |
| 憑證 | `.pem` `.key` `.crt` | — |
| 封裝 | `.zip` `.tar` `.gz` | 內容不可檢查，放行等於白名單失效 |

> 🔴 **`.json` 刻意不放行。** 它不是「程式碼」，但**機密最常長這樣**
> （service account key、token、credentials）。判準是「會不會外洩祕密」，
> 不是「是不是程式碼」—— 這兩個判準在 `.json` 上會得出相反的答案。
> 要傳結構化資料請用 `.csv`。單獨一條測試釘住它，因為它最容易被順手加上。

### `reply_file` 依副檔名自動分流（不必教 agent 選對工具）

圖片走 `send_photo`（TG 顯示預覽），其餘走 `send_document`。

🔴 **為什麼是工具自己分流，而不是在 description 教 agent 選。**
agent 只有 `reply_file` 一個送檔工具 —— `reply_task_image` 是**任務板專用**
（參數是 `status` / `title`，吃不了 `file_path`），所以「產出一張圖表然後
送出去」本來就沒有第二個選項。就算有，教 agent 二選一也不可靠：
本套件反覆記載的病就是 **agent 照最終 prompt 走，而 prompt 會漂移、會被忽略**。
**讓一個工具做對的事，比讓 agent 記得選對工具可靠。**

差別實際存在：`send_document` 傳圖片，使用者在手機上要點下載才看得到。

順帶修 `reply_file` 的 description —— 原文寫「支援任何檔案類型：
`.html/.pdf/.docx/.png/.csv` 等」，而白名單當時只有 `.html`/`.md`，
**列出的五種有三種會被拒**。agent 照 description 呼叫，拿到
「檔案類型不允許傳送」，看不出是自己被騙了。與 `log_to_leader` 寫
「發給綱手」同型：**description 是 agent 唯一讀得到的規格。**

⚠️ **圖片有兩條路，只有一條受白名單管**：

| 路徑 | 走的 API | 白名單 |
|---|---|---|
| `reply_task_image` | `send_photo` | **不檢查** —— 一直是通的 |
| `reply_file` | `send_document` | 檢查 → 1.5.4 以前 `.png` 被拒 |

所以「圖片傳不出去」只發生在 `reply_file`。放行是為了「就是要當附件傳」
的情境；**要預覽仍該用 `reply_task_image`**（`send_document` 傳圖在手機上
要下載才看得到）。

擴充**不等於放棄** —— `.exe` / `.sh` / `.py` / `.zip` / `.env` 仍然擋著。
白名單的目的就是防止 agent 把不該外流的東西送進 TG。

### 系統通報一直在對話群裡跳

啟動橫幅（`✅ AI Team 就緒 — 7/7 全員到位`）與重啟通知是**服務自己發的**
（不是 agent 的回覆），原本**無條件**送到 General topic 與每個有
`private_chat` 的人。

在只有 agent 的工作群那正是你想看的東西。但在**有人在聊天的對話群**，
每次重啟都跳一次，而它對群裡的人沒有任何意義。

新增 `notifications.startup`：

| 值 | 群組 General | 私訊 |
|---|:---:|:---:|
| `both`（預設） | ✅ | ✅ |
| `private` | ✖ | ✅ |
| `group` | ✅ | ✖ |
| `off` | ✖ | ✖ |

**重啟通知共用同一個開關** —— 它是「取代啟動通知」的同一類推播，
分開控制會出現「設了 `private` 但重啟時還是在群裡跳」，而那正是使用者
最常看到的那一則（服務會重啟，但不常首次啟動）。

`off` 只是不主動推播 —— **健康檢查與 log 不受影響**，服務起沒起來
仍可用 `/api/health` 或 `/status` 查。守門測試固定這條界線
（開關不得滲進 API 層）。

預設 `both` 維持現行行為，理由同 `access.group_policy`：
改預設會動到四個既有部署，而它們的工作群本來就該看到啟動狀態。

### 🔴 這份清單有孿生的一份，而且沒有任何測試在管

`ark_team_agent/telegram.py` 與 `ark_bot_agent/report/sender.py` 各一份。
兩邊內容碰巧相同，是因為**沒人動過** —— 不是因為有機制保證。
只改一邊，症狀是「同一種檔案在 A bot 傳得出去、在 B bot 傳不出去」，
而兩邊的 log 都只說「不在白名單」。

> 💡 **碰巧一致不算一致。** 與本專案修過的 TEAM.md 工具表、代號表、
> BM25 路徑 ×4 同型。

守門 `tests/test_file_whitelist.py` **43 個**：兩份逐項比對、報告格式不可移除、
新增格式在位、危險副檔名仍擋、格式正規化（少點或大寫會永遠比不中且不報錯）、
**接線**（兩個 send_document 都檢查、兩個 send_photo 刻意都不檢查）。
比對那條做過反證 —— 只改一邊確實變紅。

另加 `tests/test_startup_notification.py` 10 個：四種值的行為、未知值
warning + fallback、**三個發送點都受控**（漏一個就會「設了 private 還是
在群裡跳」）、重啟通知共用開關、開關不得影響健康檢查。

## [1.5.4] — 2026-08-24 · ✅ Release

新增 `access.group_policy` —— **在對話群裡，bot 代表的是一個人。**

### 群組路徑從來沒有「要不要回應」的判斷

`_on_message()` 的群組分支解析 `@mention` → 有就直送、沒有就交 planner
派給入口 agent。**沒有任何一步問過「這則訊息該不該回」**
（`grep group_policy` 在 1.5.3 是零命中）。

在 forum 架構（每個 agent 一個 topic）那是對的 —— 進到 `@qa` 的 topic
打字本來就是在跟 qa 說話。但把 bot 拉進**有人在聊天的群**，它會對每一則
訊息回應，群組立刻不能用。`ark_bot_agent` 0.7.0 踩過同一個坑。

| 值 | 行為 |
|---|---|
| `all`（預設） | 每則群組訊息都處理 —— 1.5.3 以前的唯一行為 |
| `mention_only` | 只回被 `@` 點名的 |
| `off` | 群組完全不回 |

未回應的訊息**仍寫進 session** —— 群組是資訊來源，不回話是禮貌，
不聽是浪費：agent 之後被 `@` 到時才知道前面大家在聊什麼。
寫入失敗只 log，不影響「不回應」這個主行為（略過是准入決策，
記憶是附加價值，後者不該拖累前者）。

### 🔴 預設是 `all`，與 `ark_bot_agent` 相反 —— 那是刻意的

| 套件 | 預設 | 為什麼 |
|---|---|---|
| `ark_bot_agent` | `mention_only` | 「群組每則都回」是**明確的缺陷** —— 沒有人會刻意想要一個每句都插嘴的助理 |
| `ark_team_agent` | **`all`** | 群組是 forum + topic 分流，**在 agent 自己的 topic 回應每則訊息就是設計本意**；改預設會直接改掉 nana / aiops / director / slot 四個正在用群組的部署 |

> 💡 MEMORY 記過這條（1.4.0）：**相容性保護的是「有人刻意依賴的行為」，
> 不是「碰巧存在的缺陷」** —— 同一條原則在兩個套件得出相反的預設值，
> 因為兩邊的現行行為分屬不同類。**套公式會做錯。**

### 判斷點的位置有三個理由

排在 **access 之後**（能不能用是准入問題，要不要回是禮貌問題，兩者不同層）、
**指令之後**（打 `/status` 本身就是明確的互動意圖）、
**planner 之前**（略過就不該喚醒任何 agent —— lazy spawn 有成本）。
守門測試直接驗這三個相對位置。

### 守門

`tests/test_group_policy_team.py` 16 個：三個策略值的行為、未知值 warning +
fallback、舊 `team.yaml` 缺欄位不炸、**接線**（呼叫點存在 + 三個順序 +
私訊不受影響）。接線那兩條**做過反證** —— 把接線整段移除後確實變紅。

## [1.5.3] — 2026-08-24 · ✅ Release

一族「硬編別的部署人名」的缺陷，以及**為什麼守門三次都沒抓到**。

> ⚠️ 本段原本誤寫進 1.5.2 —— 那一版 **2026-08-22 就已發布**（asset 在
> `igs-paddyyang-tw/ark_team_agent` 的 `v1.5.2`）。開發源 repo 的 tag 是
> `team-v1.5.0`，發版 repo 的是 `v1.5.2`，**兩套 tag 系統**，只看本機
> `git tag` 會得出「1.5.2 未發布」的錯誤結論。已發布的版本不能改寫內容
> —— 實測兩顆 wheel 大小就不同（596455 vs 598097）。

### 工具說明裡的幽靈收件人：`pm-agent` / 綱手

上一節把工具**表**合一了，漂移卻轉移到沒人當成規格的地方 —— **說明文字**。
`log_to_leader` 的 description 寫著「私下發給綱手（pm-agent）」，而：

| 寫的 | 實際 |
|---|---|
| 收件人是 `pm-agent` | 多數團隊沒有這個 instance（本團隊是 paddy / admin / leader / …） |
| 固定某個人 | `POST /api/log` → `_leader_for(source)`，依 group **動態解析** |

它是 ninja fork 遺產。而 1.5.2 起 TEAM.md 的工具表說明**直接取自這些
description** → 每個 agent 的 always-on steering 天天寫著一個不存在的收件人。

同源的還有 TG「執行計劃」按鈕的提詞第 3 條「無任務 → 向 pm-agent 請求派工」——
agent 照做就是 `send_to_instance` 給不存在的對象。兩處都改成描述角色
（「你的 leader，由系統依團隊編制解析」），不寫死人名；另修兩處已與實作不符的
過時註解（該處程式碼本來就是動態找 `role == "leader"`）。

> 🔴 **與 TEAM.md 幽靈成員 `cto-agent` 同型**：成員表已經動態化了，
> 漂移轉移到「說明文字」。**最容易漂移的恰好是最像文件的那一塊。**

### 代號有兩份實作，其中一份是手寫的火影名單

`TelegramAdapter._build_codenames` 從 `description` 動態解析代號（正確），
而 `scripts/task_screenshot.py` 另有**一份手寫的 13 人火影代號表**
（`pm-agent: 🔱綱手` …）。對任何非 ninja 團隊，那張表每一列都不會命中
→ 永遠 fallback 到 raw instance 名。**不會壞、不報錯，只是那個美化功能
等於不存在**，而看起來像有在運作。

實作抽成 `config.build_codenames()`（單一權威來源），兩邊都走它。
放 `config.py` 而不是 `telegram.py`：截圖腳本不該為了取代號去 import
整個 telegram 適配器（連帶拉 python-telegram-bot 依賴）。

### 兩個「範本產出即壞掉」

- **`init` 產出的 `scheduler.yaml` 兩個 job 都 `target: pm-agent`** ——
  任何新專案都沒有那個 instance，兩個排程**從第一天起就對不存在的對象派工**，
  而 `init` 回報成功。改用 `--entry-name` 的 `entry` 變數。
- **`chat_api.ChatSendRequest.instance` 預設 `pm-agent`** —— 呼叫端不給就
  靜默送進虛空（`ok: False`，看不出原因）。改為**必填**。
  原本有個測試 `test_chat_send_default_instance` 正在**固定這個缺陷**，
  讓它看起來是刻意行為；已改為驗 422。

> 💡 **填錯的預設比沒有預設更糟** —— 422 至少會當場說「你少給了 instance」。

### 守門：掃描範圍與名單都補了（這批漏網的原因）

`scripts/check_hardcoded_names.py` 三個缺口：

| 缺口 | 後果 |
|---|---|
| **只掃 `src/ark_team_agent`** | `src/` 底下有**兩個**套件，`ark_bot_agent` 從沒被掃過 |
| BANNED 只有「火影」一個詞 | `pm-agent` 與整組角色名（綱手/鳴人/佐助…）一路倖存 |
| docstring 判定用「數三引號」 | 單行 docstring（引號數偶數）判不出 → 實測 3 處假警報 |

補掃第二個套件、補齊 ninja 名單與各部署入口代號、docstring 改用 `ast`
判定並額外剝除行尾註解。補上後一次噴出 **13 處**（3 假 10 真），
全部處理完歸零。

> 🔴 **假警報要一起修** —— 常駐假警報的代價不是雜訊，是維運開始
> 習慣性忽略該檢查，等於把它廢掉。

### 範例與文件

- **`examples/team.yaml`**：instances 改中性名（原本用 `pm-agent` +
  寫死 `~/game-team-agent/...` 絕對路徑）、`team_doc` 從「整段註解掉」
  改為**預設啟用**（不設正是實測會出事的狀態，範例把它當起點等於教人踩坑）、
  補 `display_name` 與 `kiro_files.steering.team_md: always`、
  修正 `description` 格式示範（代號靠全形 `—` 分隔）。
  另訂正標題 —— 原文說「manager / leader / worker 三種 role」而
  instances 裡**沒有任何 manager**，照標題寫指揮鏈會觸發幽靈角色 warning。
- **`team.yaml` 的 `team_doc.command_chain`**：入口寫成 admin（實際是
  manager）、把 admin 叫「管家」（它是「維運管家」）。那行會寫進
  **全部 6 份 always-on TEAM.md**。順帶：一致性檢查是純子字串比對 role 名，
  四個 role 都要出現，否則每個 agent 各噴一次「隱形」warning（修前正在噴）。

### 守門


新增「工具 description 不得寫死非本團隊 instance 名」四角色 × 兩層
（description 本身 + 產出的 TEAM.md 工具表）。**只掃 description 值不掃原始碼**
—— 說明性註解要能講這段歷史，掃字面值會被自己的註解誤報（本檔記載過的假警報型態）。

本版新增守門共 **14 個**（另有 7 個在 `ark_bot_agent` 側）：
掃描器涵蓋兩套件且不誤報單行 docstring／行尾註解、兩份設定的 role 與代號一致、
`command_chain` 涵蓋每個實際存在的 role。**每條都做過反證** ——
把設定改回修前的值確認會紅，不是空跑。全量 **1892 passed**。

## [1.5.2] — 2026-08-22

團隊認知動態化收尾（工具表合一 + 執行期補欄位）+ 兩個修正。
**依版號規則歸 Patch**：bug 修復 + 既有 API 補欄位（那些欄位別的端點早有）+
行為保持的 refactor，無新公開 API、無 breaking。

> ⚠️ 本段合併自先前工作區疊的 1.6.0 + 1.7.0 兩個未發布 minor —— 那兩級跳號
> 與內容不符（1.5.0 是最後發布版，兩者都沒發過）。依「版號要對應真實事實變化、
> 實務走 Patch」收斂為單一 1.5.2。

### TEAM.md 工具表：兩套來源合一（原 1.6.0）

`backend.write_team_context` 的「MCP 工具」表原本是**手寫 markdown**，
與 `team_mcp._get_tools()` 的實際過濾邏輯是**兩套獨立來源** —— 這正是本套件
反覆記載的「宣告了但沒接上」病根形狀。實測它們已經漂移：

| 手寫表寫的 | 實際 `_get_tools()` |
|---|---|
| 漏了 `smart_delegate` / `reply_task_image` / `reply_file` / `decision_*` | 有 |
| `wiki_ingest` 標成「只有 leader/admin」 | worker 也拿得到 |

改法：抽出純函式 `tools_for_role(role)` 當**單一權威來源**，MCP server 的
`tools/list` 與 TEAM.md 工具表都走它；表格的說明文字直接取自工具 schema 的
description（不再手寫）。worker 白名單抽成 `WORKER_TOOL_NAMES` 常數。

> 💡 成員/指揮鏈/角色分佈早已動態化，唯獨工具表是手寫的 —— **最容易漂移的
> 恰好是最像「文件」的那一塊**，因為它看起來只是說明，沒人想到它其實是規格。

### 新增 `get_task`：補上多輪協作的斷點（原 1.6.0）

`list_tasks` 只給標題，接手任務或驗收的 agent 看不到 `summary` /
`changed_files` / `verification` / `residual_risk` —— 交接細節就在 `TaskBoard`
裡（`get_task` 方法早就有），只是從沒暴露成 MCP 工具。多輪協作在這裡斷掉。
新增 `get_task(task_id)` 工具，吐出完整 handoff（進度紀錄取最近 5 筆）。worker 也拿得到。

### `/api/status` 補上 role / description / display_name（原 1.7.0）

啟動時寫進 `TEAM.md` 的成員表是靜態快照。執行期 agent 走
`query_team_status` → `GET /api/status`，而 `daemon.get_status()` 每列
**只有 runtime 狀態**（status / idle / last_output），沒有 role/description ——
那些欄位只存在於另一條路徑 `get_instances_detail()`（`/api/instances`）。
結果 agent 查得到「誰在忙」，**查不到「誰負責什麼」**。
改法：把三個欄位放進**唯一狀態出口** `get_status()`，不另組平行來源。
`query_team_status` 輸出同步加「成員與職責」段（舊 daemon 帶不出時自動省略）。

### 修掉 legacy-question-convert 誤觸已回覆的 agent（原 1.7.0）

`daemon` Rule 1（把舊習慣 agent 掉在 Topic 的問句轉成 decision-request）
會對**已經成功回覆**的 agent 誤觸 —— reply() 後 `AWAITING_REPLY` 是正常態，
但只要內部推理輸出有個 `?` 就觸發一次假轉換，且失敗後不解鎖（agent 卡死）。
改法：用 `state.last_output`（只由真的產出更新）判斷；輸出比 `remind_secs * 1.5`
更新代表剛回覆過，那個 `?` 是內部獨白，跳過。同時修正一處標成 `[1.5.2]`
（本版之前不存在該號）的漂移註解。

### 守門

`tests/test_dynamic_team_tools.py`：固定「TEAM.md 工具表 = `tools_for_role`」
不變量、`get_task` 存在且 worker 可用且吐得出 handoff。全量 1859 passed。

## [1.5.1] — 2026-08-21

### 🔴 建構子字串預設值也依賴 cwd —— 掃描器抓不到的第二種形狀

```python
DB_PATH = Path("state/decisions.db")        # ← 1.4.9 抓到並修掉
db_path: str = "state/message_queue.db"     # ← 抓不到
```

守門腳本 `check_package_paths` 的 `_RE_REL_PATH` 只匹配 `Path("...")`。
而字串預設值一樣是 cwd 相對、一樣不拋例外、一樣在別的目錄啟動時寫到別處。

**同型錯誤已出現三次**（`decision_manager.DB_PATH` ✅ 抓到、
`MemorySearch("data/sessions.db")` ❌ 溜過去、本次兩處 ❌ 溜過去）
→ 補掃描規則，不再人工盤點。

### 修正

| 位置 | 改法 |
|---|---|
| `conversation_log.py` | `db_path` 預設 `"state/conversations.db"` → `None`，走 `get_state_dir()` |
| `message_store.py` | `db_path` 預設 `"state/message_queue.db"` → 同上 |

🔴 **`MessageStore` 特別嚴重** —— 訊息佇列是**寫入目標**。
寫到別的目錄不是「找不到資料」，是**訊息消失**，
而 daemon 把訊息丟進佇列後就回傳成功了。

明確傳入的路徑仍優先（測試與消費端的逃生門）。

### 相容性

**純內部路徑修正，無行為改變。** 從專案根啟動的部署（全部六個）
解析結果與 1.5.0 完全相同。

## [1.5.0] — 2026-08-21

### Added
- **hang_detector v2** — 喚醒失敗 15min 即重啟（wake_retry_minutes）、啟動零 output 10min 偵測（startup_timeout）、context 累積 500K 字元自動 reset
- **維運日報三軌** — scripts/daily_report.py（MD→HTML→TG）+ 套件範本 templates/reports/
- **啟動顯示版號** — log + TG 啟動訊息含「📦 ark-team-agent v{version}」
- **M4 TG menu 整合** — /agents 面板 team 模式顯示 team agents 按鈕 + @mention 走 dispatch
- **crash 診斷改進** — event log 同時記錄 output head + tail（rc=1 的錯誤訊息在開頭）
- **InstanceState.cumulative_output_chars** — 追蹤 context 累積量

### Changed
- hang_detector: wake_retry_minutes=15、startup_timeout_minutes=10、context_limit_chars=500000

## [1.4.11] — 2026-08-20

### Fixed
- **reply 失敗回退狀態** — no output channel / 404 時自動 AWAITING_REPLY → RUNNING（防永久卡住）
- **private_chat channel 永久綁定** — 從 config 初始化，context compact 後不遺失 reply channel

## [1.4.10] — 2026-08-20

### Added
- **訊息 TTL 自動過期** — send_message 支援 ttl_minutes（預設 15 分鐘），過期自動丟棄
- **Cancel 主動取消** — POST /api/cancel 可取消待投遞訊息
- **auto_restart** — hang_detector escalation 後自動重啟卡住的 agent
- **headless 模式** — team.yaml `mode: headless` 或 --headless flag（不啟動 TG）
- **/api/v1/dispatch** — 外部整合用派工端點（bot+team 整合）
- **/api/v1/agents** — 列出可用 agents

### Changed
- hang_detector 門檻收緊：timeout 60→30 分鐘、escalation 180→90 分鐘

## [1.4.7] — 2026-08-19

三個缺陷，**共同點是「每個指標都說沒問題，而它是壞的」**。全部從實機日誌查出來。

### 🔴 ① hang 偵測看錯欄位 —— agent 9 小時零輸出而帳面全綠

實測（aiops）：obs-agent 每小時發巡檢給 ic-agent，ic-agent 從 22:30 起
**再也沒有任何輸出**。而同一時間：

```
status=running · crash_count=0 · halted=False · 進程活 15 小時
last_activity=08:30（每小時被 obs 的訊息刷新）
```

**每一個指標都說它很好，hang 偵測（門檻 60 分）從未觸發。**

**真因**：`last_activity` 同時被 **inbound（訊息寫進 stdin）** 與
**output（產出增加）** 更新 —— 它混了「我收到東西」與「我做了事」兩種語意。
而 hang 要問的是後者。每小時一次 inbound 就把它刷新到 60 分鐘內。

> 🔴 這是 **1.2.16（`IDLE` vs `AWAITING_REPLY`）同一個形狀**：
> 一個欄位承載兩種語意，於是需要區分它們的那個判斷必然失效。

**修法**：分出 `last_output`（**只**由「產出增加」更新），hang 偵測改看它；
沒有任何產出時（剛啟動）退回 `last_activity`，避免剛起來就被誤判。
`last_activity` 語意不變 —— idle eviction 要它（剛收到工作的 agent 不該被回收）。
`/api/status` 一併暴露 `last_output`，否則「9 小時沒輸出」還是查不出來。

守門用 `ast` 檢查**每個** `last_output` 賦值都位於「output_count 增加」分支內
—— 否則就悄悄退回混用語意。

### ② TG adapter 晚 10 秒 link —— 那個窗口內的回覆被丟掉

```
01:25:02  Telegram bot connected
01:25:03  WARNING 💬 REPLY leader-agent: no telegram adapter and no web session  ← 掉了
01:25:12  Daemon API: Telegram adapter linked
```

`daemon_api.tg = tg_adapter` 排在整段啟動流程**最後**，而中間有一個**刻意的**
`await asyncio.sleep(10)`（等重啟中的 agent 安定才發啟動訊息）。
那 10 秒內 agent 已經在跑、可以回覆，但 `DaemonAPI.tg` 還是 `None`。
而 `/api/reply` 的重試**只等 3 秒** → **必然失敗**。

**修法**：TG 一連上就立刻 link。那個 sleep 是為了「發啟動訊息的時機」，
與「adapter 可不可用」無關 —— 沒有理由在這段時間扣住這個參照。
尾端保留冪等指派（涵蓋「TG 啟動拋例外」的分支），兩處 log 用不同字樣以便分辨。

### 🔴 ③ `POST /api/v1/dispatch` 從加進來就沒有一次成功過

查上面兩項時全量測試有紅燈，先確認**不是自己造成的**
（`git archive HEAD` 匯出到暫存目錄隔離跑，**不動工作樹**），
確認在 HEAD 就是紅的，追到引入它的 commit。

```python
async def dispatch_message(request: Request)   # ← Request 沒有 import
```

本檔有 `from __future__ import annotations`，所以註解是**字串**、不會在定義時求值：

- **沒有 NameError、沒有 ImportError**
- FastAPI 解析不出型別 → 把它當成**名為 `request` 的 query 參數**
- 實測：`POST /api/v1/dispatch` → **HTTP 422**
  `{"loc":["query","request"],"msg":"Field required"}`
- `app.openapi()` 也炸（`ForwardRef('Request')` 無法建 TypeAdapter）

**所以不只 openapi 測試紅 —— 整條 dispatch API 每一次呼叫都是 422。**
補上 import 後實測：openapi ✅、路由回 200。

> 💡 **`from __future__ import annotations` 會讓「漏 import」變成靜默失效** ——
> 這個套件反覆修過的家族又一個新形狀。
> 型別註解不再於定義時求值，所以**漏 import 的代價從「立刻炸」變成「行為安靜地錯」**。

### 測試

新增 `tests/test_hang_and_adapter_link.py` **9 個**。全量 **1652 passed**。

## [1.4.6] — 2026-08-18

**起點是一句「行為正確」的回報。** slot 的 data-engineer 說：
「無法 `send_to_instance` 給 admin-agent（worker 不能直接發給 admin）。
已改發給 analyst-agent（我的 leader）請求協調。」

**它做對了** —— 那是設計上的指揮鏈限制。但這句話讓我去查
「worker 到底有沒有辦法讓私訊的那個人知道」，於是找到三個缺陷。

### 🔴 1.2.26 修的那個缺陷家族，有一個兄弟倖存了

`log_to_leader` 的 private fallback：

```python
_fallback_cid = (
    self._private_chat_map.get(leader_name)
    or self._private_chat_map.get("admin-agent")     # ← 寫死
    or next(iter(self._private_chat_map.values()), None)
)
```

實測各部署的私訊入口：

| 部署 | 入口 instance | 寫死的 `admin-agent` 查得到？ |
|------|--------------|---------------------------|
| aiops / paddy / slot / fish | `admin-agent` | ✅ 碰巧命中 |
| **nana** | `ark-agent` | ❌ **落空** |
| **director** | `tech-agent` | ❌ **落空** |

**它沒有炸掉，只是因為第三層「任一 `private_chat`」在單人部署下剛好是同一個人**
—— 多個 `private_chat` 時就會挑到錯的人。

1.2.26 就是為這件事加了 `owner_chat_id()`（依 role 推導），**只是這處沒改到**。

**而 CI 沒抓到，是因為 `BANNED` 只加了 `ark-agent`、沒加 `admin-agent`。**

> 🔴 **CI 名單的漏項不會表現為誤報，會表現為「那一類問題永遠不被發現」** ——
> 這是 1.2.26 寫下的教訓，這次由同一個家族的兄弟再次證明。
> 本版把 `admin-agent` 補進 `BANNED`（**加之前先量過噪音**：程式碼行只命中 2 處 ——
> 1 個真缺陷、1 個範本舉例字串）。

### worker → admin 的 403 現在會告訴 agent 該怎麼做

原本只回 `"{source} (worker) can only send to leader or worker"` —— 只說不准、不給路。
而 1.1.9 就定下過原則（P2P 停用時的 403 附替代路徑，**避免 agent 自行發明繞道**），
worker→admin 這條沒跟上。

現在：

```
{source}（worker）不能直接發給 admin —— 請改用 send_to_instance 發給
{實際的 leader 名}（你的 leader）由其轉呈，或用 log_to_leader 私下回報。
```

**帶的是那個 agent 實際的 leader 名**（`_leader_for(source)`），不是泛稱。

### 順手刪一段死碼

那行 403 `raise` **重複了兩次**，第二行永遠不會執行。無害，
但它是「這段被複製貼上過」的訊號 —— **而複製貼上正是兩份實作漂移的起點**。

### ℹ️ 設計本身不改：worker 仍然不能直接發給 admin

這是刻意的指揮鏈。worker 有三條路讓人知道：

| 管道 | 到哪 |
|------|------|
| `log_to_leader()` | leader 的 topic；topic 不可用時 fallback 到 **owner 的私訊**（本版修好的就是這條） |
| `send_to_instance(leader)` | 由 leader 轉呈 |
| `reply()` | 使用者發問處（`reply_to_origin` 開啟時）或自己的 topic |

**系統層的緊急通報不受此限** —— `notify_paddy()` 與 L3 拍板卡自 1.2.26 起
依 role 推導 owner，直接進私訊。

### 測試

新增 `tests/test_worker_escalation.py` **6 個**，含用 `ast` 掃「`raise` 之後的死碼」
與「api.py 程式碼行不得出現任何寫死的入口 agent 名」。全量 **1599 passed**。

## [1.4.5] — 2026-08-18

### 新增 qa 專屬 SOUL/BRAIN 範本

實機 log 顯示 `Using SOUL.md template for qa-agent: .../templates/agents/coder/SOUL.md`
—— **測試工程師拿到 coder 的人格**。既有部署無害（`soul_md` policy 是 `once`，
不覆寫既有檔案），但**新部署的 qa agent 會是個 coder**。

範本挑選邏輯本來就會**先試 instance 名**（`qa-agent` → `qa/`），
所以只要補上 `templates/agents/qa/` 就生效，不需要改程式。

內容不是複製 coder 改標題 —— 寫的是 QA 真正該有的判準，
其中「假綠燈的常見形狀」直接來自本套件修過的缺陷：

- 掃描目標路徑不存在 → 掃 0 個檔卻回報 ✅
- 只驗寫法沒驗實際值 → 擋得住舊寫法，擋不住「出口組錯」
- 提詞裡寫死的路徑與實際資料位置不同 → 永遠掃不到，但回報成功
- 測試被自己的註解命中（掃描式測試要排除註解與 docstring）
- 語法檢查通過但結構錯誤（插入位置不對造成死碼）

### 🔴 順手抓到的更嚴重問題：`leader` 映到不存在的範本

```python
role_map = {"admin": "admin", "leader": "tech-lead", ...}   # ← tech-lead 不存在
```

`templates/agents/tech-lead/` **從來沒有過**，而 `leader/` 就躺在旁邊沒人用。

**為什麼一直沒被發現**：挑選邏輯先試 instance 名，所以叫 `leader-agent` 的
碰巧命中 `leader/`，映射表根本沒被走到。但 leader 若叫別的名字
（`pm-agent`、`tech-agent`…）→ `pm/` 不存在 → 退回 `tech-lead/` 也不存在
→ **`soul_content` 是空的，SOUL.md 不會被寫出來**。沒有錯誤訊息，
只是那個 agent 沒有人格檔。

> 🔴 **而既有測試把這個缺陷寫成了規格**：
> `test_leader_defaults_tech_lead` 斷言 `== "tech-lead"`，綠燈好幾個月 ——
> 因為它驗的是「映射表的值」，**沒有驗那個值指向的範本是否存在**。
>
> 💡 **驗「設定值等於某字串」時，要順手驗「那個字串指到的東西真的在」** ——
> 否則測試會保護一個壞掉的映射。

### 測試

新增 `tests/test_agent_templates.py` **25 個**完整性守門：
每個 `role_map` 目標必須存在、每個範本都要有 SOUL+BRAIN、範本不可為空、
qa 範本必須真的在談測試（不是複製 coder）、
以及「instance 名的查找必須在 role fallback 之前」
（順序反了的話新增 `qa/` 也不會生效）。

全量 **1593 passed**。

### ℹ️ 已知範圍：其他常見 worker 仍借用 coder

`security` / `architect` / `design` / `researcher` / `tester` 都沒有專屬範本 →
借用 `coder`。**沒有擅自新增** —— 人格內容需要真的想清楚該有什麼判準，
複製貼上反而製造「看起來有、實際上沒對」的假象。
需要時用 `team.yaml` 的 `template` 明示，或另案補範本。

## [1.4.4] — 2026-08-18

**兩個 1.4.3 的漏，都是從實機 log 抓到的，不是靠讀 code 想出來的。**

### 🔴 群組路徑的 `@xxx` 沒被清掉

實機 log 的證據 —— 同一句話兩條路徑結果不同：

```
私訊：📨 私聊 @mention → qa-agent「你是誰」        ← 乾淨 ✅
群組：Sent to qa-agent: @qa 你是誰                 ← 沒清掉 ❌
```

1.4.3 只更新了局部變數 `text`，而群組分支實際送出的是 **`delivered_text`**
（`ctx_text = delivered_text`）。

> 💡 **同一件事兩條路徑結果不同，就是漏改的訊號。**
> 我在 1.4.3 的測試裡驗了「`@` 有被移除」，但驗的是 `ChatRouter` 的回傳值 ——
> **沒有驗「送出去的那個變數」**。掃字面值擋得住寫法錯誤，擋不住「改到錯的變數」。

後果：agent 收到帶 `@qa` 的字串，可能誤判成「這要轉給別人」。

### `reply_to_origin` 開啟時不再發「✅ 已轉給 XXX」

1.4.3 說「那句 ack 屬部署的提詞，套件不改」—— **講錯了一半**：
套件在私訊路徑**自己也有一份**（`telegram.py`）。實測使用者看到的是：

```
⏳ 👃 喬巴 目前離線，正在啟動中，請稍候...    ← 有資訊（解釋等待）
✅ 已轉給👃 喬巴                              ← 沒資訊（答案就要來了）
```

`reply_to_origin` 開啟時答案會回到**同一個視窗**，這句話沒有路可指了。
已閘住；**lazy spawn 的「正在啟動中」保留** —— 那句解釋了為什麼要等，兩者不可一起砍。

> ⚠️ 寫這個守門測試時踩到自己的坑：我的說明註解引用了那兩句訊息文字，
> 純字串比對被**自己的註解**命中而誤報。測試改成只掃程式碼行 ——
> 這正是本套件記載過的假警報型態（`ast`／排除註解），這次犯在測試上。

全量 **1546 passed**。

## [1.4.3] — 2026-08-18

### 群組也解析 `@mention` —— 與私訊路徑一致

**回報**：`@qa 你是誰` 之後，喬巴的答案**先**到、入口 agent 的
「✅ 已轉給喬巴（QA）」**後**到，順序倒置。

**真因不是順序，是群組路徑根本不解析 `@mention`**：

| 路徑 | 誰決定目標 | `@qa` 的下場 |
|------|-----------|-------------|
| 私訊 | `ChatRouter.route()` | ✅ 直送 qa-agent |
| **群組** | `planner.plan()` | ❌ **不看 `@`**，一律給 topic 主人 |

所以在 General 打 `@qa`，訊息其實送給**入口 agent**，靠**它的 LLM 去讀懂那個 `@`**，
再 `send_to_instance` 派給 qa。而 agent 的 ack 產生於 `send_to_instance` **之後** ——
被派工者又可能秒答（「你是誰」不用做事）→ **答案先到、ack 後到**。

那條路徑有四個代價：**順序倒置**、**多一次 LLM 呼叫**、
**可能判錯人**（而 `@qa` 本來是使用者已經做完的決定）、**多喚醒一個 agent**。

**修法**：群組分支在 `planner.plan()` 之前先過 `ChatRouter`，
解析出**單一明確目標**時直接接手，`@xxx` 從文字中移除（agent 收到乾淨需求）。

**刻意交回 planner 的三種情況**（不接手）：

| 情況 | 為什麼 |
|------|--------|
| `@all` 廣播 | 廣播語意由既有流程處理，不在這裡重新實作一份 |
| `@nobody`（解析不出目標） | 交回 planner，**不可靜默丟掉訊息** |
| `@qa @cto`（多目標） | 群組分支的 `Action` 是單一 instance，支援多目標要改成迴圈（範圍更大）。代價是這種情況仍有雙回覆 —— 但那本來就是 orchestrator 該做的事 |

### 順帶：「已轉給 XXX」這句 ack 該由誰發

套件在**私訊路徑**本來就有一個 ack（`telegram.py`，派工當下就發、時機正確、
會去掉職稱括號）。而入口 agent 的提詞另外還教它自己 reply 一次 ——
**兩個 ack，而 agent 那份必然遲到**。

更關鍵的是：**`reply_to_origin` 啟用後那句話已經沒有資訊價值** ——
它原本的用途是「去別的地方看答案」，而現在答案會回到同一個視窗。**指路的話沒有路可指了。**

> 💡 **回報「我做了什麼」沒有價值，回報「你接下來會遇到什麼」才有。**
> 該說話的唯一情況是「這件事會花很久，大概 N 分鐘」。

這部分屬**部署的提詞**（各團隊的 `intent-classify.md` / `SOUL.md`），套件不改 ——
但範本註解與本 CHANGELOG 把理由寫清楚，讓各部署自己決定要不要跟。

### 測試

`tests/test_group_mention.py` **10 個**（含三種刻意不接手的情況、
以及「接手條件必須同時檢查單一目標／非廣播／目標存在」的守門 ——
少任何一個都會讓訊息跑錯地方或消失）。全量 **1544 passed**。

## [1.4.2] — 2026-08-18

### 私訊問 → 回私訊（`reply_to_origin` 現在涵蓋兩種場景）

1.4.0/1.4.1 只處理了**群組 topic**。私訊那條路徑原本是：

```python
if has_mention and target != bound_instance:
    _reply_channel[send_target] = "topic"      # ← 你在私訊問，答案跑到群組 topic
```

**同一類體驗問題，只是場景不同**：回覆去「agent 的家」而不是「你問的地方」。
使用者在私訊 `@喬巴 …`，答案出現在群組的 Topic #7。

**修法**：`_origin_topic` 的值從 `int`（只有 topic id）改成
**`(kind, dest)`** —— `("topic", topic_id)` 或 `("private", chat_id)`。

> 💡 **為什麼不開第二個 dict 記私訊** —— 「回到你問的地方」是**一個**概念，
> 兩個平行的 dict 必然漂移（本套件修過太多次兩份實作不一致）。
> 值多一個 kind 欄位，兩處寫入、一處讀取，判斷點仍然只有一個。

⚠️ **私訊的目的地記的是 `msg.chat_id`（使用者那個視窗），不是 agent 自己設定的
`private_chat`** —— 被派工的 worker 通常**沒有**那個設定，靠它會查不到而退回 topic，
等於問題沒解。派工鏈的傳遞與群組情境完全相同。

未啟用 `reply_to_origin` 時，私訊的三分支判斷原封保留（`@mention` 別人 → topic）。

### 測試

`tests/test_origin_routing.py` 16 → **22 個**（私訊回原視窗、私訊派工鏈傳遞、
必須用記下的 chat_id 而非 agent 設定、未啟用時保留舊分支、值必須是 pair）。
全量 **1534 passed**。

## [1.4.1] — 2026-08-18

### 🔴 訂正 1.4.0：origin 回覆路由改為 opt-in（預設關）

**1.4.0 讓它無條件生效，那是錯的。** 兩個問題：

1. **沒有任何開關可以關掉。** 而它改變的是「訊息出現在哪裡」——
   那屬於各團隊的動線習慣，**不該由套件替它們決定**。
2. **實測受影響的不只回報者**：`group` 欄位的使用量 ——
   aiops **6** 處、director **4** 處、slot **8** 處、fish 也有。
   那是刻意的三層架構（worker 輸出收攏到組 leader 的 topic），
   而 1.4.0 在使用者互動情境下把它拿掉了。

> 🔴 **而我在 1.4.0 的 CHANGELOG 寫「歸類的價值不變」—— 那句話只對自主輸出成立。**
> 對 group 架構下的使用者互動，歸類確實消失了。**敘述比實際樂觀，這是本套件
> 一直在修的那類毛病（文件說的比實際好聽），這次犯在自己身上。**

**修法**：新增 `communication.reply_to_origin`，**預設 `false`**。

```yaml
communication:
  reply_to_origin: true      # 預設 false；扁平 topic / 純私訊架構開了沒有代價
```

| 架構 | 建議 | 理由 |
|------|------|------|
| 扁平 topic（每 agent 一個 topic）／純私訊 | **開** | 沒有 `group`，開了不犧牲任何歸類 |
| 三層 group（總機 → 組 leader → 職人） | **維持關** | 組內對話的連續性靠的就是共用 leader topic |

**單一判斷點**：`_origin_routing_on()` —— `/api/send` 的傳遞與 `/api/reply` 的覆蓋
都問它，並用掃描測試固定「只能有兩個呼叫點」。
兩處各自判斷必然漂移（1.3.3 的 `restart.flag` 就是兩份實作不一致的下場）。

**未啟用時完全不碰 origin 狀態** —— 不是「記了但不用」，而是連 `_origin_topic`
都不寫。理由是「記了但不用」會讓下一個讀 code 的人看到一堆沒有消費者的狀態，
那正是 1.3.3 那個誤導的來源。

### 這次的版號檢討（記著，避免再犯）

1.4.0 把兩件無關的事綁在一起：`_builtin:ingest-queue`（純新增、零風險，不設 job
就不會執行）與 origin 路由（改變六個部署的既有行為）。
綁在一起的後果是**想要其中一個的人被迫接受另一個**。應該分兩版發。

> 💡 **判準**：一個版本裡若同時有「純新增」與「改變既有行為」，就該拆。
> 版號能表達「有多大」，但表達不了「有幾件事」。

### 測試

`tests/test_origin_routing.py` 11 → **16 個**（新增預設關、group 歸類保留、
loader 讀得到開關、單一判斷點四項守門）。全量 **1529 passed**。

> ⚠️ **實作時我把 `_origin_routing_on()` 插進 `__init__` 中間，吞掉了它後面三行**
> （`self.app = self._build_app()` 等）—— 那三行變成 `return` 之後的死碼。
> `ast.parse` **通過**（語法完全合法），是全量測試噴出 35 failed + 9 errors 才抓到。
>
> 💡 **「語法檢查通過」不代表結構正確。** 在既有方法中間插入新方法時，
> 一定要確認插入點是不是那個方法的結尾 —— 這類錯誤只有跑測試才看得見。

## [1.4.0] — 2026-08-18

兩個功能，都來自 slot 的回報。**minor 跳號**是因為第一項改變了既有的回覆路由行為。

### 🎯 回覆回到「使用者發問的地方」，不再固定回 agent 的歸屬 topic

**回報**：「對話在管家指定 agent 後卻會回到 topic，對使用感受不好 —— 大家習慣在 General 對話。」

**根因**：回覆的目的地一直是 `_resolve_output_topic(instance)` ——
**agent 的歸屬**（自己的 topic，或 worker 所屬 leader 的 topic），
而不是**對話發生的地方**。inbound 的 `message_thread_id` 有被讀到
（`telegram.py`），但只用來判斷「誰該處理」，沒有留下來當回覆目的地。
`_reply_channel` 只有 `private` / `topic` 兩態 —— 套件裡**沒有「回原處」這個概念**。

這不是 bug，是取捨：原意是「訊息歸類」，同一 agent 的輸出集中在一個 topic 好找歷史。
**問題是使用者的心智模型不同** —— 使用者想的是「我在跟這個群講話」，
系統想的是「每個 agent 有自己的辦公室」。而 `general_topic: true` 的設計是
**只有總機坐在 General**，所以 General 本來不預期出現別人的發言。
但 General 就是大廳，大家自然在大廳講話 —— 那不是壞習慣，是動線本來就該這樣。

**修法**：`_reply_channel` 加第三態 `origin`。判準是**有沒有人正在那個 topic 等**：

| 情境 | 回哪裡 |
|------|--------|
| 使用者在某 topic 發問（含 General） | **回同一個 topic** |
| 由該次發問**派工出去的下游** | **同上**（origin 沿派工鏈傳遞） |
| 排程 / 自主輸出 | 維持歸屬 topic —— 「歸類」的價值不變 |

> 🔴 **派工鏈的傳遞是關鍵**，少了它這個修法解不了回報的問題：
> 使用者在 General 對入口 agent 發問，入口再 `send_to_instance` 派給 worker ——
> **worker 這一側從來沒收到過使用者訊息**，`_reply_channel` 是預設值 `topic`。
> 所以 `/api/send` 會把來源的 origin 傳給目標。
>
> 反向也處理了：`source` 是 `scheduler` / `system` 時**清掉** origin。
> 不清的後果是「使用者上午問過一次，之後每天的排程日報都被塞進他當時發問的 topic」——
> 那不是對話連續，是污染。

**這個區分套件裡本來就有先例** —— `last_inbound_source`（`user` vs `system`，1.2.16 為了
區分 `AWAITING_REPLY` / `IDLE` 而加）就是同一個概念。不需要新的追蹤機制。

**刻意不做的**：兩邊都發（General + agent topic）。訊息數翻倍會踩 rate limit，
而 General 變雜訊區之後使用者還是只看一個地方。

### 📚 新增 `_builtin:ingest-queue` —— 把開放式任務變成有明確輸入的清單

**回報**：「自我成長機制存在但處於休眠狀態。」現象正確，歸因可以更精確。

不是「沒有新任務 = 沒有新學習」，而是**沉澱被設計成一個沒有明確輸入的任務**：
排程說「請把學到的寫進 `knowledge/raw/`」，agent 收到後得自己回想今天學到什麼 ——
而它的 session 可能已經 compact 過，回想不了。

> 🔴 **而下游會回報成功**：ingest job 掃自己提詞裡寫死的路徑、找不到新檔、
> 回報「新增 0 篇」—— 看起來一切正常。
> 實測有個部署的 ingest 提詞只寫 `agents/*/knowledge/raw/`，
> 而待萃取的 309 個原始檔在**根目錄的** `knowledge/github/raw/`
> → **永遠不在掃描範圍內，而 job 照樣回報成功。掃不到東西卻回報成功 = 假綠燈。**
> 路徑寫死在提詞裡就會發生這種事，所以改由套件掃。

**這個 job 只做一件事**：機械掃出「檔名尚未出現在同櫃 `index.md`」的原始檔，
把**清單本身**當成任務內容發出去。萃取仍然由 agent 做（那需要 LLM）。

```
現在： 「請沉澱知識」                     → agent 不知道從哪開始
之後： 「這 3 個檔還沒 ingest：A、B、C」   → 有明確輸入
```

```yaml
- name: ingest-queue
  cron: "0 22 * * *"
  target: _builtin:ingest-queue
  prompt: "librarian-agent:5"      # <agent>:<每次幾個>，預設 5
```

三個實作決策：

1. **判斷依據是「檔名有沒有出現在 `index.md`」**，刻意不用 mtime 或狀態檔 ——
   mtime 會因 `git pull` / `cp` 全體更新，一次噴出幾百個假新檔；
   狀態檔會與實際 index 漂移，而 `index.md` 本來就是知識庫的真相來源。
2. **掃描範圍含根目錄與各 instance 的 `working_directory`** —— 不是只掃 `agents/*/`（見上）。
3. **有批量上限（預設 5）** —— 一次丟 309 個檔跟丟 0 個一樣沒用，都會讓 agent
   不知從何下手。**沒有待處理時不叫醒任何人**（無效喚醒會 spawn lazy worker，有成本）。

### 測試

`tests/test_origin_routing.py` 11 個 + `tests/test_ingest_queue.py` 14 個。
全量 **1504 passed**。

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
