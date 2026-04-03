# 第八章：進階 — MCP、Commands、Agents、Skills

前七章教你跟 Claude Code 工作：對話、改 code、配置環境、跑測試。這些是基礎。

用了幾個月之後，你會開始問一個不同的問題：**我能不能讓 Claude 不只是幫手，而是一整個會自動運作的系統？**

這一章不是預覽，是完整的拆解。讀完你應該能理解：一套有 69 個指令、多個 Agent 角色、4 個知識模組的系統，是怎麼從無到有建起來的。

---

## 8.1 MCP — 讓 Claude 連接外部世界

Claude Code 預設只能讀寫你本地的檔案。這已經很有用了，但有時候不夠。

**MCP（Model Context Protocol）** 是一個標準協議，讓 Claude 可以透過外部 Server 存取更多資料和服務。加了 MCP，Claude 就可以讀你 GitHub 上的 Issue、查你的 DB schema、搜尋網路上的最新資料。

### 設定方式

在專案根目錄建立 `.mcp.json`：

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

存檔之後，下次啟動 Claude Code，它就會自動連接這些 Server。你可以直接問：「看一下 GitHub 上最近有哪些 open bug？」或「幫我分析 users table 的 schema，有沒有缺 index？」—— Claude 會透過 MCP 拿真實資料，再給你分析。

### 實際設定 GitHub MCP：step by step

以 GitHub MCP 為例，這是最常用的一個，說明一下實際流程：

**第一步：取得 GitHub Personal Access Token**

到 GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens。權限至少要給 `Contents: Read`、`Issues: Read and Write`、`Pull requests: Read and Write`。

**第二步：在專案根目錄建立 `.mcp.json`**

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_你的token"
      }
    }
  }
}
```

**第三步：把 `.mcp.json` 加進 `.gitignore`**

這步一定要做，token 不能 commit 上去。

```
echo ".mcp.json" >> .gitignore
```

**第四步：重啟 Claude Code**

Claude Code 啟動時會讀 `.mcp.json`，初始化 MCP Server。你會在啟動 log 裡看到 `GitHub MCP server connected` 之類的訊息。

**第五步：驗證連線**

直接問 Claude：「列出這個 repo 最近 5 個 open issue」。如果回傳真實的 issue 清單，就成功了。

### 哪些 MCP Server 值得開？哪些要小心？

我自己常用的 MCP Server：

| MCP Server | 能力 | 什麼情況用 | 建議 |
|-----------|------|----------|------|
| `server-github` | Issue、PR、Review、Commit | 幾乎每天用 | 幾乎必開 |
| `server-postgres` | DB schema、query 結果 | 後端開發時才開 | 按需開，用完關 |
| `server-slack` | 讀訊息、發通知 | 需要把 Claude 的輸出送到 Slack | 小心寫入權限 |
| `server-linear` | Ticket、Sprint、Issues | 管理 epic 和 story 時 | 搭配 CLI 一起用 |
| `server-brave-search` | 網路搜尋 | 需要即時資訊、文件查找 | token 消耗大 |
| `server-playwright` | 控制瀏覽器 | 自動化測試、抓網頁資料 | 測試用很好 |

老實說：**GitHub MCP 是最值得投資的一個**。它讓 Claude 可以直接看你的 repo 狀態，不需要你手動複製貼上 issue 內容。其他的按需要開，不要為了「感覺功能齊全」就全部啟動。

### 三個真實的坑

**坑一：token 爆量。** 每個 MCP Server 啟動後會佔用 context window。一個大概 500–2000 token，開 5 個就是 5000–10000 token 的額外消耗。一個對話裡 context 變短，Claude 就「記不住」之前說過的事情。我的習慣是：今天在做什麼任務，就只開那個任務需要的 MCP。

**坑二：secrets 外洩。** `.mcp.json` 裡面有 API token 和資料庫密碼。預設它不在 `.gitignore` 裡面，一不小心就 commit 上去了。建完 `.mcp.json` 的第一件事就是加進 `.gitignore`。

**坑三：啟動慢。** 每個 MCP Server 啟動需要 1–5 秒。開太多，你在等 Claude 初始化時會明顯感覺到。不要貪多。

---

## 8.2 Commands — 可重用的工作流（完整拆解）

你有沒有發現自己在 Claude 對話框裡貼同樣的 prompt，一遍又一遍？

「幫我用中文 review 這個 PR，注意安全漏洞和命名規範」。每次都打，每次打法還不太一樣，Claude 給的結果品質也不穩定。

**Custom Slash Commands** 解決這個問題。把常用的 prompt 存成 Markdown 檔案，下次用 `/指令名稱` 一鍵觸發。

```
.claude/commands/
├── review.md          → /review
├── test-plan.md       → /test-plan
└── deploy-check.md    → /deploy-check
```

### 一個真實的 Command 檔案長什麼樣？

這是我 content-asset-system 裡的 `/yt-init` 指令，用來初始化一個新的 YouTube 影片工作資料夾。這是完整的 `.claude/commands/gaia/content/yt-init.md`：

```markdown
---
description: "[1/4] Initialize a new YouTube episode from scratch (create folder, content.md, generate AI content)"
argument-hint: <ep> <slug> "<title>" [--source <google_docs_url>] [--generate]
---

