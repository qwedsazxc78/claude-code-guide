# 第三章：Chat — 跟 Claude 聊天的藝術

*問對問題，比寫對 code 更重要*

很多人用 Claude Code 的方式，跟在 ChatGPT 貼問題沒兩樣。

複製 error message，貼過去，等答案。複製答案，貼回去，試試看。不行，再貼一次。

這個流程有個致命問題：Claude 根本不知道你的 code 長什麼樣。它給的答案，可能完全不適合你的專案。

Claude Code 裡的對話，運作方式完全不同。

---

## 3.1 不只是聊天 — Claude 會讀你的整個專案

當你在 Claude Code 裡問問題，Claude 不是在真空中回答你。

它知道你的專案結構。它知道你用的是 TypeScript 還是 Python。它知道你的 `tsconfig.json` 長什麼樣。如果你問「為什麼這個 API 一直 401？」它會去看你的 auth middleware，找到實際的問題所在。

這不是魔法，是因為 Claude 在啟動時就掃描了你的 workspace。它有整個專案的上下文。

實際意義是什麼？你可以問更自然的問題。

不用說「我有一個 Express app，有個 middleware 負責驗證 JWT token，程式碼大概長這樣...」，你只要說「auth middleware 哪裡有問題？」它就知道你在說哪個檔案。

這是真正的「AI 助理」，不是「AI 搜尋引擎」。

舉個例子：我的 content-asset-system 有 69 個 slash commands，每個都是一個 .md 檔。我問 Claude「幫我看一下 /pulse-post 和 /pulse-check 有沒有重複邏輯」，它自己去讀了兩個 command 檔案、找到了共用的 validation 邏輯，然後建議我抽成一個 shared util。整個過程我沒貼任何一行 code。

### 一段完整的對話範例

讓我展示一個真實的 debug 對話流程。假設你接手了一個 FastAPI 專案，API 回傳速度突然變慢：

**你的 prompt：**
> @src/api/endpoints.py 這個 GET /users endpoint 最近回傳變很慢，大概 3-5 秒。可以幫我看一下嗎？

**Claude 的回覆（大約 10 秒後）：**
> 我看了 endpoints.py 第 45-62 行的 `get_users` 函式。有兩個問題：
>
> **問題一**：第 52 行 `users = db.query(User).all()` 會載入所有 User 的所有欄位，包含 `profile_image`（BLOB 欄位）。建議改成只撈需要的欄位：
> `users = db.query(User.id, User.name, User.email).limit(50).all()`
>
> **問題二**：第 58 行有 N+1 query — 每個 user 都會額外查一次 `department` table。建議用 `joinedload`：
> `users = db.query(User).options(joinedload(User.department)).limit(50).all()`

**你的 follow-up：**
> 改第一個就好，N+1 那個我知道，那是 feature 不是 bug（部門資料是 lazy load 設計的）

**Claude 的回覆：**
> 了解，只改 BLOB 的部分。
> [直接顯示 diff，只動那一行]

這就是 Claude Code 跟 ChatGPT 的核心差異：它讀了你的程式碼，理解了上下文，提出精準的建議。你說不改 N+1，它就不改——不會堅持、不會再問一次。你不需要複製貼上任何東西，也不需要解釋你的架構設計決策。

---

## 3.2 @mention：精準指定上下文

Claude 雖然知道你的專案，但它不可能把所有檔案都放進對話。Token 有限，要精挑細選。

`@mention` 讓你主動告訴 Claude：「這個任務，特別需要看這些東西。」

**@filename** — 帶入一個完整檔案：

```
@src/auth/middleware.ts 這個 middleware 有沒有 security 問題？
```

**@filename#5-10** — 只帶入特定行數，節省 token（語法請以官方文件為準，版本可能有差異）：

```
@src/api/users.ts#45-80 這段 query 邏輯有沒有 N+1 的問題？
```

**@terminal:bash** — 把 terminal 輸出帶進來（debug 的神器，後面會詳細說）：

