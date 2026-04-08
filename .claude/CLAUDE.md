# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Claude Code 完全指南 — Agent Operating Context

> 繁體中文開源書，8 章 + 1 附錄，~80 頁 PDF
> Repo: https://github.com/qwedsazxc78/claude-code-guide

## What This Project Is

「Claude Code 完全指南」— 繁體中文市場首本 Claude Code 開源指南。
作者 Alex Hsieh（SRE/DevOps 背景），Cloud-F1 顧問。
授權：CC BY-NC-SA 4.0。

這本書是 Claude Code Workshop 產品線的頂部漏斗（lead magnet），
免費開源 → 建立權威 → 導流到付費課程 / 陪跑制 / 企訓。

## File Layout

```
├── README.md                      # GitHub 首頁 + TOC + badges
├── index.md                       # GitBook 全書導覽 + 讀法建議
├── SUMMARY.md                     # GitBook 導航
├── CHANGELOG.md                   # 版本紀錄
├── book.json                      # GitBook config
├── LICENSE                        # CC BY-NC-SA 4.0
├── CONTRIBUTING.md                # 貢獻指南
├── ch01-what-is-claude-code.md    # Part I：AI Coding 入門
├── ch02-setup-first-conversation.md
├── ch03-chat-debug.md             # Chat tab
├── ch04-cowork.md                 # Cowork tab（非工程師也能用）
├── ch05-code.md                   # Code tab
├── ch06-config-claudemd.md        # Part II：旗艦章（CLAUDE.md 工程）
├── ch07-background-automation.md  # 背景執行與平行開發
├── ch08-daily-workflow.md
├── ch09-advanced-preview.md       # 深入拆解（Gaia 案例）
└── appendix-a-decision-guide.md   # 初學者決策指南
```

## Hard Rules

### Voice DNA（Alex 語氣）
- 第一人稱（我）
- 對話感 + 權威感，誠實、不過度推銷
- **禁用詞：絕對、一定、最好的、唯一**（唯一例外：在說明 Voice DNA 規則本身時可以出現）
- Permission-based CTA（「如果你覺得可以，試試看」）
- 短段落（≤ 3 行）
- 招牌句：「你有沒有遇過這種情況？」「重點來了」「簡單來說」

### Skool CTA 規則
- **只有 ch01、ch06、ch09 有 Skool URL**
- 其他章節結尾用 chapter transition（「下一章我們聊...」），不放連結
- Referral URL: `https://www.skool.com/ai-brain-alex/about?ref=5dde9b20e8e7432aa9a01df6e89685f4`
- Classroom URL: `https://www.skool.com/ai-brain-alex/classroom?ref=5dde9b20e8e7432aa9a01df6e89685f4`

### 技術正確性 Checklist

每次修改內容前，確認以下資料仍正確（Claude Code 更新快）：

| 項目 | 上次驗證 | 驗證來源 |
|------|---------|---------|
| 定價：Pro $20 / Max $100 / Max $200 | 2026-04-03 | claude.com/pricing |
| 模型：Opus 4.6 / Sonnet 4.6 / Haiku 4.5 | 2026-04-03 | platform.claude.com/docs |
| 四種模式：Edit / Auto-Accept / Plan / Auto | 2026-04-03 | code.claude.com/docs |
| Hook exit codes：0=proceed / 2=block / 1=error | 2026-04-03 | code.claude.com/docs/en/hooks-guide |
| CoWork + Computer Use | 2026-04-03 | claude.com/product/cowork |
| Auto Mode（2026-03-24） | 2026-04-03 | claude.com/blog/auto-mode |
| @mention 語法 | 2026-04-03 | code.claude.com/docs/en/interactive-mode |

**更新流程：** 修改內容 → 跑一次 web search 確認 → 更新 CHANGELOG.md → commit

### 數字一致性 Checklist

修改任何數字時，grep 全書確認所有出現位置都同步：

| 數字 | 出現在 |
|------|--------|
| 28KB / 718 行 CLAUDE.md | ch01, ch02, ch03, ch06 |
| 200-300 則/天 | ch01, ch02, ch03, ch08 |
| $200/月 | ch01, ch02, ch08 |
| Hera 7 agents | ch01, ch07, ch09 |
| Athena 9 agents | ch01, ch07, ch09 |
| Gaia 4 agents + 69 commands | ch01, ch03, ch06, ch09 |
| Pro $20 / Max $100 / $200 | ch02 |
| /compact 30-40 則 | ch03, ch08 |
| Rate limit 每 2-3 小時 | ch02, ch08 |
| Plan 40% / Edit 50% / Auto-Accept 10% | ch05 |
| 50+ YouTube | ch01 |
| SRE/DevOps 背景 | ch01, ch06, ch07, ch08, README |

### Subtaglines

每章 H1 標題下有 italic subtagline。修改標題時同步更新：
- README.md 目錄表
- index.md 全書導覽
- SUMMARY.md GitBook 導航

## Related Projects

| Project | Path | 關係 |
|---------|------|------|
| content-asset-system | ~/Documents/git_saas/content-asset-system | 書的原始來源（assets/course-claude-code-workshop-short/book/） |
| ai-web-template | ~/Documents/git_saas/ai-web-template | Hera 7 agents，ch08 案例 |
| ai-coding-template | ~/Documents/git_saas/ai-coding-template | Athena 9 agents，ch08 案例 |

**同步規則：** 這個 repo 是 source of truth。修改後複製到 content-asset-system/book/。

## Common Commands

```bash
# GitBook 本地預覽（需先 npm install -g gitbook-cli）
gitbook serve

# GitBook 建置靜態檔
gitbook build

# 數字一致性檢查（修改數字後用）
grep -rn "28KB" ch*.md
grep -rn "200-300" ch*.md
grep -rn "Gaia" ch*.md
grep -rn "Hera" ch*.md
grep -rn "Athena" ch*.md

# 禁用詞掃描（提交前用）
grep -rn '絕對\|一定\|最好的\|唯一' ch*.md | grep -v 'Voice DNA'
```

## Commit Convention

```
feat: 新增內容 or 章節
fix: 修正錯誤 or 過時資訊
chore: CHANGELOG、CI、meta files
docs: README、CONTRIBUTING、LICENSE
```

所有 commit 加：`Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