Initialize a new YouTube episode with full workflow support.

**Arguments:** $ARGUMENTS

## Quick Start Examples

\`\`\`bash
# Basic init (creates folder + content.md)
/yt-init 50 mcp-server "MCP Server 完整教學"

# With Google Docs source
/yt-init 50 mcp-server "MCP Server 完整教學" --source "https://docs.google.com/..."

# Init + auto generate AI content
/yt-init 50 mcp-server "MCP Server 完整教學" --generate
\`\`\`

## Workflow Steps

### Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- `ep` (required): Episode number (e.g., `50`)
- `slug` (required): URL-friendly slug (e.g., `mcp-server`)
- `title` (required): Video title in Chinese (in quotes)
- `--source` (optional): Google Docs URL for script
- `--generate` (optional): Auto-run AI content generation after init

### Step 2: Create Episode Structure

Use the `gaia` CLI:
\`\`\`bash
gaia youtube init {ep} {slug} "{title}"
\`\`\`

This creates:
\`\`\`
assets/youtube/ep{ep}-{slug}/
└── content.md  (with proper frontmatter structure)
\`\`\`

### Step 3: Prompt for Source Materials

After folder creation, prompt user:

\`\`\`
✅ Created: assets/youtube/ep{ep}-{slug}/content.md

📥 Next: Add source materials to the folder:
   1. script.md - 整理版口白 (from Google Docs)
   2. subtitle.srt - 原始字幕 (optional)

Ready to generate AI content? [Y/n]
\`\`\`

### Step 4: Generate AI Content (if --generate or user confirms)

If `--generate` flag is set OR user confirms:

1. Check if `script.md` exists in the folder
2. If exists, extract summary to `## 🧠 原始口白摘要` section
3. Run `/yt-generate ep{ep}-{slug}` to generate:
   - `### 📝 YouTube Description`
   - `### 🕒 Timestamp`
   - `### 🏷️ SEO 優化標題建議`
   - `## 🔗 關聯知識`
   - `readme.md`

### Step 4.5: Auto-populate related_assets (Knowledge Links)

After content generation, automatically add `related_assets` to the frontmatter:

1. Extract keywords from `title` and `tags`
2. Search evergreen notes — find related concepts in `assets/evergreen/`
3. Search other YouTube episodes — find topically similar episodes
4. Add to frontmatter (max 6 links, wikilink format)

## Related Commands

