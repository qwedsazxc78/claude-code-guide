# 第五章：Config — 把 Claude 配置成你的專屬工程師

前四章你學會了怎麼跟 Claude 說話、怎麼讓它動手改 code。

但有一個問題我每次 onboard 新用戶都會遇到：他們花了大量時間在每次對話開頭重複解釋同樣的事情。「我們用 TypeScript strict mode」、「price 欄位用分不用元」、「不要動 lib/legacy/」。

Claude 不記得你上一次告訴它什麼。每次對話，它都是乾淨的板子。

Config 就是解決這個問題的。CLAUDE.md、settings.json、Hooks — 這三個檔案合在一起，讓你從「每次對話都要重新解釋」變成「Claude 一開口就知道我的規矩」。

這也是我認為整個 Claude Code 生態系裡最值得深入研究的部分。

---

## 5.1 CLAUDE.md — AI Coding 的靈魂

### CLAUDE.md = Infrastructure as Code for AI

我的背景是 SRE / DevOps，不是傳統軟體開發。

在 SRE 的世界，有一個概念叫 Infrastructure as Code：你不用手動設定伺服器，而是把環境的期望狀態寫成程式碼，讓工具去執行。優點是可複現、可審計、可版本控制。

CLAUDE.md 對 AI 來說就是這個概念。

你把期望 Claude 的行為方式、你的專案規範、你的偏好——全部寫進這個檔案。Claude Code 每次啟動都會自動讀取它。不用解釋，不用提醒，它就直接知道。

這是我用過所有 AI coding 工具裡面，最接近「一次設定，長期受益」的機制。

### 三層架構

```
~/.claude/CLAUDE.md              ← 全域（所有專案的共同偏好）
your-project/CLAUDE.md           ← 專案層級（最常用的那層）
your-project/src/module/CLAUDE.md ← 子目錄（特定模組的規範）
```

越內層的設定越優先。全域放你的語言偏好和個人習慣；專案層放這個 repo 的規矩；子目錄放特定模組才需要知道的規則。

實際上大多數人 80% 的時間只需要管專案層。子目錄只有在你的 repo 裡有明顯不同風格的模組才需要。

### 我的 CLAUDE.md 演化史

說實話，我的第一版 CLAUDE.md 只有 50 行。

6 個月後，它長到 28KB、718 行。

不是因為我一開始就知道要寫什麼。是因為每次 Claude 犯錯，我就加一條規則。

「Claude 把 ep 欄位輸出成字串 `"49"`，而不是整數 `49`。」→ 加一條規則：

```yaml
# Frontmatter Field Types
ep: 49    # Integer (NOT string "49")
```

「Claude 建了一個命名是 `ep49_n8n_2.0` 的資料夾，用了底線。」→ 加一條：

```
# Naming Conventions
Episode folders: ep{XX}-{slug}   ← lowercase, hyphen-separated
Never use: ep49_n8n_2.0          ← 底線不行
```

「Claude 把 status 值寫成 `'in_progress'`，但我的系統只接受 6 個特定值。」→ 加一條精確的列舉：

```yaml
# Status Values (Exact Strings Only)
status: "draft"
status: "ready_to_generate"
status: "generated"
status: "ready_to_publish"
status: "published"
status: "archived"
```

你有沒有遇過這種情況？Claude 聰明歸聰明，但它猜不到你系統裡的隱性規範。這些規範只存在你腦子裡，或者散落在程式碼的角落。CLAUDE.md 就是把這些隱性知識顯性化的地方。

718 行聽起來很多。其實裡面有相當大一部分是表格和程式碼範例，人工閱讀大概 20 分鐘能讀完。

### 我的 CLAUDE.md 裡有什麼

除了命名規範和 status 值，我還在裡面放了一些比較進階的東西，分享一下思路：

**Semantic Detection（察言觀色）**

我定義了 10 種用戶訊號，讓 Claude 根據語氣調整行為（以下列出幾個代表性的）：

| 訊號 | 觸發詞 | Claude 的反應 |
|------|--------|--------------|
| 挫折 | 「又壞了」、「怎麼又...」 | 放慢速度，解釋根本原因 |
| 不確定 | 「不確定」、「可能要...」 | 提供 2-3 個具體選項 |
| 緊迫 | 「快一點」、「直接做」 | 減少解釋，直接行動 |
| 混亂 | 「等等」、「看不懂」 | 換個方式重新解釋 |

這讓 Claude 不只是執行指令的工具，而是能夠讀懂場景的工作夥伴。

