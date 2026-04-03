# 第一章：AI Coding 的時代來了

---

你有沒有遇過這種情況？

打開 ChatGPT，描述需求，複製貼上程式碼，貼到 IDE 裡，發現有 bug，再把錯誤訊息貼回去，來回搬運十幾次⋯⋯最後花在「搬 code」的時間，比真正寫程式的時間還長。

這不是你的問題。這是工具的問題。

**AI Coding 的時代來了，但大多數人還在用錯誤的姿勢用 AI。**

---

## 1.1 從 Copy-Paste 到真正的 AI Coding

我的背景是 SRE / DevOps，不是傳統軟體工程師。但我現在每天用 Claude Code 工作 6 小時以上，說實話，我已經回不去了。

說個真實數字：我每天在 Claude Code 裡送出大約 200-300 則訊息，一個月花大概 $200 美金。但省下的開發時間，保守估計每月值 NT$50,000 以上。我用它建了三套 Production 等級的 AI Agent 系統 —— Hera（7 個 Agent 做網站）、Athena（9 個 Agent 做 SaaS 開發）、Gaia（4 個 Agent + 69 個 Commands 管內容）。這不是業餘實驗，是每天在跑的生產系統。

不是因為 Claude Code 比 ChatGPT 聰明很多（模型層面其實差不多），而是因為**工作流完全不一樣**。

**Copy-Paste Coding（大多數人在做的）：**

1. 開 ChatGPT，描述需求
2. 複製 AI 產出的程式碼
3. 貼到 IDE 裡
4. 發現有 bug，再貼錯誤訊息回去
5. 來回 10 次，終於搞定

**AI Coding（真正的 AI Coding）：**

1. 在你的專案裡直接問 Claude Code
2. Claude 看得到你所有的檔案、你的架構、你的配置
3. Claude 直接在你的檔案裡改，你看 diff 決定要不要接受
4. 一次搞定

重點來了：兩者的差距不是 AI 有多聰明，而是**上下文（context）有多完整**。

ChatGPT 只看得到你貼給它的那段 code。Claude Code 看得到你整個 codebase、你的依賴、你的測試、你的設定檔。給它更完整的上下文，它就能給你更精準的答案。

---

## 1.2 Claude Code 是什麼？跟 ChatGPT 有什麼不同？

Claude Code 不是一般的 AI 聊天機器人。它是一個**直接嵌入你開發環境的 AI 工程師**。

它能做的事：

- **直接讀取你的整個 codebase** — 不用複製貼上
- **直接修改你的檔案** — 不用手動搬 code
- **直接執行命令** — 跑測試、git 操作、安裝套件
- **理解專案上下文** — 知道你的架構、風格、規範

| 能力比較 | ChatGPT / Web Chat | Claude Code |
|---------|-------------------|-------------|
| 看得到的範圍 | 你貼的那段 code | 整個 codebase |
| 能改檔案嗎 | 只能給你看 | 直接改，你 review diff |
| 能跑測試嗎 | 要你自己去 terminal 跑 | 直接跑，結果即時回報 |
| 理解的 context | 聊天內容 | 專案結構 + 設定 + 歷史 |
| 工作流 | 複製 → 貼上 → 複製 → 貼上 | 在專案裡直接工作 |

簡單來說：ChatGPT 是一個很聰明的顧問，你要把資料帶去給它看。Claude Code 是一個直接坐在你旁邊的工程師，它自己會看。

---

## 1.3 Claude Code vs Cursor vs Copilot — 怎麼選？

這是大家最常問我的問題。我的答案是：**這三個工具其實不完全重疊，可以依你的需求選。**

| 工具 | 強項 | 弱項 | 月費 |
|------|------|------|------|
| **Claude Code** | 複雜任務、架構分析、多檔編輯、自動化流程 | 純程式碼補全稍遜 | $20–$200 |
| **Cursor** | 程式碼補全、即時建議、習慣 VSCode 的用戶 | 複雜對話能力較弱 | $20 |
| **GitHub Copilot** | 整合 GitHub 生態、企業 SSO | 通用對話能力有限 | $10–$19 |