- `/yt-generate` - Generate AI content for existing episode
- `/yt-expand` - Expand to other platforms (FB, Threads, Skool)
- `/yt-circulation` - Full content circulation workflow
```

### 這個設計裡有幾個值得注意的地方

**`$ARGUMENTS` 佔位符。** 這是 Commands 最核心的機制。你輸入 `/yt-init 50 mcp-server "MCP Server 完整教學"`，`$ARGUMENTS` 就會被替換成 `50 mcp-server "MCP Server 完整教學"`，然後 Claude 從這個字串裡 parse 出 ep、slug、title。這讓一個指令可以處理任意輸入。

**分步驟的 workflow。** `/yt-init` 不是一段簡短的 prompt，而是一個有明確步驟的工作流程（Step 1 到 Step 4.5）。Claude 會照著走，不會漏步驟，也不會自由發揮。這解決了「每次輸出不一樣」的問題。

**`[1/4]` 是系列設計。** description 前面的 `[1/4]` 表示這是 YouTube 工作流的第一步，後面還有 `/yt-generate`、`/yt-expand`、`/yt-circulation`。Commands 可以互相呼叫，形成一條完整的流水線。

**Related Commands 是導覽。** 每個指令末尾列出相關指令，讓你知道下一步要用什麼。這讓 69 個指令不會變成一座迷宮。

### 命名空間系統

我的 content-asset-system 有 69 個指令，用命名空間組織：

```
.claude/commands/
├── gaia/
│   ├── content/          # YouTube + 課程 + 研究 (24 個指令)
│   │   ├── yt-init.md    → /yt-init
│   │   ├── yt-generate.md → /yt-generate
│   │   ├── yt-expand.md  → /yt-expand
│   │   ├── course-quick.md → /course-quick
│   │   └── research-init.md → /research-init
│   ├── flywheel/         # 社群發佈 + 分析 (21 個指令)
│   │   ├── pulse-post.md → /pulse-post
│   │   ├── digest-generate.md → /digest-generate
│   │   └── revenue-dashboard.md → /revenue-dashboard
│   └── core/             # RAG 知識庫 + 常青內容 (18 個指令)
│       ├── brain-search.md → /brain-search
│       ├── brain-transform.md → /brain-transform
│       └── evergreen-seed.md → /evergreen-seed
└── bizops/               # 顧問客戶管理 (7 個指令)
    ├── consulting-init.md → /consulting-init
    └── consulting-kpi.md → /consulting-kpi
```

命名空間的價值不只是整理。`/yt-*` 這個 prefix 一看就知道是 YouTube 相關，`/brain-*` 就知道是知識庫操作。你不需要記住 69 個指令，只需要記住 prefix 的語意。

### 怎麼設計你的第一個 Command

**第一步：找重複。** 這週你在 Claude 對話框裡貼了超過三次的 prompt，那就是一個 Command 的候選。「幫我 review 這個 PR，注意安全漏洞和命名規範」、「把這段程式碼的邏輯解釋給初學者聽」、「幫我寫這個功能的測試計畫」——都是好的起點。

**第二步：寫 `.md` 檔。** 在 `.claude/commands/` 下面建一個 Markdown 檔案。Frontmatter 放 description，內文就是你的 prompt。用 `$ARGUMENTS` 標記需要替換的部分。

```markdown
---
description: "Code review：安全 + 命名 + 品質"
---

請用繁體中文 review 以下程式碼或檔案：

**目標：** $ARGUMENTS

Review 的重點：
1. 安全漏洞（OWASP Top 10）
2. 命名規範是否符合 CLAUDE.md
3. 有沒有多餘的 console.log 或 debug code
4. 具體的改進建議（每個問題都附範例）