**Self-Learning [LEARN] flags**

當 Claude 在工作中發現可重用的模式，我要求它用 `[LEARN]` 標記：

```
[LEARN] Blog posts with FAQ schema get 3x more AI citations — add to content-pipeline skill
```

這樣下次 session 結束我用 `/memory-learn` 把標記提取出來，好的洞察就不會消失在對話歷史裡。

**Health Monitoring**

我讓 Claude 定期檢查專案的健康信號，比如「skill 檔案超過 30 天沒更新要提醒我」、「TASK.md 超過 7 天沒動要問我是不是還在做這件事」。

這些都不是一開始就有的。是用多了，發現問題，才慢慢加進去的。

**Alex Voice DNA — 禁用詞清單**

我在 CLAUDE.md 裡放了一份寫作語氣規範，包含禁止用的詞彙：絕對、一定、最好的、唯一。

這樣 Claude 幫我產生任何用戶面向的內容，都會自動遵守我的品牌語氣，不用每次提醒。

### 該寫什麼、不該寫什麼

有一個反直覺的教訓我想特別說：CLAUDE.md 越長不代表越好。

我曾經試過把一份 2000 行的 API 文件整個貼進 CLAUDE.md。結果 Claude 反而變笨了。

原因很簡單：CLAUDE.md 每次對話都會佔用 context window。你塞一份 2000 行的 API 文件，就等於每次對話都少了那麼多 context 可以用。而且那份文件裡 90% 的內容當下對話根本用不到，但它就靜靜地佔著空間。

重點來了：**CLAUDE.md 是規範，不是文件庫。**

應該寫進去的：
- 命名規範（精確到有反例）
- 禁止碰的目錄或模式
- 特殊商業邏輯（price 用分不用元這種）
- 跑測試、build 的指令
- 狀態值的精確枚舉

不應該寫進去的：
- 整份 API 文件
- 所有 DB schema
- 每個 function 的說明（Claude 自己會讀 code）
- 任何機密資訊（CLAUDE.md 會進 git）

一個好的判斷標準：「這是新工程師第一天我會主動告訴他的事嗎？」如果是，寫。如果是「他自己看 code 就會知道的事」，不用寫。

### 子目錄 CLAUDE.md：因地制宜

我有兩個子目錄 CLAUDE.md 的例子，思路截然不同。

`assets/blog/CLAUDE.md` 裡面放的是 blog 文章的 SEO 規則、語氣規範、兩個 repo 的同步流程。因為 blog 這個子系統有自己獨特的工作流，如果塞進主 CLAUDE.md 會讓它更難讀。

外包客戶的專案資料夾（像 `assets/outsourcing-project/鼎盛資科-clarins-precious/`）也有自己的 `CLAUDE.md`，裡面放的是這個客戶特有的文件規範：「客戶文件禁用 code blocks」、「語氣必須正式」、「所有文件加版本紀錄」。

子目錄 CLAUDE.md 的精神是：不同語境下有不同規矩，不要讓所有規矩都擠在根目錄的那份檔案裡。

---

## 5.2 settings.json — 權限與行為控制

如果說 CLAUDE.md 是告訴 Claude「要做什麼」，settings.json 就是告訴 Claude「能做什麼」。

這個區別很重要。你不會給新同事 root 權限對吧？同理，Claude 需要的是剛好夠用的權限，不多也不少。

### 三層設定架構

```
~/.claude/settings.json          ← 全域個人偏好（不進 git）
.claude/settings.json            ← 專案共享設定（進 git）
.claude/settings.local.json      ← 個人在此專案的設定（.gitignore）
```

優先順序：`settings.local.json` > `settings.json` > `~/.claude/settings.json`

最重要的那一層是 `.claude/settings.json`，因為它進 git，團隊所有人都受它約束。

### allow / deny 實用 patterns

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm run test)",
      "Bash(npm run lint)",
      "Bash(npm run build)",
      "Bash(git status)",
      "Bash(git diff)",
      "Bash(git log *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(DROP TABLE*)"
    ]
  }
}
```

`allow` 是白名單，列在裡面的操作 Claude 不用每次確認直接執行。`deny` 是黑名單，一律擋住。兩邊都沒列的，每次都會跳出來問你。

寫 allow 有一個原則：精確比廣泛好。

`"Bash(*)"` 全部放行等於給 Claude root 權限。`"Bash(npm run *)"` 只開放 npm 指令，安全很多。`"Bash(npm run test)"` 只開放跑測試，更精確。

從最嚴格開始，用起來覺得太麻煩了再放寬。反過來——從寬鬆開始、出事了再收緊——代價高很多。

### 團隊共享策略

`.claude/settings.json` 進 git 的最大好處是：新成員 clone 就自動生效，不用另外設定。

我建議至少把這幾條 deny 加進去，無論什麼專案：

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)"
    ]
  }
}
```

