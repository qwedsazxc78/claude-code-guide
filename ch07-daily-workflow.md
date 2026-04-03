# 第七章：我的日常工作流與踩坑指南

你有沒有遇過這種情況？學了一堆工具，但每天打開 Claude Code 還是不知道從哪下手？

這一章不講功能，只講我自己每天怎麼用。數字都是真的，踩坑都是我自己踩的。你可以直接拿去模仿，然後慢慢改成自己的節奏。

---

## 7.1 Alex 的一天（真實數據）

我有 SRE/DevOps 背景，目前每天大概在 Claude Code 裡花 6 個小時以上，每天約 200-300 則訊息，用的是 Max 方案，每個月大概 200 美金。說貴嗎？省下來的工時遠超過這個數字。

一天的節奏大概長這樣：

```
07:00  開 VS Code
  │    Claude 自動讀取 CLAUDE.md
  │    SessionStart hook 載入 TASK.md（昨天在哪，今天繼續）
  │    看 Linear ticket，確認今天第一個任務
  │
  ├── 🔵 Plan Mode：描述需求，Claude 分析 + 規劃（5-10 分鐘）
  │   └── 我 review 計畫，調整方向，才讓它動手
  │
  ├── 🟢 Edit Mode：Claude 開始寫 code
  │   ├── 逐個 review inline diff
  │   ├── Accept 大部分、Reject 少部分
  │   └── 跟 Claude 討論 Reject 的原因
  │
  ├── 🟡 背景 Agent：同時跑 test suite，不佔主線對話
  │
  ├── 午休前：commit + push，Claude 幫我寫 commit message
  │
  ├── 下午：第二個任務（重複 Plan → Edit 循環）
  │   └── 遇到架構問題，切換 Opus，想清楚再切回來
  │
  ├── 傍晚：整理當天的會議筆記，讓 Claude 幫我摘要
  │
  └── 下班前 ritual
      ├── /compact 壓縮 context
      ├── 更新 TASK.md 紀錄今天到哪、明天從哪開始
      └── push 最後的改動
```

重點來了：每一天我都做同樣的事 —— Plan → Edit → Test，然後重複。沒有什麼神秘的。這個循環一天大概跑 3-5 輪，每輪一個 feature 或一個修復。

### 模型切換策略（具體數字）

我用三個模型，分配比例很固定：

| 任務 | 模型 | 比例 | 原因 |
|------|------|------|------|
| 日常 coding、debug | Sonnet 4.6 | 80% | 夠快夠好，不浪費額度 |
| 架構設計、複雜決策 | Opus 4.6 | 15% | 需要深度思考，值得多花時間 |
| 快速問答、確認小事 | Haiku 4.5 | 5% | 省額度，不用大砲打小鳥 |

切換方式：VS Code 用 `Cmd+P`（Mac）或 `Meta+P`（Linux/Win），CLI 用 `/model`。不需要重開對話，幾秒鐘的事。

### Context 管理的 rhythm

每 30-40 則訊息我就會跑一次 `/compact`。訊號很明顯：Claude 開始重複一樣的話、命名規範開始漂移、或者回應變得很廢話。這些都是 context 快滿的症狀。

`/compact` 之後，Claude 會把對話壓縮成摘要，繼續在同一個 context 裡工作。不用重開，不用重新解釋背景。

換任務的時候，我不用 `/compact`，直接開新 tab。不同任務不共享 context，思路更清楚。

---

## 7.2 七大最佳實踐

這七條是我用了一年多，踩過坑之後留下來的。每條都有「為什麼」。

### 1. Plan before Code（3+ 檔案的改動必用）

直接讓 Claude 動手的結果：改了 15 個檔案才發現方向錯了，然後花 30 分鐘 rewind。先進 Plan Mode 的結果：5 分鐘 review 計畫，一次過。

規則很簡單：改動會碰到 3 個以上的檔案，就先 Plan。改一個小函式可以直接 Edit，但凡是跨模組的改動，一律先 Plan。

### 2. CLAUDE.md 是靈魂（新專案第一件事）

沒有 CLAUDE.md 的結果：Claude 每次都在猜你的規範，今天用 camelCase，明天用 snake_case，後天在問你偏好哪個。