回傳格式：
- 嚴重問題（必須改）
- 建議改進（可以考慮）
- 做得好的地方
```

**第三步：測試並迭代。** 跑幾次，看輸出哪裡不符合預期，修改 prompt。Commands 是活的文件，可以隨時調整。

這些 Command 檔案可以 commit 到 git，整個團隊共享同一套工作流標準。這是 Commands 最有價值的地方——不是讓你少打字，是讓團隊的最佳實踐變成可執行的指令。

---

## 8.3 Agents — 專業角色定義（完整拆解）

Commands 解決「重複動作」的問題。但有另一種問題更難處理：**你希望 Claude 在某類任務上有穩定的角色和行為邊界**。

你不希望幫你 review code 的 Claude 突然開始自動改你的程式。你不希望幫你寫文章的 Claude 突然開始建議你改架構。

**Custom Agents** 讓你定義有特定角色和職責的 AI。

### 一個真實的 Agent 檔案長什麼樣？

這是我的 Content Writer Agent，負責所有平台的內容生成。完整的 `.claude/agents/gaia/content-writer.md`：

```markdown
---
name: content-writer
description: Generate platform-specific content following style guidelines. Applies style-dna.yaml brand voice, uses platform templates, and adapts content for YouTube, Facebook, Threads, Skool, and course formats. Use for any content generation task.
tools: Read, Edit, Write, Grep, Glob
model: sonnet
---

You are the Content Writer, generating platform-specific content.

## Alex Voice DNA (Mandatory)

All user-facing content MUST follow Alex's voice. Load these two files before writing:

- **Voice structure**: `assets/blog/blog-post-dna.yaml` — 9-section structure, tone, signature phrases
- **Style rules**: `.claude/skills/gaia/resources/style-dna.yaml` — Brand voice, restrictions, CTA rules

**Core voice rules:**
- First person (我), conversational yet authoritative
- Honest about limitations — never oversell
- Forbidden words: 絕對、一定、最好的、唯一
- Permission-based CTAs ("如果你覺得可以，試試看")
- Short paragraphs (≤ 3 lines), real-world analogies

## When Invoked

1. Load Alex voice DNA — Read `assets/blog/blog-post-dna.yaml`
2. Load style guide — Read `.claude/skills/gaia/resources/style-dna.yaml`
3. Load platform rules — Check platform-specific constraints
4. Load template — Read the appropriate template
5. Search related content — Find previous content for consistency
6. Generate content — Write following Alex's voice and brand voice
7. Self-check quality — Verify against the checklist below

## Platform Constraints

| Platform | Max Length | Style |
|----------|------------|-------|
| YouTube | No limit | Professional, comprehensive |
| Facebook | 300 words | Moderate, soft CTA |
| Threads | 280 chars/post | Hook-first |
| Skool | No limit | Engagement-focused |
| Course | No limit | Educational, structured |

## Quality Checklist