這三條是我見過最多人因為 Claude「幫了一個忙」而後悔的操作。

---

## 5.3 Hooks — 自動化觸發器

Hooks 是 Claude Code 裡面最被低估的功能。

它的概念跟 Git Hooks 一樣——在特定事件發生時，自動執行一段指令。但 Claude Code 的 Hooks 是在 AI 的操作前後觸發，不是在 git commit 前後。

### 四種事件

| 事件 | 觸發時機 | 最常用來做什麼 |
|------|---------|--------------|
| `SessionStart` | 對話開始時 | 載入 context、檢查環境 |
| `PreToolUse` | Claude 執行工具前 | 擋住危險操作 |
| `PostToolUse` | Claude 執行工具後 | 自動格式化、lint |
| `PostToolUseFailure` | 工具執行失敗後 | 錯誤通知 |

### 我真正在用的 Hooks

**自動 Prettier（每個前端專案都應該設定）**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "npx prettier --write $FILE_PATH"
      }
    ]
  }
}
```

Claude 改完任何檔案，自動跑 Prettier 格式化。

這個 hook 的投資報酬率極高：設定一次，AI 產出的格式永遠對。你再也不會遇到「Claude 改的 code 格式跟其他人不一樣」的 PR review 問題。

**擋住 rm -rf**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo \"$COMMAND\" | grep -qE 'rm\\s+-rf' && exit 1 || exit 0"
      }
    ]
  }
}
```

只要 Claude 想跑任何包含 `rm -rf` 的指令，直接擋住。exit code `1` = 阻止，exit code `0` = 繼續。

這個設計很優雅：你不用在 settings.json 裡列舉每一種危險指令的變體，Hook 可以用 grep 做複雜的邏輯判斷。

**我系統裡的 linear-auto-track**

這是我的 CLAUDE.md 裡面的真實案例：

```bash
#!/bin/bash
# .claude/hooks/linear-auto-track.sh
# 當工作在 GAI-42 這類分支上時，自動把 Linear issue 設為 In Progress
# 每個 session 只觸發一次（透過 /tmp 的 lock file 去重）
```

這個 hook 在 PostToolUse 時觸發，讀取當前 git branch 名稱。如果 branch 名稱符合 `GAI-{number}-*` 或 `CLO-{number}-*` 的格式，就自動呼叫 Linear API 把對應 issue 設為 In Progress。

我在 CLAUDE.md 裡可以這樣寫：「不用手動跑 `gaia linear update`，hook 會自動處理。」然後 Claude 就真的不會再多跑那個指令。

### exit code 的語義

Hooks 的行為取決於腳本的 exit code：

- `exit 0` = 繼續執行
- `exit 1` = 阻止這次操作

這個設計很乾淨。你的 hook 可以做任何複雜的邏輯，最後只需要決定「繼續還是擋住」。

一個實用的注意事項：在 PostToolUse hook 的指令加 `2>/dev/null || true`，這樣就算 Prettier 或 ESLint 出錯，Claude 的主要工作流不會被中斷。Hook 的失敗不應該影響 AI 的主要任務。

---

## 5.4 實戰：我的 CLAUDE.md 設計哲學

整理一下這幾年摸索出來的原則。

### 「每犯一次錯，加一條規則」

這是我 CLAUDE.md 從 50 行長到 718 行的方式。

不要試圖在一開始就想清楚所有規則。你不知道 Claude 會在哪裡犯你沒預料到的錯。讓它工作，當它犯錯，就把那個規則寫進去，帶著反例。

帶反例很重要。只寫「用整數」沒有「用整數（NOT 字串 `"49"`）」有效。因為如果你知道反例，代表你真的遇過這個問題，那個規則就會寫得很精準。

### 寫什麼 vs 不寫什麼

我有一個更粗糙但實用的判斷標準：

**寫進去：** 如果 Claude 沒看到這個資訊，它有超過 20% 機率做錯的事。

**不寫進去：** 如果 Claude 讀完 code 自己就能推斷出來的事。

Claude 很擅長從 code context 裡推斷慣例。你不需要解釋「我們的 TypeScript 是 strict mode」，因為它讀到 `tsconfig.json` 就知道了。但是「price 欄位用分不用元」這種商業邏輯，code 裡看不出來，你必須明確寫。

