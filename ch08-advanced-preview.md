# 第八章：進階預覽 — MCP、Commands、Agents、Skills

> 這一章是預覽。完整的 Agent 架構設計、Skills 系統設計、MCP 整合實戰，會在進階班詳細教。

前七章教你跟 Claude Code 工作：對話、改 code、配置環境、跑測試。這些是基礎。

但如果你跟我一樣，用了幾個月之後，你會開始問一個不同的問題：**我能不能讓 Claude 不只是幫手，而是一整個會自動運作的系統？**

這一章預覽四個讓 Claude Code 從「工具」升級成「系統」的機制。

---

## 8.1 MCP — 讓 Claude 連接外部世界

Claude Code 預設只能讀寫你本地的檔案。這已經很有用了，但有時候不夠。

**MCP（Model Context Protocol）** 是一個標準協議，讓 Claude 可以透過外部 Server 存取更多資料和服務。加了 MCP，Claude 就可以讀你 GitHub 上的 Issue、查你的 DB schema、搜尋網路上的最新資料。

設定方式很直觀，在專案根目錄建立 `.mcp.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxx"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

設定之後，你就可以直接問 Claude：「看一下 GitHub 上最近有哪些 open bug？」或「幫我分析 users table 的 schema，有沒有缺 index？」—— Claude 會透過 MCP 去拿真實資料，再給你分析。

我自己常用的 MCP Server：

| MCP Server | 能力 | 什麼情況用 |
|-----------|------|----------|
| `server-github` | Issue、PR、Review、Commit | 幾乎每天用，幾乎必開 |
| `server-postgres` | DB schema、query 結果 | 後端開發時才開 |
| `server-slack` | 讀訊息、發通知 | 需要把 Claude 的輸出送到 Slack |
| `server-linear` | Ticket、Sprint、Issues | 管理 epic 和 story 時 |
| `server-brave-search` | 網路搜尋 | 需要即時資訊、文件查找 |
| `server-playwright` | 控制瀏覽器 | 自動化測試、抓網頁資料 |

### 三個坑

用了一段時間，我踩過的坑主要是三個。

**坑一：token 爆量。** 每個 MCP Server 啟動後會佔用 context window。一個大概 500–2000 token，開 5 個就是 5000–10000 token 的額外消耗。一個對話裡 context 變短，Claude 就「記不住」之前說過的事情。

**坑二：secrets 外洩。** `.mcp.json` 裡面有 API token 和資料庫密碼。預設它不在 `.gitignore` 裡面，一不小心就 commit 上去了。我的做法是：建完 `.mcp.json` 的第一件事就是把它加進 `.gitignore`，然後用環境變數或 secret manager 管理金鑰。

**坑三：啟動慢。** 每個 MCP Server 啟動需要 1–5 秒。開太多，你在等 Claude 初始化時會明顯感覺到。我的習慣是：今天在做什麼任務，就只開那個任務需要的 MCP。GitHub MCP 幾乎必開，DB MCP 只在後端開發時開，其他按需要。

---

## 8.2 Commands — 可重用的工作流

你有沒有發現自己在 Claude 對話框裡貼同樣的 prompt，一遍又一遍？

「幫我用中文 review 這個 PR，注意安全漏洞和命名規範」。每次都打，每次打法還不太一樣。

**Custom Slash Commands** 解決這個問題。把常用的 prompt 存成 Markdown 檔案，下次用 `/指令名稱` 一鍵觸發。

```
.claude/commands/
├── review.md          → /review
├── test-plan.md       → /test-plan
└── deploy-check.md    → /deploy-check
```

`.claude/commands/review.md` 的內容很簡單：

```markdown
---
description: "Code review：安全 + 命名 + 品質"
---

請用繁體中文 review 以下程式碼：

1. 安全漏洞（OWASP Top 10）
2. 命名規範是否符合 CLAUDE.md
3. 有沒有多餘的 console.log 或 debug code
4. 具體的改進建議

目標：$ARGUMENTS
```

使用時只需要：

```
/review src/api/users.ts
```

`$ARGUMENTS` 會被替換成你輸入的檔案路徑。

這些 Command 檔案可以 commit 到 git，整個團隊共享同一套工作流標準。這是我覺得 Commands 最有價值的地方——不是讓你少打字，而是讓團隊的最佳實踐變成可執行的指令。

我的 content-asset-system 裡有 69 個 slash commands，組織成幾個命名空間：

- `/yt-init`、`/yt-generate`、`/yt-analyze` — YouTube 影片工作流
- `/brain-search`、`/brain-transform`、`/brain-index` — RAG 知識庫操作
- `/pulse-post`、`/pulse-check` — 社群媒體排程
- `/consulting-init`、`/consulting-kpi` — 顧問客戶管理

每個指令背後都是一段精煉過的 prompt，把我對某類任務的最佳實踐封裝進去。進階班會教怎麼設計一套可以長期維護的 Command 系統。

---

## 8.3 Agents — 專業角色定義

**Custom Agents** 讓你定義有特定角色和職責的 AI。

跟在對話框裡直接問不同，Agent 有明確的「人設」：它只做什麼、不做什麼、用什麼工具、怎麼回應。這讓同一套任務的執行更一致。

`.claude/agents/code-reviewer.md` 的結構：

```markdown
---
description: "資深 Code Reviewer"
---

