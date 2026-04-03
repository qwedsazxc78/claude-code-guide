# 附錄 A：我該用哪個功能？— 初學者決策指南

> 這是全書最重要的一頁。如果你只能記住一個東西，記住這張圖。

---

## 一張圖搞懂 6 個擴充功能

```
你想讓 Claude...

  「記住我的專案規則」
  ──────────────────→  CLAUDE.md（專案層級）
                        放在專案根目錄，每次對話自動載入
                        例：命名規範、status 值、禁止改的目錄

  「記住我個人的偏好」
  ──────────────────→  Global CLAUDE.md（全域層級）
                        放在 ~/.claude/CLAUDE.md
                        例：語言偏好、常用快捷鍵提醒

  「自動做某件事」
  ──────────────────→  Hooks（事件觸發器）
                        設定在 settings.json，Claude 做某個動作時自動執行
                        例：改完 code 自動跑 Prettier、阻擋 rm -rf

  「重複用同一段 prompt」
  ──────────────────→  Commands（斜線指令）
                        .claude/commands/ 裡的 .md 檔案
                        例：/review、/deploy-check、/yt-init

  「讓 Claude 扮演特定角色」
  ──────────────────→  Agents（角色定義）
                        .claude/agents/ 裡的 .md 檔案
                        例：Code Reviewer（只讀不改）、Content Writer（繁中寫作）

  「讓 Claude 自動知道某個領域」
  ──────────────────→  Skills（知識模組）
                        .claude/skills/ 裡的資料夾
                        例：Python 開發規範、n8n workflow 慣例
```

---

## 用生活比喻理解

| 功能 | 比喻 | 一句話 |
|------|------|--------|
| **CLAUDE.md** | 新人入職手冊 | 「我們公司的規矩都在這，讀完再開工」 |
| **Global CLAUDE.md** | 你的個人簡歷 | 「不管去哪家公司，我都習慣這樣工作」 |
| **Hooks** | 門禁系統 | 「進門自動刷卡、離開自動關燈」 |
| **Commands** | SOP 文件 | 「每次做這件事，都照這個步驟」 |
| **Agents** | 部門專員 | 「這件事交給 QA 專員處理，他只負責測試」 |
| **Skills** | 培訓教材 | 「你來之前先讀這份資料，了解我們的背景」 |

---

## 決策樹：從你的需求開始

```
我遇到一個問題...

Q1: 這個問題是「每次都會碰到」還是「偶爾碰到」？

  偶爾碰到 → 直接在 Chat 裡問 Claude，不需要任何擴充
  
  每次都碰到 ↓

Q2: 你希望 Claude「自動做」還是「你手動觸發」？

  自動做 → Q3
  手動觸發 → Q4

Q3: 它是在「Claude 做了某件事之後」自動做嗎？

  是 → Hooks
      例：Claude 改完 .ts 檔 → 自動跑 ESLint
      例：Claude 要執行 rm -rf → 自動擋住

  不是，是每次對話開始時就要知道的 → CLAUDE.md 或 Skills
      專案規則 → CLAUDE.md
      領域知識（跨多個對話都需要）→ Skills

Q4: 你手動觸發的是「一段 prompt」還是「一個角色」？

  一段 prompt（步驟化的工作流）→ Commands
      例：/review → 每次用同樣的標準做 code review
      例：/yt-init 50 "標題" → 用固定流程建 YouTube 資料夾

  一個角色（有特定限制的 persona）→ Agents
      例：@code-reviewer → 只能讀檔案、不能改、不能執行指令
      例：@content-writer → 用繁中寫作、遵守 style-dna
```

---

## 從零開始的順序（不要一次全做）

大部分人的錯誤是：看了這麼多功能，想一次全部設定好。

**不要。** 按這個順序，一次加一個：

### 第一週：只用 CLAUDE.md

```
                    ┌─────────────┐
                    │  CLAUDE.md  │  ← 從這裡開始
                    └─────────────┘
```

寫 20 行就好。包含：
- 專案是做什麼的（2 句話）
- Tech stack
- 命名規範
- 「不要動 XXX 目錄」

這 20 行就能讓 Claude 從「通用 chatbot」變成「懂你專案的助手」。

### 第二週：加一個 Hook

```
                    ┌─────────────┐
                    │  CLAUDE.md  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Hooks    │  ← 加一個自動化
                    └─────────────┘
```