### 2000 行 API 文件反而讓 Claude 變笨

前面提過這個故事，但值得再展開一下。

問題不只是 context 被佔滿。更細微的問題是：當 Claude 讀到一份很長的文件，它會試圖把所有資訊都考慮進去，反而分散了對真正重要事情的注意力。

CLAUDE.md 的長度和 Claude 的品質之間，不是線性正相關。有一個甜蜜點，過了那個點，多加的內容反而有害。

我的體感是：純文字 500 行左右是上限。超過這個，就要想想能不能拆到子目錄的 CLAUDE.md，或者改用 Skill 文件（按需載入）。

### 子目錄 CLAUDE.md 的邊界思考

什麼時候值得為子目錄建一個獨立的 CLAUDE.md？

當那個子目錄的規則跟主專案的規則有明顯差異，或者規則的受眾不同。

我的 `assets/blog/CLAUDE.md` 是因為 blog 的工作流涉及兩個 repo 的同步，還有一套 SEO 規則，這些規則只有在寫 blog 時才需要知道。每次對話都載入這些，很浪費。

外包客戶的資料夾有獨立的 CLAUDE.md，是因為對那個客戶的文件風格要求，完全不適用於我自己的內部文件。如果全部混在一起，Claude 可能會把客戶文件寫得太技術性，或者把內部文件寫得太像業務簡報。

邊界清楚的 CLAUDE.md 架構，讓你可以在不同語境裡切換，不用每次手動提醒 Claude「這次的受眾不同」。

---

## 5.5 團隊 CLAUDE.md — 多人協作怎麼辦

一個人用 CLAUDE.md 很簡單。但當你的團隊有 5-8 個人，問題就來了。

### 誰 Own CLAUDE.md？

我的建議：CLAUDE.md 跟 code 一樣走 PR review。

任何人都可以提 PR 修改 CLAUDE.md，但 merge 前要有至少一個人 review。原因很簡單：CLAUDE.md 裡的一條規則會影響所有人跟 Claude 的互動。你加了一條「所有 API 回傳都用 camelCase」，但另一個同事的模組全是 snake_case — 這就是衝突。

PR review 的好處不只是品質把關，更是強迫提 PR 的人把「為什麼加這條規則」寫清楚。三個月後看 git blame，你會謝謝自己。

### 團隊 CLAUDE.md 模板

如果你不知道從哪裡開始，這是一個通用的 Team CLAUDE.md 骨架：

```markdown
# Project Overview
[用 2-3 句話描述這個 repo 是做什麼的、主要技術棧]

## Tech Stack
- Language: TypeScript 5.x (strict mode)
- Framework: Next.js 14 (App Router)
- DB: PostgreSQL + Prisma

## Naming Conventions
- 檔案：kebab-case（`user-profile.ts`，NOT `userProfile.ts`）
- 元件：PascalCase（`UserCard.tsx`）
- 資料庫欄位：snake_case（`created_at`，NOT `createdAt`）
- 反例：price 欄位用分（integer），NOT 元（decimal）

## Git Conventions
- Branch：`{type}/{ticket-id}-{short-description}`（e.g. `feat/APP-42-user-auth`）
- Commit：Conventional Commits（`feat:`, `fix:`, `chore:`）
- 不要直接 push to main

## Testing Rules
- 每個新功能至少一個 integration test
- Unit test 覆蓋核心 business logic
- 跑測試：`npm run test`

## Forbidden Patterns
- 不要動 `lib/legacy/`（舊系統，等待廢棄）
- 不要在 API handler 裡放 business logic（放在 service layer）
- 不要 commit `.env`、`*.pem`、`credentials.json`

## Team-Specific Rules
[這裡放只有你們團隊才知道的隱性規矩]
```

這個骨架的設計邏輯：每個 section 都針對「Claude 最容易猜錯的地方」。Tech Stack 告訴它版本；Naming Conventions 帶反例；Forbidden Patterns 直接告訴它哪裡不能碰。

### 解決衝突的原則

當團隊成員有不同偏好（camelCase vs snake_case、tabs vs spaces）：

1. 寫進 CLAUDE.md 的就是團隊標準，個人偏好放 Global `~/.claude/CLAUDE.md`
2. 有爭議？用 linter 設定作為裁判（ESLint、ruff 說了算）
3. CLAUDE.md 不是個人風格宣言，是團隊工程標準