你是一位專注程式碼品質和安全性的 Code Reviewer。

## 你的職責
- 用建設性的方式指出問題
- 每個問題都給出具體的修改範例
- 不只說「有問題」，要說「建議改成這樣」

## 你不做的事
- 不直接修改程式碼（只建議，讓人類決定）
- 不跑測試（那是 Test Runner 的工作）
- 不設計新功能（那是架構師的工作）
```

Agent 跟 Command 最大的差別：Command 是一段 prompt，Agent 是一個有職責邊界的角色，可以搭配其他 Agent 協作。

我目前有幾個常用的 Agent：

- **Code Reviewer** — 只 review，不改，給具體建議
- **Debugger** — 根因分析，不是「幫你改掉」，是「告訴你為什麼壞了」
- **Test Runner** — 跑測試、解讀錯誤、回報結果
- **Content Writer** — 根據 Alex 的 voice DNA 寫文章，有特定的禁用詞和寫作風格

更進階的玩法是把多個 Agent 組成 **Agent Team**。我有三個不同的系統：

| 系統 | Agent 數量 | 主要任務 |
|------|-----------|---------|
| **Hera** (ai-web-template) | 7 agents | 網站設計、前端開發 |
| **Athena** (ai-coding-template) | 9 agents | 全端開發、測試、部署 |
| **Gaia** (content-asset-system) | 4 agents | 內容製作、發佈、分析 |

Hera 系統裡，`/hera:build` 一個指令可以從設計稿跑到完整的靜態網站。Athena 系統裡，`/athena:loop` 會讓整個 team 自動推進當前的 epic。這些不是魔法——它們是一條條角色定義和工作流規則積累起來的結果。進階班會教完整的 Agent Team 設計方法。

---

## 8.4 Skills — 自動載入的知識

如果說 Commands 是「快捷鍵」，Agents 是「角色」，那 Skills 是「背景知識」。

**Skills** 是自動載入的知識模組。Claude 開始工作前會讀 Skill 定義，知道這個專案裡有哪些領域知識和工具可以用。你不需要每次都在對話裡解釋「我們的 blog 系統怎麼運作」、「n8n workflow 的規範是什麼」——Skill 幫你記住了。

Commands 和 Skills 的差別：

| | Command | Skill |
|--|---------|-------|
| 觸發方式 | `/指令名稱`，手動呼叫 | 自動載入，或自然語言觸發 |
| 適合 | 單一動作，快捷鍵 | 領域知識，持續性背景 |
| 內容 | 一段 prompt | 知識模組 + 資源檔案 |

我有四個 Skills，覆蓋四個領域：

- **content-pipeline** — 部落格 DNA、YouTube 工作流、SEO 規則、blog 發佈流程
- **python-service** — uv 套件管理、Typer CLI 寫法、pre-commit 設定、ruff 規範
- **linear-workflow** — branch 命名規則、commit message 格式、epic 同步、Linear CLI 用法
- **n8n-workflow** — n8n 架構邊界、data flow 規範、webhook patterns、節點命名規則

這些 Skill 讓 Claude 在任何對話裡都能直接引用正確的知識，不需要我每次解釋背景。

---

## 8.5 從工具到系統的演化

我不是一開始就有這些的。

**第一個月**，只有 CLAUDE.md。把專案的規則、命名慣例、禁用語法寫進去。這個階段的 Claude 是「有記憶的助手」。

**第三個月**，加了 10 個 Commands。把最常重複的工作流封裝成指令：code review、blog 生成、YouTube 描述生成。這個階段的 Claude 是「有工具的助手」。

**第六個月**，完整的 Agent Team + Skills + 4 層 Memory 系統。Commands 成長到 69 個，Skills 有 4 個，Agents 有 10 幾個分散在三個專案裡。這個階段的 Claude 更像是「一個會自己運作的工作流系統」。

**重點來了**：這不是一次設計出來的，是一條一條規則、一個一個 Command、慢慢積累的結果。你今天可以從最簡單的地方開始：找到你這週重複輸入超過三次的 prompt，把它存成一個 Command。就這樣。

從 CLAUDE.md 到 Commands，從 Commands 到 Agents，從 Agents 到 Agent Team——每一步都有它的時機，不需要跳步，也不需要一次全做。

---

## 進一步

如果這一章讓你對 Agent 架構、Skills 設計、MCP 整合有興趣，進階班會把這些做完整的拆解：

- 設計一套可以長期維護的 Command 命名系統
- Agent 職責邊界怎麼定義、怎麼避免角色混亂
- Skill 的資源管理：style guide、templates、config 怎麼組織
- Agent Team 的協作模式：串行、並行、Orchestrator pattern
- Memory 系統的五個層次：從 CLAUDE.md 到語意偵測

這本書教你從 0 到 1。進階班教你從 1 到 100。

加入 Skool 社群，和其他學員交流 → https://www.skool.com/ai-brain-alex/about?ref=5dde9b20e8e7432aa9a01df6e89685f4