```
@terminal:bash 這個 error 怎麼修？
```

（注意：`@mention` 的語法會隨版本更新，Claude Code 的 `@` 主要用於引用檔案和 MCP 資源。Terminal 輸出的引用方式請以 [官方文件](https://code.claude.com/docs/en/interactive-mode) 為準。）

**Alt+K** — 快捷鍵直接開啟 @mention 選單，不用手打路徑。

最強的 @mention 用法是指向你的 CLAUDE.md。我的 CLAUDE.md 有 28KB，定義了所有命名規範、status values、frontmatter schema。每次開新對話，我會先 @CLAUDE.md 確保 Claude 讀過規範。這一個動作就能避免 80% 的「Claude 不遵守我的規範」問題。

你有沒有遇過這種情況？給 Claude 一個問題，它的回答跟你的實際 code 差了十萬八千里？通常是因為上下文不夠精確。用 `@mention` 把對的東西帶進來，答案的品質會差很多。

---

## 3.3 Extended Thinking：什麼時候讓 Claude 深度思考

Claude Code 有個功能叫 Extended Thinking。開啟之後，Claude 在回答前會先做很長的內部推理，把問題想清楚再說話。

聽起來很好，對吧？

實際上，不是每個問題都值得用。

**適合用 Extended Thinking 的情況：**

- 架構設計決策（「我要選 monorepo 還是 multi-repo？」）
- 複雜的 bug 追蹤（「這個 race condition 為什麼只在 production 出現？」）
- 效能瓶頸分析（「為什麼這個 endpoint 的 p99 latency 這麼高？」）
- 多方案比較（「用 Redis 還是 in-memory cache，我的情況哪個比較好？」）

**不適合用的情況：**

- 改個變數名稱
- 加幾行 comment
- 生成一個標準的 CRUD endpoint

深度思考要時間，也要更多 token。小問題開 Extended Thinking，只是讓自己等更久。

開啟方式是 **Alt+T** 切換。我的習慣是：問問題之前，先想一秒「這個問題需要深思熟慮嗎？」需要就開，不需要就不開。

Extended Thinking 大約多花 2-3 倍的 token。我平常一天用 200 則訊息，如果全開 Extended Thinking，大概只能撐 80 則。所以我大概只有 10% 的問題會開 —— 通常是跨模組的架構決策。

---

## 3.4 Multi-Tab：同時處理多個任務

我通常同時開 2-3 個對話分頁。

這對用過瀏覽器的人來說完全直覺：**Cmd+N**（Windows 是 Ctrl+N）開新對話，頂部的 tab bar 切換。每個 tab 有自己獨立的對話歷史、上下文和模式設定，不會互相干擾。

**三個最實用的 Multi-Tab 模式：**

**模式一：主線 + 探索**

一個 tab 在做正事，另一個 tab 在探索不確定的東西。探索完畢，把結論帶回主線繼續。不會讓探索的混亂污染正式的工作 context。

**模式二：Debug + 開發平行**

production 有個緊急 bug 要修，但手上還有新功能到一半不想中斷。兩個 tab，互不影響，各做各的。

**模式三：Code + Review**

一個 tab 在寫新的 code，另一個 tab 同時 review 另一個 PR。上下文完全分開，不會讓 review 的想法影響到寫 code 的思路。

還有一個工具要記：**`/compact`**。

對話太長之後，Claude 會開始「忘東忘西」——忘記你之前定的規則，重複說過的話。這是 context window 快滿了的信號。輸入 `/compact`，它會把對話壓縮：保留關鍵決策，丟掉冗餘過程，釋放空間。

我大約每 30-40 則對話 `/compact` 一次。把它想成清瀏覽器 cache，定期做就好。

有一個判斷信號：當 Claude 開始重複你之前已經決定過的事、或者忘記你的 naming convention，就是該 /compact 的時候。我曾經一個 session 聊到 80 則沒 compact，Claude 開始把 snake_case 寫成 camelCase，CLAUDE.md 裡明明寫得清清楚楚。/compact 完就恢復正常了。

---

## 3.5 實戰：30 秒修好一個 Bug

理論說完了，來實際跑一遍 debug 流程。

這是我每天在用的方式，從發現 error 到修好，通常不超過 30 秒。

上週我在 Gaia CLI 加了一個新的 search command，跑 pytest 出現 `TypeError: 'NoneType' object is not subscriptable`。我切到 Claude Panel，打 `@terminal:bash 這個 error 怎麼修？`。Claude 花 10 秒讀完 stack trace，找到是 Qdrant 回傳 None 但我沒做 null check，直接幫我加了防禦性程式碼。從看到紅字到修好，不到 30 秒。

這個流程最大的意義不是「快」，是「不用複製貼上」。你不需要描述你的程式碼結構，不需要解釋你在做什麼，不需要把 error message 格式化成 ChatGPT 看得懂的樣子。

Claude 在你旁邊，它看到你看到的東西。直接問就好。

---

## 3.6 不只工程師能用：客服逐字稿分析

Chat 不只是拿來 debug 的。這個案例讓你看到，不寫 code 的人也能用 Chat 做有價值的事。

假設你是產品經理，手上有一個月份的客服對話紀錄——50 個 `.txt` 檔案，每個都是一段客服和客戶的對話。你想知道：客戶到底在抱怨什麼？哪些問題最常出現？

以前的做法：一個一個打開，手動分類，做成表格。大概要花一整天。

用 Chat 的做法：

```
@support-logs/ 幫我分析這個資料夾裡所有客服對話紀錄，
找出 top 5 最常出現的客戶抱怨，
每個抱怨列出出現次數和代表性的原話引用。
```

Claude 會自動讀取所有檔案，分析內容，然後產出一份結構化的報告：

```
Top 5 客戶抱怨：

1. 退款流程太慢（出現 18 次）
   代表性原話：「我已經等了兩週還沒收到退款...」

2. App 登入頻繁失敗（出現 14 次）
   代表性原話：「每次開 App 都要重新登入...」

3. 客服回覆時間長（出現 11 次）
   ...
```

這個分析大概 2 分鐘就跑完。你不需要會寫程式，不需要會用 Excel 的 pivot table，只要會打中文描述你的需求。

這就是 Chat 的核心價值：**你給上下文，Claude 給分析。** 不管你是工程師還是產品經理，這個模式都一樣。

### 更多 Chat 使用場景

- **接手別人的 codebase**：問 Claude 解釋架構，省下數週讀 code 時間 → [Claude Code Legacy](https://claudcod.com/blog/claude-code-legacy-code/)
- **安全漏洞掃描**：Plan Mode 問 codebase 有無 SQL injection / XSS → [Claude Code Docs](https://code.claude.com/docs/en/overview)
- **技術債評估**：找出重複邏輯、缺測試、過時 patterns → [Plan Mode Guide](https://claudefa.st/blog/guide/mechanics/planning-modes/)
- **解讀複雜邏輯**：指定檔案行數，Claude 逐步解釋 regex 或遞迴函式 → [Common Workflows](https://code.claude.com/docs/en/common-workflows)

---

很多人問我「你每天 6 小時都在 Claude Code 裡做什麼？」答案是：80% 在 Chat 模式裡問問題。不是因為我不會寫 code，是因為讓 Claude 先看、先想、先建議，我再決定要不要做，這個效率比自己硬幹快太多了。

**重點來了**：Chat 功能的本質是「給 Claude 正確的上下文，得到正確的答案」。`@mention` 精準帶入上下文，Extended Thinking 處理複雜問題，Multi-Tab 管理平行任務，`@terminal` 直接 debug。不管你是工程師還是 PM，這四個工具組合在一起，就是真正的 AI 工作體驗。

下一章我們進 Cowork——不會寫 code 的人也能用 AI 處理本地檔案。

---
