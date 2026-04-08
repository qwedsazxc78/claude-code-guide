# 第七章：背景執行與平行開發

*一個人，做三個人的事*

SRE 出身的人有一個根深蒂固的直覺：卡住等待是在浪費資源。

你跑一個 CI pipeline，不會傻傻盯著螢幕等它跑完。你推一個 build，不會停下來等 deploy 結束。你知道平行處理是常識，不是優化。

Claude Code 的背景執行功能，就是把這個邏輯帶進 AI 開發工作流。這一章我們聊三件事：Background Agent 讓你的 AI 變成平行 process、Headless Mode 讓 Claude 進入 CI/CD pipeline、Agent Teams 的現實評估。

---

## 7.1 Background Agent — AI 的 parallel process

Background Agent 的概念很簡單：同時跑多個 Claude 實例，彼此互不干擾。

你在主對話裡繼續開發新功能，背景有一個 Agent 在跑 test suite，另一個在更新 API 文件。這不是科幻，這是我現在的日常。

### 兩種啟動方式

**VS Code 裡**，直接在主對話框告訴 Claude 你要背景執行：

```
你：「用背景任務幫我 review src/ 底下所有 error handling 的方式」
```

Claude 會啟動一個背景 Agent，你可以繼續在主對話做別的事。背景任務跑完會通知你結果。

**CLI 裡**，加一個 `&` 讓它跑在背景：

```bash
claude -p "幫我把所有 test file 補上 missing test cases" \
  --output-format stream-json &
```

### 三個最常用的場景

**場景一：主線寫功能，背景跑測試。**
你在實作 payment 邏輯，背景 Agent 同時跑整個 test suite。測試跑完報告結果，你不用中斷節奏。

**場景二：主線寫 code，背景更新文件。**
你在加一個新的 API endpoint，背景 Agent 同步把 README 和 API 文件更新好。

**場景三：多任務平行進行。**
```
背景 Agent 1：跑 unit tests
背景 Agent 2：更新 API 文件
背景 Agent 3：review 另一個 PR
→ 你在主線程繼續寫主要功能
```

### Alex 的日常使用模式

我現在一般會開 3 到 5 個背景任務同時跑。通常的組合是：main thread 寫主要功能、一個背景跑測試、一個背景做 code review、有時候再加一個背景更新文件。

不過有個坑要注意：**背景 Agent 和主線程千萬不能改到同一個檔案。** 兩個 process 同時寫同一個檔案，結果不可預期。開背景任務前，先想清楚工作邊界。

給剛開始用的人一個建議：先只開一個背景任務。習慣了再開兩三個。太多背景任務反而變成管理負擔——你花在追蹤每個任務狀態的時間，比任務本身節省的時間還多。把它想成你有一個很能幹的實習生，一次交代一件清楚的事就好。

什麼適合放到背景、什麼不適合：

| 適合背景執行 | 不適合背景執行 |
|------------|--------------|
| 跑測試 | 需要你即時做決定的任務 |
| 寫或更新文件 | 方向還不確定的開發 |
| Code review | 會碰到同一個檔案的任務 |
| 重複性的批次作業 | 需要即時互動的 debug |

---

## 7.2 Headless Mode — 讓 AI 進入 CI/CD Pipeline

Headless Mode 是 Claude Code 從「開發工具」升級成「自動化元件」的關鍵。

概念很直接：`-p` flag 讓 Claude 進入 pipe mode——沒有 UI，沒有互動，跑完輸出結果到 stdout 就結束。你可以把它放進任何 CI/CD pipeline、pre-commit hook、或 cron job 裡。

```bash
# -p 代表 headless (pipe mode)
claude -p "review 這個 PR 的 code quality，重點看安全性和效能"
```

Claude 會讀取 codebase、執行你的指令、輸出結果，然後退出。不等你回應，不問問題，就是跑。

### 認證方式

Headless 模式不能用瀏覽器登入，要用 API Key：

```bash
export ANTHROPIC_API_KEY="sk-ant-xxxx"
claude -p "你的指令"
```

### PR 自動 Code Review

這是我在 GitHub Actions 裡用的模式，也是我覺得 headless mode 最划算的應用：

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Claude Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          npx @anthropic-ai/claude-code -p \
            "review 這個 PR 的 code changes，
             重點看安全性和效能問題。
             用 markdown 格式輸出報告。"
