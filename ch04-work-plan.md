# 第四章：Work — 讓 Claude 動手改 Code

聊天是跟 Claude 溝通想法，Work 是讓它真的動手。

這兩件事差很多。讓 AI 改你的 code，需要一套比聊天更嚴謹的工作方式。否則你會遇到一個經典場景：Claude 改了 20 個檔案，你才發現方向完全錯了。

這一章的目標很簡單：讓你有信心說「Claude，你來改」，同時不失去對 code 的掌控。

---

## 4.1 三種模式：Edit / Auto-Accept / Plan

Claude Code 有三種工作模式，用 **Shift+Tab** 循環切換。底部狀態列會顯示目前是哪個模式。

**Edit Mode（預設）**

Claude 每次修改前都會問你。它會顯示 diff，你決定 Accept 還是 Reject，然後才繼續下一步。這是最安全的模式。

**Auto-Accept Mode**

Claude 改完之後直接套用，不問你。速度快很多，但你要對自己說的需求有把握。適合重複性作業、測試檔案、或者你已經在 Plan Mode 確認過方向的後續執行。

**Plan Mode**

Claude 只分析，不動任何檔案。它會告訴你打算怎麼做，但不真的做。這是安全程度最高的模式，也是我最常用的起點。

三種模式的安全比較：

| 模式 | Claude 的行為 | 安全度 | 適用場景 |
|------|-------------|--------|---------|
| Edit | 改前問你 | ⭐⭐⭐ | 學習中、重要檔案、生產環境 |
| Auto-Accept | 直接套用 | ⭐⭐ | 重複作業、測試檔、已確認方向 |
| Plan | 只規劃不改 | ⭐⭐⭐⭐ | 新功能、架構評估、大改動前 |

簡單來說：小改動用 Edit，大改動先 Plan 再 Edit，信任的重複作業才考慮 Auto-Accept。

我的使用比例大概是：Plan Mode 40%、Edit Mode 50%、Auto-Accept 10%。很多人一開始會覺得 Auto-Accept 很爽，但被 Claude 改壞兩次之後就學乖了。我通常只在改 test files 或者格式化的時候用 Auto-Accept。

---

## 4.2 Inline Diff：逐行 Review AI 的修改

當 Claude 在 Edit Mode 修改檔案，VS Code 會顯示 side-by-side 的 inline diff。

讀法很直觀：
- **綠色** = 新增的程式碼
- **紅色** = 刪除的程式碼
- 每個改動區塊都有 Accept / Reject 按鈕

你有三個選擇：全部接受、全部拒絕、或逐個決定。

逐個決定是我最推薦的方式。來看一個我自己的例子：

我在做 Gaia CLI 的 search command 重構時，讓 Claude 把 5 個 search 相關的函式從散落在不同檔案整合到 `scripts/search/` 目錄。Claude 改了 12 個檔案，我逐個 review diff。第 8 個檔案我發現它把一個 import 路徑改錯了 —— 指向了一個不存在的 module。Reject 之後告訴它正確的路徑，3 秒改好。如果我 Accept All，這個 bug 可能要花 30 分鐘才找得到。

Claude 不會生氣，也不會重做全部。它只會修那個你 Reject 的地方。

Review 的時候有幾個特別要注意的點：超過 50 行的改動要仔細看，紅色刪除的部分要確認不是誤刪，還有 import — Claude 有時候加了新的 import 但沒真的用到。

不要無腦 Accept All。掃個 10 秒，就能抓到大多數的問題。你的 review 是最後一道防線。

---

## 4.3 Checkpoint & Rewind：不滿意就時間旅行

這是 Claude Code VS Code Extension 最讓我驚喜的功能。

每當 Claude 修改檔案，Extension 會自動建立 checkpoint——記錄哪些檔案被改了、改之前是什麼、對應到哪則對話訊息。你完全不需要手動操作。

想回溯的時候，把滑鼠移到任何一則 Claude 的回覆訊息上，會出現 ⏪ 按鈕。點進去有三個選項：

| 選項 | 程式碼 | 對話 | 適用情境 |
|------|--------|------|---------|
| Fork conversation | 不動 | 從這裡分支 | 想試試不同問法 |
| Rewind code | 回溯 | 保留完整 | 這個方案不行，重來 |
| Fork + Rewind | 回溯 | 從這裡分支 | 完全重新開始 |

來看一個很常見的情境：

你說「幫我加一個 caching layer」，Claude 改了 8 個檔案——安裝了 Redis、加了 redis client、改了 5 個 service 的 query、加了 cache invalidation 邏輯。

然後你看了一眼說：「等等，不用 Redis 這麼重，太複雜了。」

沒有 Rewind，你要做什麼？`git stash`，然後還要手動找哪些檔案被動過，新安裝的 package 也要自己刪。

有 Rewind，你把滑鼠移到那則對話訊息上，點 ⏪，選 Rewind code，8 個檔案全部回到原來的狀態。然後你說「用 in-memory cache 就好，不要 Redis」，Claude 這次用 Map 做了一個簡單的 LRU cache。

有了 Rewind 之後，你對 Claude 的態度可以大膽很多。以前怕改壞，所以每個改動都要很保守。現在你可以說「大膽做！反正不好我就 rewind。」這讓你可以更積極地探索不同方案。

---

## 4.4 Plan Mode：先想再做的開發方法

Plan Mode 的核心概念很簡單：讓 Claude 先寫企劃書，再開工。

進入 Plan Mode 之後，Claude 會讀你的 codebase，分析你的需求，然後寫出一份詳細的計畫——但不動任何一個檔案。

一個好的 Plan 長這樣：