重點來了：如果你們連 CLAUDE.md 該怎麼寫都有爭議，代表你們的工程規範本來就沒對齊。CLAUDE.md 只是讓這個問題浮出水面，不是製造問題。

---

## 5.6 安全基礎 — Claude 看到什麼？

這是企業導入時 CTO 會問的第一個問題。

### Claude Code 的資料流

當你用 Claude Code：

1. 你的 prompt + 被引用的檔案內容 → 送到 Anthropic API
2. Claude 的回覆 → 回到你的本地環境
3. Claude 執行的指令（`npm test`、`git commit`）→ 在你的機器上執行

重點：你的程式碼會傳到 Anthropic 的伺服器。這不是秘密，這是 LLM 的基本運作方式。問題是你要不要接受這個前提，以及怎麼在這個前提下工作得安全。

### Anthropic 的資料政策

- **Pro/Max 訂閱**：Anthropic 聲明不會用你的 input/output 訓練模型
- **API 使用**：同樣不訓練，但要注意 log retention 政策
- 詳細條款請查閱 Anthropic 的 Usage Policy 和 Privacy Policy（政策會更新，以官方最新版為準）

### 實務建議

**1. API Key 管理**

不要用 `export ANTHROPIC_API_KEY=sk-...` 裸放在 shell profile。用 `.env` + `.gitignore`，或更好的：用 1Password CLI、系統 keychain，或 AWS Secrets Manager 這類工具管理。

**2. settings.json deny list**

把敏感目錄加入 deny：

```json
{
  "permissions": {
    "deny": [
      "Read(.env*)",
      "Read(credentials/*)",
      "Read(secrets/*)",
      "Read(**/*.pem)"
    ]
  }
}
```

這樣 Claude 就算被要求讀取這些檔案，也會直接拒絕。

**3. CLAUDE.md 不放密碼**

CLAUDE.md 會 commit 到 git。任何 secret、API key、DB 連線字串都不該出現在裡面。只放規範，不放憑證。

**4. .mcp.json 加入 .gitignore**

MCP 設定檔可能包含 DB 連線字串或 API key。預設應該 gitignore，需要共享的部分另外處理。

### 給 Regulated Industry 的建議

如果你在 fintech、healthcare 等受管制產業：

- 評估 Anthropic 的 Enterprise tier（SSO、audit log、data isolation）
- 在 pilot 階段不要用包含 PII 的 codebase 測試
- 建立 AI code review checklist（不是只靠 Claude review，人也要 review）
- 諮詢你的 legal/compliance team 關於 AI-generated code 的使用政策

這不是完整的安全指南，但足夠讓你開始評估。企業級導入需要更深入的規劃 — 如果你想討論具體場景，歡迎來 Skool 社群提問。

---

## Lab 3：建立你的專案配置

這一章的練習目標：用 20 分鐘建出一個基礎但完整的配置。

**Step 1：寫 CLAUDE.md（10 分鐘）**

至少包含：
- 專案簡介（2-3 句話，技術棧）
- 目錄結構（哪些目錄做什麼、哪些不能動）
- 命名規範（至少 1 種，帶反例）
- 1 個特殊商業邏輯或隱性規則

**Step 2：建立 settings.json（5 分鐘）**

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(npm run *)",
      "Bash(git status)",
      "Bash(git diff)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)"
    ]
  }
}
```

根據你的專案調整 allow 清單。

**Step 3：加入第一個 Hook（5 分鐘）**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "npx prettier --write $FILE_PATH 2>/dev/null || true"
      }
    ]
  }
}
```

從自動 Prettier 開始。確認它跑起來之後，再考慮加其他 hook。

---

## 章末：最重要的一個檔案

我做了這麼多年 SRE，有一句話一直在腦子裡：

> 「你的系統有多可靠，取決於你的 infrastructure 設計有多嚴謹。」

CLAUDE.md 就是你 AI 工作流的 infrastructure。

你設計得好，Claude 每次都能在對的語境裡做對的事，不用你每次重新解釋。你設計得馬虎，你就會花大量時間修正 AI 的偏差、解釋它本來就應該知道的事。

這不是玄學，是工程。把規則寫清楚，把邊界劃清楚，把你的語境傳遞給它。

在任何 AI 專案裡，CLAUDE.md 是少數幾個你花時間在上面、報酬會複利增長的檔案。

---

想動手練？精華版 Workshop 有完整 Lab → https://www.skool.com/ai-brain-alex/classroom?ref=5dde9b20e8e7432aa9a01df6e89685f4