從最常見的開始：Claude 改完檔案後自動跑 formatter。

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "command": "npx prettier --write $FILE_PATH"
    }]
  }
}
```

這個 Hook 一設定，你再也不用手動跑 Prettier。

### 第三週：加一個 Command

```
                    ┌─────────────┐
                    │  CLAUDE.md  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Hooks    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Commands   │  ← 把重複的 prompt 存起來
                    └─────────────┘
```

找到你這一週打過最多次的 prompt，存成 `.claude/commands/review.md`：

```markdown
---
description: "Review code for quality and security"
---

Review the following code changes:
- Check naming conventions
- Look for security vulnerabilities (SQL injection, XSS)
- Verify error handling
- Check test coverage

$ARGUMENTS
```

現在你只要打 `/review` 就能用一致的標準做 code review。

### 第一個月以後：視需要加 Agent 和 Skill

```
                    ┌─────────────┐
                    │  CLAUDE.md  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Hooks    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Commands   │
                    └──────┬──────┘
                    ┌──────┴──────┐
              ┌─────▼─────┐ ┌────▼────┐
              │  Agents   │ │ Skills  │  ← 系統夠複雜才需要
              └───────────┘ └─────────┘
```

- **Agent**：當你需要 Claude 扮演一個有明確邊界的角色（例：只做 review 不能改 code）
- **Skill**：當你有一套領域知識需要在多個對話中重複使用（例：公司的 API 設計規範）

大部分人用 CLAUDE.md + 1-2 個 Hooks + 3-5 個 Commands 就夠了。Agent 和 Skill 是你的系統複雜到一定程度之後才需要的。

---

## 常見問題

### Q: CLAUDE.md 跟 Command 有什麼不同？

**CLAUDE.md** 是「自動載入的背景知識」— 每次對話都會讀，你不用做任何事。
**Command** 是「手動觸發的工作流」— 你輸入 `/指令名稱` 時才執行。

例：「所有檔案用 snake_case 命名」→ 放 CLAUDE.md（每次都要知道）
例：「幫我做 code review」→ 做成 Command（需要時才用）

### Q: Command 跟 Agent 有什麼不同？

**Command** 定義的是「做什麼事」— 一段帶有步驟的 prompt。
**Agent** 定義的是「用什麼身份做」— 一個有角色限制的 persona。

例：`/review` 是一個 Command — 「照這個標準做 review」
例：`@code-reviewer` 是一個 Agent — 「你是 reviewer，只能讀不能改」

Command 是 SOP，Agent 是人。你可以讓一個 Agent 去執行一個 Command。

### Q: Skill 跟 CLAUDE.md 有什麼不同？

**CLAUDE.md** 是專案級的規則 — 針對這個 repo 的命名、架構、慣例。
**Skill** 是領域級的知識 — 跨多個專案都適用的 pattern 和經驗。

例：「這個專案用 FastAPI + PostgreSQL」→ CLAUDE.md
例：「所有 Python 專案都用 uv 不用 pip、用 ruff 不用 flake8」→ Skill

### Q: Hook 跟 Command 有什麼不同？

**Hook** 是自動觸發的 — Claude 做了某個動作後自動執行。
**Command** 是手動觸發的 — 你輸入 `/指令名稱` 時執行。

例：「改完 .py 檔自動跑 ruff」→ Hook（自動，每次都跑）
例：「幫我做部署前檢查」→ Command（手動，需要時才用）

### Q: 我只想快速上手，該先學哪個？

```
CLAUDE.md → 第一天就該寫
Hooks → 第二週加一個
Commands → 第三週加一個
其他 → 之後再說
```

---

## 速查表

| 功能 | 放在哪裡 | 觸發方式 | 適合放什麼 |
|------|---------|---------|-----------|
| CLAUDE.md | 專案根目錄 | 自動（每次對話載入） | 專案規則、命名、架構 |
| Global CLAUDE.md | ~/.claude/CLAUDE.md | 自動（每次對話載入） | 個人偏好、語言、習慣 |
| Hooks | .claude/settings.json | 自動（事件觸發） | formatter、linter、阻擋危險指令 |
| Commands | .claude/commands/*.md | 手動（/指令名稱） | 重複的 prompt、工作流程 |
| Agents | .claude/agents/*.md | 手動（@角色名稱） | 有邊界限制的角色 |
| Skills | .claude/skills/*/ | 自動（符合條件時載入） | 領域知識、跨專案的 pattern |

---

*回到 [目錄](README.md)*