```
需求：加一個 notification system，支援 email、push、in-app 三種管道

建議目錄結構：
  src/services/notification/
  ├── email.ts
  ├── push.ts
  ├── in-app.ts
  └── index.ts

需要新增的檔案：4 個
需要修改的現有檔案：2 個（routes.ts, app.ts）
資料庫 schema 變更：notifications table
相依性：nodemailer（email）、firebase-admin（push）
風險：現有 user service 需要注入 notification service，注意 circular dependency
執行順序：schema → in-app → push → email → routes
```

看到這份計畫之後，你說：「email 的部分先不做，先做 push 和 in-app 就好。資料庫用 MongoDB 不要 PostgreSQL。」

Claude 更新計畫，你確認，然後 Shift+Tab 切回 Edit Mode，說「按照剛才的計畫開始做」。

**什麼時候 Plan Mode 是必要的：**

- 新功能涉及 3 個以上的檔案 → 用 Plan Mode
- 大型重構、架構變更 → 用 Plan Mode
- Code Review（只看不改）→ 用 Plan Mode
- 修一個小 bug、改個名稱 → 直接 Edit

我最近在設計 Gaia 的 Linear 整合功能。進 Plan Mode 問：「我要讓 Gaia CLI 能直接 push issues 到 Linear，支援 GAI 和 CLO 兩個 team，需要哪些改動？」Claude 掃完整個 codebase，列出了：需要新增 `linear-workspace.yaml` config、修改 CLI entry point、加 4 個新的 subcommands（push、issues、update、sync）。它甚至提醒我現有的 epic markdown 格式需要跟 Linear issue 保持同步。這就是 Plan Mode 的價值 —— 它不只告訴你要改什麼，還告訴你你沒想到什麼。

我用 Claude Code 管理超過 20 個 n8n workflow 的 JSON 備份。每次 n8n 大版本更新，我讓 Claude 用 Plan Mode 比對新舊 JSON，找出 breaking changes。這是 Plan Mode 的一個非典型但非常實用的用法。

Plan Mode 還有一個我很喜歡的副作用：它幫你把模糊的想法文字化。

很多時候你腦中有個想法，但其實沒想清楚。讓 Claude 寫成計畫之後，你會發現「啊，原來我漏想了這個」或者「原來這個比我想的複雜很多」。Plan Mode 不只是給 Claude 看的，也是給你自己看的。

---

## 4.5 實戰：從一句話到一個完整功能

把前面四個工具串在一起，跑一遍完整的開發流程。

**情境：你要在現有的 API 裡加一個「使用者活動紀錄」功能。**

**Step 1：Plan**

Shift+Tab 切到 Plan Mode，輸入：

```
加一個 user activity log 功能。
每當使用者登入、修改資料、刪除帳號，
都要記錄時間、動作類型、IP 位址。
要能查詢某個使用者的完整活動歷史。
```

Claude 分析你的 codebase，輸出計畫：需要新增 `ActivityLog` model、修改 auth middleware 和 user controller、加一個新的 GET endpoint。

**Step 2：Review 計畫，調整方向**

你看了計畫，覺得 IP 位址的部分不需要，而且想用 MongoDB 而不是現有的 PostgreSQL。

```
IP 位址不用記，其他 OK。資料庫改用 MongoDB，
我已經有一個 mongoose connection 在 src/db/mongo.ts。
```

Claude 更新計畫，改用 Mongoose 的 schema 設計。

**Step 3：Edit Mode 執行**

Shift+Tab 切回 Edit Mode。

```
按照剛才的計畫開始做，先從 ActivityLog model 開始。
```

Claude 開始改 code，每個檔案都顯示 diff 讓你 review。你逐個 Accept，遇到不對的地方就 Reject 並說明原因。

**Step 4：跑測試，確認**

```
請 Claude 執行測試：pytest tests/
```

Claude 看 terminal 輸出，如果有 failing test，直接幫你修。

### 如果你用 TypeScript / Express

同樣的流程，換個語言：

**Step 1：Plan Mode**

```
我需要在 Express API 加一個 GET /api/health endpoint，
回傳 { status: "ok", uptime: process.uptime(), timestamp: new Date() }。
加上對應的 Jest test。
```

Claude 會列出：修改 `src/routes/index.ts`、新增 `src/routes/health.ts`、新增 `tests/health.test.ts`。

**Step 2-4：同樣的 Review → Edit → Test 流程。**

你有沒有注意到？需求描述的方式、Plan Review 的方式、看 diff 的方式，都一樣。只有最後跑測試從 `pytest` 換成 `jest`。

語言不同，工作流一樣。Plan Mode 不在乎你用什麼框架。

---

整個流程下來，你從一句需求描述，到有一個運作正確的功能，沒有盲目接受任何一行 code，也沒有在「方向錯了然後 undo 很久」的地方浪費時間。

你可能會問：用了 Plan → Review → Edit，開發速度真的有變快嗎？我的感覺是：小功能大概快 3-5 倍，大功能大概快 2 倍。但最大的差距不是速度，是信心。以前大改動之前我會猶豫半天，現在我直接說「Claude，先規劃一下」，看完計畫就知道該不該做。決策速度快了，開發速度自然就快了。

**重點來了**：Plan → Review → Edit → Test 這四個步驟，就是 AI 時代的開發節奏。Checkpoint 是你的安全網，Inline Diff 是你的品質關卡，Plan Mode 是你的方向確認機制。三個工具一起用，你對 Claude 的掌控度其實比你想的高很多。

第五章我們聊 CLAUDE.md——如何把你的規範寫進 Claude 的記憶，讓每次對話都從正確的起點開始。

---