Before returning any content, verify:
1. Alex voice DNA applied — first person, honest, no forbidden words
2. Hook matches persona style
3. CTA is permission-based (not pushy)
4. Length is within platform constraints
5. All content is in Traditional Chinese
```

### 這個設計裡有幾個關鍵決策

**`tools: Read, Edit, Write, Grep, Glob`。** Agent 的 `tools` 欄位限制了它能使用的工具。Content Writer 只能讀寫檔案，不能跑 Bash 指令、不能連網路。這不是偶然的——寫內容不需要執行程式碼，給它過多工具反而增加風險。

**`model: sonnet`。** 不同的 Agent 可以指定不同的模型。需要大量推理的 Orchestrator 用 opus，日常內容生成用 sonnet，簡單的格式處理可以用 haiku。這讓你在成本和效果之間做出選擇。

**「When Invoked」是工作流，不是角色描述。** 很多人寫 Agent 只寫「你是一個 Content Writer」。這樣太模糊。好的 Agent 定義了收到任務後的具體步驟：先載入哪個檔案、再做什麼、最後驗證什麼。Claude 會照著走。

**「Quality Checklist」是自我稽核。** 在 Agent 末尾加一個 checklist，讓 Claude 在回傳結果之前自己核對。這比在每次對話裡提醒「記得檢查禁用詞」有效得多。

### 我目前的 Agent 架構

```
.claude/agents/
├── gaia/                     # Gaia 系統（內容飛輪）
│   ├── brain-orchestrator.md # 協調整個 agent team，分派任務
│   ├── content-analyst.md    # 研究分析來源內容
│   ├── content-writer.md     # 生成各平台內容
│   ├── course-designer.md    # 設計課程結構（KUDM 框架）
│   ├── social-publisher.md   # 跨平台發佈與分析
│   └── linear-manager.md     # 建立/管理 Linear issue
├── code-reviewer.md          # 只 review，不改，給具體建議
├── debugger.md               # 根因分析（不是「幫你改掉」）
├── doc-writer.md             # 文件生成
├── project-contract-manager.md # 外包合約與專案管理
└── test-runner.md            # 跑測試、解讀錯誤、回報結果
```

### 什麼時候用 Agent，什麼時候用 Command？

這是一個很實際的問題，判斷框架如下：

| | Command | Agent |
|--|---------|-------|
| 觸發方式 | `/指令名稱`，一次性觸發 | 被其他 Agent 或 Command 呼叫 |
| 適合場景 | 單一動作，快捷鍵 | 需要持續角色行為、有職責邊界 |
| 工具限制 | 無（繼承對話的工具） | 可以在 frontmatter 明確限制 |
| 協作方式 | 獨立執行 | 可以跟其他 Agent 搭配 |
| 何時建立 | 你重複輸入同樣的 prompt | 你需要角色一致性、或多角色協作 |

簡單來說：**Command 是快捷鍵，Agent 是有職責邊界的角色。**

如果你只是想把一段 prompt 存下來重複使用，用 Command。如果你想讓某類任務有穩定的行為模式，不受對話上下文影響，用 Agent。

---

## 8.4 Skills — 自動載入的知識

如果說 Commands 是「快捷鍵」，Agents 是「角色」，那 Skills 是「背景知識」。

**Skills** 是自動載入的知識模組。你不需要每次在對話裡解釋「我們的 blog 系統怎麼運作」、「n8n workflow 的規範是什麼」——Skill 幫你記住了。

### 一個 Skill 檔案長什麼樣？

這是 `content-pipeline` skill 的開頭部分（`.skills/content/content-pipeline.md`）：

```markdown
---
name: content-pipeline
domain: content
description: "Content generation patterns for Alex's flywheel — blog, YouTube, SEO/GEO"
version: "1.0.0"
last_updated: "2026-02-21"
sessions_used: 0
learned_from:
  - "v1.0.0: seed from CLAUDE.md + blog-post-dna.yaml + MEMORY.md"
cross_refs:
  - "assets/blog/blog-post-dna.yaml"
  - ".claude/skills/gaia/resources/style-dna.yaml"
---

# Content Pipeline Skill

## Blog Post DNA (9-Section Structure)

Every blog post follows this exact structure:

1. **professional_intro** — hook + bullet preview (NOT generic greeting)
2. **definition** — answer capsule first (30-50 bold words), then expand
3. **alex_observation** — strategic analysis + honest limitations
4. **step_by_step** — workflow-first with screenshots/code
5. **comparison_table** — 3+ dimensions minimum
6. **key_takeaways** — numbered list
7. **faq** — 4+ questions, progressive complexity
8. **next_steps** — CTA with Skool referral link
9. **related_resources** — YouTube embed + Skool + GitHub links

## Voice Rules