花 30 分鐘寫好 CLAUDE.md 的結果：未來 30 小時不用再解釋。Claude 在這個專案裡的所有行為都對了。

新專案第一件事不是寫 code，是寫 CLAUDE.md。這投資比任何其他事都划算。

### 3. 善用 Checkpoint（大膽試，不好就 rewind）

很多人不敢讓 Claude 做大改動，因為怕改壞了。這個擔心是多餘的，因為有 Checkpoint。

工作流是這樣的：讓 Claude 大膽做 → 跑完看結果 → 不滿意就 Rewind → 重新下更精確的指令。Rewind 比硬著頭皮修 Claude 改壞的東西快太多了。

我一天平均用 1-2 次 Checkpoint/Rewind。不是出事才用，是主動用。

### 4. 平行化背景任務

串行的工作方式：寫完 code → 跑測試 → 等 3 分鐘 → 繼續寫。每天加起來可能浪費了一個小時在等。

平行的工作方式：主線繼續寫下一個功能，背景 Agent 在跑測試。測試完 Background Agent 通知你，你去看結果。兩件事同時進行。

獨立的任務交給背景做，自己專注在需要思考的地方。

### 5. MCP 按需載入（不要全掛）

一次掛 10 個 MCP 的結果：Claude 的 context 被工具說明佔掉一大塊，回應變慢，思路變散。

按需載入的方式：今天要查 GitHub PR，開 GitHub MCP；今天要用 Notion，開 Notion MCP；其他不用的，關掉。

我日常只常駐 GitHub MCP，其他的任務開始前才掛，任務完成後關掉。

### 6. 用 Hooks 自動化重複工作

如果你做同一件事超過 3 次，就應該寫成 Hook。

我有一個 PostToolUse hook 在每次 Claude 寫完 code 之後自動跑 `prettier`。另一個 SessionStart hook 在每次開啟 Claude Code 時自動讀取 TASK.md，讓 Claude 知道昨天停在哪裡。這兩個 hook 省了我每天至少 20 分鐘的手動動作。

覺得麻煩才設一次，設完之後它就幫你一直跑。

### 7. 把 CLAUDE.md + settings.json commit to git

這條特別針對有團隊的情境。如果每個人各自設定，規範就不統一，Claude 在不同人的電腦上行為不一樣。

把 `.claude/settings.json` 和 `CLAUDE.md` commit 進去，團隊成員 clone 下來就自動生效。這是讓 AI 工作流可以規模化的關鍵。

注意一件事：commit 前確認 `settings.json` 裡面沒有 API Key 或個人 token。

---

## 7.3 七大踩坑指南

每個開始用 Claude Code 的人都會踩到幾個坑。提前看一遍，遇到的時候就知道怎麼處理。

### 坑 1：Claude 改錯檔案

症狀是 Claude 動了不該動的檔案，或者用了錯的命名規範。

原因很簡單：CLAUDE.md 沒說清楚。Claude 不是故意搗亂，它是在用它猜出來的規範工作。

解法：在 CLAUDE.md 加上限制區塊。「不要修改 `config/` 底下的任何檔案」、「命名規範：資料夾用 kebab-case，Python 變數用 snake_case」。越具體越好，不要靠 Claude 去猜。

### 坑 2：一直被問權限

每次跑 test Claude 都要問你確認，每次讀某個檔案都要停下來問。這個很煩，也很打斷工作流。

原因是沒設 allow list。Claude Code 預設保守，它不知道哪些操作你覺得安全。