```

每次有人開 PR，Claude 自動跑一輪 review，結果輸出到 CI log。

### 成本估算

每次 PR review 大概 $0.05 到 $0.20 美金，視 PR 大小和 codebase 複雜度而定。

乍看要花錢，但算一下帳就知道值。一個中型 PR，人工 review 最少 20 分鐘。Claude 跑 30 秒，抓出人 review 時容易忽略的安全漏洞、效能問題、或者邊緣情況。你的 review 時間可以集中在 Claude 沒辦法判斷的商業邏輯和架構決策上。

我把 Claude 的 CI review 想成一個「不會疲勞、不會急著去開會」的 first-pass reviewer。它沒辦法取代人，但它把人 review 的品質上限往上推了。

### SRE 的角度

從 SRE 的視角看 headless mode，本質就是「把 Claude 當成一個 CI job」。

它的行為跟其他 CI job 一樣：有 input（codebase + prompt）、有 output（分析結果）、有 exit code、可以設定 timeout、可以在 pipeline 裡串接。最大的差異是，這個 job 的「處理邏輯」不是寫死的規則，而是 LLM 的理解能力。

有一個 flag 要提醒你：`--dangerously-skip-permissions`。在 CI 環境裡，Claude 沒辦法互動式地問你要不要允許某個操作，所以有時候需要這個 flag 跳過權限確認。**在 CI 裡用是 OK 的，但在本地開發千萬不要用。** 名字裡有 dangerously 不是裝飾。

2026 年 3 月推出的 Auto Mode 是更安全的替代方案。在支援的環境裡，優先用 Auto Mode 而不是 `--dangerously-skip-permissions`。Auto Mode 背後有一個分類器監控每個動作，會擋住超出範圍的操作和 prompt injection，讓自動化更安全。

---

## 7.3 Agent Teams — 誠實評估

這個功能讓很多人興奮，但我要先說實話：**90% 的情況，Subagents + Background Agent 的組合已經夠用了。**

### Subagents vs Agent Teams

先搞清楚這兩個東西的差異：

| 特性 | Subagents | Agent Teams |
|------|-----------|-------------|
| 範圍 | 單一 session 內 | 跨 session |
| 溝通方式 | 回報給主 agent | Agents 之間互相溝通 |
| 穩定度 | 穩定，官方支援 | 實驗性 |
| 適合情境 | 快速子任務 | 大型跨模組協作 |

**Subagents** 是 Claude 在單一 session 裡自動產生的子代理。你說「幫我 review 這 20 個檔案的 error handling」，Claude 主 agent 會自動拆成三個子任務平行跑，結果匯總回來給你。這是穩定功能，你不需要做任何設定，Claude 自己決定要不要用。

**Agent Teams** 是另一回事。你要手動啟用實驗功能：

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

啟用後，多個 Claude 實例可以互相溝通、挑戰彼此的方案、平行執行不同模組的任務。理論上適合大型 refactoring 或 codebase migration 這種規模的工作。

### 什麼時候考慮用 Agent Teams

Agent Teams 的 token 消耗約 3 到 4 倍。三個 teammate 協作，token 用量是單一 session 的 3-4 倍，但省下的時間可能只有 2 倍。這個帳要算清楚。

還有穩定性問題——這是實驗性功能，可能遇到 agents 之間的結果衝突、重複工作、或者不一致的輸出。

我的實際建議：

- 日常開發 → Subagents，Claude 自動管理，不用你操心
- 大型遷移（整個 codebase 從 JS 轉 TypeScript）→ 可以考慮 Agent Teams
- 重要專案 → 不管哪種方式，最終都要人工 review

Agent Teams 很酷。但現在不是你必須用的東西。

### Alex 的三套 Agent System（進階預告）

我自己建了三套 multi-agent production 系統（Hera、Athena、Gaia），第九章會用 Gaia 做完整的深入拆解。這裡只要知道：它們的核心不是 Agent Teams 這個實驗功能，而是 Subagents + Background Agent + 明確的 CLAUDE.md 邊界定義。

---

## Workshop Lab 4：平行開發練習

這個 Lab 的目標很具體：體驗「主線工作 + 背景任務同時跑」的感覺。

**練習步驟：**

1. 在主對話，請 Claude 規劃一個小功能（用 Plan Mode）
2. 同時開一個背景任務：「背景幫我 review 目前 codebase 的命名規範，有沒有不一致的地方」
3. 繼續在主對話做功能規劃，讓背景任務自己跑
4. 背景任務完成後，看看它找到了什麼

這個練習沒有標準答案，但你會建立一個真實感受：AI 可以在你工作的同時幫你做另一件事。這個感受比任何說明都有說服力。

如果你想多試一步：設定 `ANTHROPIC_API_KEY`，然後跑 `claude -p "列出這個專案的技術棧和主要模組"`，感受一下 headless mode 的輸出格式。

---

**重點來了**：背景執行的本質是「用平行化換時間」。從 SRE 的角度看，這跟你管 production 系統時跑多個 monitoring job 的思路完全一樣——每個 job 各司其職，互不干擾。掌握了這個能力，你的開發速度不是線性成長，而是倍數成長。

第八章我們把前面所有工具串在一起，看看一個完整的日常工作流長什麼樣。