- Identity: "pragmatic architect explaining to a smart friend"
- Perspective: first person ("我" / "我們")
- Tone: conversational + authoritative
- Honest opinions are encouraged: "聊勝於無", "確實不太行"
```

### Skills 跟 Commands 有什麼不同？

| | Command | Skill |
|--|---------|-------|
| 觸發方式 | `/指令名稱`，手動呼叫 | 自動載入，或自然語言觸發 |
| 適合 | 單一動作，快捷鍵 | 領域知識，持續性背景 |
| 內容 | 一段工作流程 prompt | 知識模組 + 跨引用的資源檔案 |
| 生命週期 | 一次對話 | 跨對話，長期積累 |

具體來說：當你在對話裡說「幫我寫一篇 blog」，Skill 已經讓 Claude 知道我的 blog 有 9 個固定區段、禁用詞是什麼、SEO 規則怎麼套。你不需要在每次對話裡重新解釋這些。

### 我的四個 Skills

- **content-pipeline** — blog DNA、YouTube 工作流、SEO 規則、發佈流程
- **python-service** — uv 套件管理、Typer CLI 寫法、pre-commit 設定、ruff 規範
- **linear-workflow** — branch 命名規則、commit message 格式、epic 同步、Linear CLI 用法
- **n8n-workflow** — n8n 架構邊界、data flow 規範、webhook patterns、節點命名規則

Skill 不是一次寫完的。它從一個簡單的筆記開始，隨著你踩到坑、發現模式、累積最佳實踐，慢慢長大。

---

## 8.5 深入案例：Gaia 內容系統

這一節把前四節說的東西放在一個真實系統裡看。

Gaia 是我的內容管理系統，處理從 YouTube 影片到部落格文章、Skool 貼文、Email 序列的完整內容飛輪。它不是一個 app，它是建在 Git repo 上的一套 Claude Code 工作流。

### 系統架構一覽

```
content-asset-system/
├── assets/                    # 所有內容（SSoT）
│   ├── youtube/              # YouTube 集數：ep{XX}-{slug}/content.md
│   ├── blog/                 # 部落格文章
│   ├── course-*/             # 課程素材
│   ├── evergreen/            # 永久知識筆記
│   └── pulse/                # 社群媒體測試貼文
├── .claude/
│   ├── agents/gaia/          # 6 個 AI 角色
│   ├── commands/             # 69 個 slash commands
│   └── skills/gaia/          # Gaia skill（共用資源）
├── .skills/                  # 4 個知識 Skill
└── src/gaia/                 # Python CLI (gaia 指令)
```

### 一個真實的工作流：從 YouTube 影片到多平台內容

假設我今天拍完一集 YouTube 影片，EP50，主題是 MCP Server。流程是這樣的：

**第一步：初始化影片資料夾**

```
/yt-init 50 mcp-server "MCP Server 完整教學：設定、整合、避坑"
```

Claude 讀取 `/yt-init` 指令，執行以下動作：
- 跑 `gaia youtube init 50 mcp-server "MCP Server 完整教學"` 建立資料夾
- 建立 `assets/youtube/ep50-mcp-server/content.md`，填入正確的 frontmatter 結構
- 問我：有沒有 `script.md`（整理版口白）要放進去？

**第二步：放入素材，生成 AI 內容**

把 Google Docs 的口白整理成 `script.md` 放進資料夾，然後：

```
/yt-generate ep50-mcp-server
```

Content Analyst Agent 讀取 `script.md`，提取重點。Content Writer Agent 接著生成：
- YouTube 影片描述（符合 SEO 規範）
- 章節時間戳
- 5 個 SEO 標題建議
- 相關影片連結（從 evergreen 知識庫自動找）

**第三步：擴散到其他平台**

```
/yt-expand ep50-mcp-server --platforms fb,threads,skool
```

Content Writer Agent 根據 `style-dna.yaml` 裡的平台限制，把同一個內容改寫成：
- Facebook 貼文（≤ 300 字，soft CTA）
- Threads 串文（每則 ≤ 280 字，hook-first）
- Skool 貼文（engagement-focused，附討論問題）

**這整個流程大概 20 分鐘**，大部分時間是我在確認 AI 生成的內容品質，不是在打字。

### 從指令到 Agent 的協作

`/yt-generate` 這個指令背後不是單純的 prompt，而是多個 Agent 的協作：

```
/yt-generate 觸發
    ↓
Brain Orchestrator 判斷任務類型
    ↓
Content Analyst 讀取 script.md，提取重點摘要
    ↓
Content Writer 接收摘要，生成各個區段
    ↓