解法在 `settings.json`：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test)",
      "Bash(uv run pytest)",
      "Read"
    ]
  }
}
```

把你覺得安全的操作白名單化，Claude 就不會再問了。

### 坑 3：Context 爆掉

症狀：Claude 開始忘東忘西、回應變慢、開始重複它剛才說過的話、命名規範開始漂移。

原因：對話太長，或者掛了太多 MCP 把 context 佔滿了。

解法分兩種：如果只是對話長了，用 `/compact` 壓縮，繼續同一個對話。如果是換了完全不同的任務，直接開新 tab，乾淨的 context 思路更清晰。如果掛了很多 MCP，先關掉不用的。

我的規則是：30-40 則訊息跑一次 `/compact`，看到 Claude 開始漂移就立刻跑，不要等到問題變嚴重。

### 坑 4：改太兇

症狀：Claude 一次改了 20 個檔案，或者加了很多你沒要求的東西。

原因通常是兩個：一是 Auto-Accept 開著，Claude 沒有 review 機制就一路改到底；二是指令太模糊，Claude 自由發揮了。

解法：重要任務一律用 Edit Mode，每個 diff 都 review 才 accept。指令要具體，「優化這個函式」不夠，「把這個函式的時間複雜度從 O(n²) 改成 O(n log n)，不要動其他地方」才夠。

Auto-Accept 爽一時，被改壞一次就知道了。我現在 Auto-Accept 只用在跑 test 或格式化，其他一律 Edit Mode。

### 坑 5：Hooks 擋住操作

症狀：Claude 做什麼都被擋，或者每個動作都慢了好幾秒。

原因是 Hook script 有 bug，或者 Hook 本身太複雜跑太慢。

解法：暫時在 `settings.json` 加上 `"disableAllHooks": true`，確認是不是 Hook 的問題。確認之後，去修那個有問題的 Hook script，或者簡化它、加上 timeout。Hook 應該要快，最好在 1 秒內跑完。

### 坑 6：額度用完了

症狀：Claude 提示你要用 API credit 了，或者帳單突然變很貴。

Pro 方案有使用量上限，重度使用者通常每 2-3 小時就會碰到 rate limit。我的解法是多開幾個 Claude Code tab，一個 tab 被限速的時候切到另一個繼續工作。同時用 `/status` 確認目前的使用狀況。

如果你每天用量很大，認真考慮升 Max 方案。我算過，省下的工時遠超過費用差異。

還有一個常見問題是不小心用到 API credit 而不是訂閱額度。用 `/status` 看清楚你在用哪個 quota。

### 坑 7：Diff 看不懂

症狀：Claude 改完，diff 太大，根本看不完，直接 Accept All。

這個做法短期省事，長期埋雷。看不懂 diff 的根本原因是一次改太多。

解法：把任務拆小。與其讓 Claude 一次重構整個模組，不如拆成「先改資料結構」、「再改邏輯層」、「最後改 API 層」。每次改 3-5 個檔案，diff 就可以在 30 秒內掃完。每次都 review，養成習慣。

---

## 7.4 漸進式導入建議

不要一口氣學全部。Claude Code 功能很多，一次全開會讓你不知道從哪用起。

**第一週：只用 Chat + Debug**

每天用 Claude 回答技術問題、解釋 error message、討論解法。不讓它動你的 code，只是對話。這一週你在建立對 Claude 的信任感，了解它的能力範圍和限制。

**第二週：加入 Plan Mode + Edit Mode**

開始讓 Claude 動手改 code，但先用 Plan Mode 確認方向，再用 Edit Mode 逐個 review diff。這一週你在學習如何給好指令，以及如何 review AI 的改動。開始用 Checkpoint/Rewind，知道後悔也來得及。

**第三週：寫 CLAUDE.md + 加入 Hooks**

花時間把你的工作習慣寫進 CLAUDE.md。觀察自己每天在重複做哪些事，把它們做成 Hook。這一週投入的時間，之後每天都會收到回報。

**第四週：Background Agent + CI/CD**

開始用背景 Agent 跑測試、生文件、做平行任務。如果你有 CI/CD pipeline，試著把 Claude Code 整合進去做自動化 review 或 test generation。這一週你從「工具使用者」變成「工作流設計者」。

一個月後你就會有自己的節奏，不再需要看這份指南。

---

如果你已經跑完這四週，想帶走真的可以上線的成果，完整版 Workshop 有三個交付 Lab：60 分鐘做一個上線的個人網站、45 分鐘做一個自動化工具、45 分鐘做一份專業研究報告。不是練習，是真的可以用的東西。

第八章我們進進階地帶：multi-agent 系統、CLAUDE.md 的深度設計、還有 Gaia 這套內容飛輪系統的完整拆解。