我的誠實意見：

我自己兩個都用。Cursor 負責日常 Tab 補全，大概佔我 20% 的使用時間。Claude Code 負責架構設計、跨檔案重構、和整個專案的配置管理，佔 80%。但說真話，從今年初開始，Cursor 我用得越來越少，因為 Claude Code 的 Plan Mode 加上 Background Agent，已經能覆蓋大部分情境。目前我還離不開 Cursor 的場景是：寫一行 code 的時候，Tab 自動補全真的很爽，Claude Code 目前還沒有這個。

---

## 1.4 三種介面：VS Code Extension / Terminal / CoWork

Claude Code 有三種使用方式，每種有不同的適合場景。

| 介面 | 恐懼指數 | 適合誰 | 功能完整度 |
|------|----------|--------|-----------|
| **VS Code Extension** | ⭐（最低） | 所有開發者 | 90% |
| **Terminal CLI** | ⭐⭐⭐ | 進階用戶 | 100% |
| **CoWork（桌面版）** | ⭐（最低） | 非工程師也能用 | 70% |

**VS Code Extension** 是我推薦的入門方式。不用離開熟悉的 IDE，所有功能都在側邊欄，視覺化操作。本書大部分內容都以這個介面示範。

**Terminal CLI** 是功能最完整的方式。支援 Headless 模式（無介面執行）、在 CI/CD pipeline 裡跑、批次處理任務。如果你不怕終端機，這個介面功能最強。

**CoWork（桌面版）** 是 Anthropic 推出的獨立應用程式，適合不熟悉 VS Code 的用戶。功能較簡化，但門檻最低。

我個人 80% 用 VS Code Extension，15% 用 Terminal（跑 Background Agent 和 CI/CD），5% 用 CoWork（示範給非工程師客戶看的時候）。在幫中華電信做企業訓練時，全部用 VS Code Extension 示範，因為那是工程師最熟悉的環境。

我的建議：**先在 VS Code 裡把 80% 的功能學會，再視需求進入 Terminal。** 大部分人在 VS Code 裡就已經夠用了。

---

## 1.5 本書的學習路線圖

這本書跟著「Claude Code 全攻略工作坊精華版」的四大支柱走：

| 章節 | 你會獲得的能力 |
|------|--------------|
| **第一章** AI Coding 的時代來了（你在這裡） | 理解 Claude Code 跟 ChatGPT 的本質差異，建立正確的 AI Coding 心智模型 |
| **第二章** 環境設定與第一次對話 | 完成安裝、選對方案、跑出第一次真正的 AI Coding 體驗 |
| **第三章** Chat — 學會跟 Claude 對話 | 用 @mention、Extended Thinking、Multi-tab 精準控制對話品質 |
| **第四章** Work — 讓 Claude 動手改你的 Code | 用 Plan Mode、Inline Diff、Checkpoint 安全地讓 AI 改你的 code |
| **第五章** Config — 把 Claude 配置成你的專屬工程師 | 寫 CLAUDE.md 和 Hooks，讓 Claude 理解你的規範、自動化重複流程 |
| **第六章** Cloud Worker — 背景執行與平行開發 | 開 Background Agent 處理大任務，用多 Agent 同時推進不同工作 |
| **第七章** 我的日常工作流與踩坑指南 | 看到一個真實工程師如何把 Claude Code 整合進每日開發節奏 |
| **第八章** 進階：MCP、Agents、Skills | 連接外部工具、建立自訂 Agent、打造屬於你的 AI 工作流擴充 |

每一章都有**觀念說明 + 實際操作範例**。如果你想邊讀邊練，精華版工作坊有完整的 Lab 環境，可以直接在真實專案裡操作。

---

我從 2025 年底開始全職用 Claude Code，到現在已經用它蓋了三套 Production Agent 系統、產出 50+ 集 YouTube 內容、服務了兩家企業客戶。這不是宣傳，是日常。下一章，我們來把它裝起來。

---

想動手練？精華版 Workshop 有完整 Lab → https://www.skool.com/ai-brain-alex/classroom?ref=5dde9b20e8e7432aa9a01df6e89685f4