結果寫入 content.md
```

Brain Orchestrator 的角色是協調，它決定要呼叫哪個 Agent，傳遞什麼資料。Content Analyst 和 Content Writer 各自專注在自己的職責上，不會互相干擾。

這就是為什麼要把工作拆成多個 Agent，而不是一個什麼都做的大 Agent：**分工讓每個角色的行為更可預測、更容易調整。**

### Skills 在這個系統裡的角色

每次 Content Writer Agent 被呼叫，它第一件事是讀 `assets/blog/blog-post-dna.yaml` 和 `style-dna.yaml`。這兩個檔案就是 content-pipeline Skill 的核心資源。

但 Skill 不只是存檔案位置。它的 frontmatter 記錄了：這個 Skill 最後更新是什麼時候（`last_updated`）、從哪些 session 裡學到的（`learned_from`）、使用了幾次（`sessions_used`）。這讓我知道哪個 Skill 已經過時、需要更新。

### 「從 0 到 69 個指令」的演化故事

我不是一開始就有 69 個指令的。

最早的系統只有一個目的：把 YouTube 影片的口白存進 Git，讓 Claude 幫我寫影片描述。那時候大概 5 個 Command，結構很簡單。

然後我開始把部落格也放進來。原本用 Airtable 管的內容，慢慢都移到 Git。每次移一個新的工作流，就加幾個 Command。

然後客戶開始找我做顧問。顧問的文件流程（合約、健檢表、月報）也搬進來，加了 bizops 命名空間。

然後課程上線，課程結構管理也加進來。

每一個新的命名空間都是因為有一個真實的工作需求，不是為了讓數字好看。

**重點：69 個指令不是一次設計出來的，是問題驅動的積累。** 每個 Command 的背後都有一個「這件事我做超過三次了」的時刻。

---

## 8.6 從工具到系統的演化路徑

我不是一開始就有這些的。

**第一個月**，只有 CLAUDE.md。把專案的規則、命名慣例、禁用語法寫進去。這個階段的 Claude 是「有記憶的助手」。光是這步，就讓每次對話的品質提高很多。

**第三個月**，加了 10 個 Commands。把最常重複的工作流封裝成指令：code review、blog 生成、YouTube 描述生成。這個階段的 Claude 是「有工具的助手」。

**第六個月**，開始加 Agent。不是因為 Commands 不夠用，而是因為我想讓「寫內容」和「review 程式碼」這兩件事有明確的角色邊界。Content Writer 不應該突然去改我的 Python，Code Reviewer 也不應該插手寫文案。

**第九個月**，加了 Skills。當我發現自己在不同的 Command 和 Agent 裡重複解釋同樣的背景知識，就把那些知識抽出來放進 Skill。

**現在**：69 個 Commands，4 個 Skills，10 幾個 Agents，跨三個 repo。這個階段的 Claude 更像是「一個會自己運作的工作流系統」——我說要做什麼，它知道怎麼做，不需要我每次解釋背景。

### 你今天可以做的一步

找到你這週重複輸入超過三次的 prompt。

不需要是完美的 prompt。不需要想好整套系統架構。就是那個讓你每次都要複製貼上、每次打法還不太一樣、Claude 給的結果也不穩定的 prompt。

把它存成 `.claude/commands/review.md`，下次用 `/review` 觸發。

這是第一步，也是最重要的一步。系統是從這裡開始長出來的，不是從架構圖開始的。

從 CLAUDE.md 到 Commands，從 Commands 到 Agents，從 Agents 到 Skills——每一步都有它的時機，不需要跳步，也不需要一次全做。

---

如果你想跟其他也在用 Claude Code 建工作流的人交流，歡迎加入 Skool 社群。裡面有我分享的 Command 模板、Gaia 系統的設計過程，以及其他人實際在用的案例：

→ https://www.skool.com/ai-brain-alex/about?ref=5dde9b20e8e7432aa9a01df6e89685f4
