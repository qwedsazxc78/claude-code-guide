# Changelog

本書遵循 [Keep a Changelog](https://keepachangelog.com/) 格式。
Claude Code 更新快，這份 changelog 幫你追蹤書的內容是否跟上。

---

## [0.2.0] — 2026-04-08

重大結構重組：8 章 → 9 章，對齊 Claude Desktop App 三大 tab（Chat / Cowork / Code）。

### 結構變更

- **新增 ch04 Cowork**：獨立成章，聚焦本地檔案處理，生活化案例（Downloads 大掃除、收據→Excel）
- **ch03→ch09 全面重新編號**：Work→Code（ch05）、Config（ch06）、背景執行（ch07）、日常工作流（ch08）、進階（ch09）
- **ch07 移除 CoWork 節**：已獨立為 ch04
- **ch07 改名**：Cloud Worker → 背景執行與平行開發

### 內容更新

- ch03：新增 3.6 客服逐字稿分析案例（非工程師也能用的 Chat 場景）
- ch05：新增 5.6 Life OS /today 進階案例 + 更多 Code 使用場景連結
- ch09（原 ch08）：新增 MCP→CLI 趨勢觀察、Skill 作為 SOP 執行器
- 每章新增外部參考連結（use case 來源）

### 導航同步

- SUMMARY.md、README.md、index.md、CLAUDE.md 全部同步更新
- 所有 chapter transition（「下一章我們聊...」）更新為新章號
- 所有節號（4.x→5.x 等）同步更新

---

## [0.1.0] — 2026-04-07

首次公開發布。8 章 + 1 附錄，~80 頁 PDF，繁體中文。

### 內容

- **Part I 入門篇**（ch01-04）：AI Coding 入門、環境設定、Chat & Debug、Work & Plan
- **Part II 進階篇**（ch05-08）：CLAUDE.md 工程（旗艦章）、Cloud Worker、日常工作流、MCP/Commands/Agents/Skills 完整拆解
- **附錄 A**：初學者決策指南（CLAUDE.md / Hooks / Commands / Agents / Skills 決策樹）
- **index.md**：全書導覽 + 讀法建議

### 品質

- 經三方 review（讀者視角 / Tech Lead 視角 / 競爭者視角）修正
- 所有範例來自作者真實生產系統（Hera 7 agents / Athena 9 agents / Gaia 69 commands）
- 事實查核對照 2026-04 最新資料（Auto Mode、Hook exit codes、CoWork Computer Use）

### 已驗證正確性（截至 2026-04-03）

| 項目 | 狀態 |
|------|------|
| 定價：Pro $20 / Max $100 / Max $200 | ✅ |
| 模型：Opus 4.6 / Sonnet 4.6 / Haiku 4.5 | ✅ |
| 四種模式：Edit / Auto-Accept / Plan / Auto | ✅ |
| Hook exit codes：0=proceed / 2=block / 1=error | ✅ |
| CoWork + Computer Use（Mac 3/23, Windows 4/3）| ✅ |
| Auto Mode（2026-03-24 推出）| ✅ |

---

## 開發紀錄

### v1.0.0-rc6 — 事實查核修正

- ch04：三種模式→四種模式（新增 Auto Mode）
- ch05：Hook exit code 修正（0/2/1）
- ch06+ch07：加入 Auto Mode 說明
- ch06：CoWork 新增 Computer Use 能力
- ch03：@terminal:bash 加版本警語

### v1.0.0-rc5 — Subtaglines + 導覽

- 每章加入 italic subtagline
- 新增 index.md 全書導覽頁
- README 目錄表加入 subtagline 欄位

### v1.0.0-rc4 — 初學者決策指南

- 新增附錄 A：6 功能決策樹 + 速查表
- ch05/ch08 加入快速導覽表

### v1.0.0-rc3 — 三方 Review 修正

- ch08 從廣告改為真正深入章節（221→612 行，含 Gaia 完整拆解）
- 移除 5 個重複 Skool CTA，只保留 ch01/ch05/ch08
- ch06 Hera/Athena 描述統一
- ch05 新增 5.5 團隊 CLAUDE.md + 5.6 安全基礎
- ch03 新增 FastAPI 完整對話範例
- ch04 新增 TypeScript/Express 範例
- ch02 安裝步驟精簡

### v1.0.0-rc2 — Alex 真實故事 Enrichment

- 全書加入 Alex 的真實數字（$200/月、200-300 則/天）
- 替換所有假範例為真實 production bug stories
- 加入 SRE/DevOps 背景說明
- 加入 3 套 Agent System 介紹

### v1.0.0-rc1 — 初版

- 8 章完整內容
- GitBook 基礎設施（SUMMARY.md、book.json、LICENSE、CONTRIBUTING.md）
