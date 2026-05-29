<a id="the-2026-tool-use-and-computer-agent-landscape"></a>
# 2026 年 Tool-Use 與 Computer Agent 全景圖

AI 代理與外部世界互動的方式已經出現劇烈轉變。在 2024 年，「tool use」通常是指模型輸出一個 JSON function call，再由你的後端去執行。到了今天，我們已經擁有真正的自主代理：它們會 clone repo、執行 shell 指令、透過螢幕截圖控制桌面，甚至在 WhatsApp 上傳訊息給你，而這一切都透過像 MCP 這類標準化協定來協調。本章將梳理這些工具的生態、架構，以及讓它們彼此不同的設計決策。

<a id="table-of-contents"></a>
## 目錄

- [生態系總覽](#ecosystem-overview)
- [分類法](#category-taxonomy)
- [OpenClaw：爆紅的個人 AI 代理](#openclaw)
- [OpenHands：自主開發者代理](#openhands)
- [Open Interpreter：本地程式碼執行](#open-interpreter)
- [Claude Computer Use：基於視覺的自動化](#claude-computer-use)
- [Claude Code：終端代理](#claude-code)
- [IDE 代理：Cursor、Windsurf、Cline](#ide-agents)
- [比較矩陣](#comparison-matrix)
- [市場趨勢與採用情況（2026）](#market-trends)
- [系統設計面試切入點](#interview-angle)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="ecosystem-overview"></a>
## 生態系總覽

2026 年的 tool-use 生態系已經收斂成四種架構模式，每一種都針對不同程度的自主性、安全性與整合深度做了最佳化：

```
+-----------------------------------------------------------------------+
|                     2026 Tool-Use Ecosystem                           |
+-----------------------------------------------------------------------+
|                                                                       |
|  +-------------------+  +-------------------+  +-------------------+  |
|  |  LOCAL AGENTS     |  |  CLOUD AGENTS     |  |  IDE AGENTS       |  |
|  |                   |  |                   |  |                   |  |
|  |  OpenClaw         |  |  Claude Code      |  |  Cursor           |  |
|  |  Open Interpreter |  |  OpenAI Codex     |  |  Windsurf         |  |
|  |  OpenHands (local)|  |  OpenHands Cloud  |  |  Cline            |  |
|  |  LM Studio Agent  |  |  Google Jules     |  |  GitHub Copilot   |  |
|  +-------------------+  +-------------------+  +-------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +-------------------+  |
|  |  COMPUTER-USE     |  |  MCP SERVERS      |  |  MESSAGING AGENTS |  |
|  |                   |  |                   |  |                   |  |
|  |  Claude Computer  |  |  10,000+ servers  |  |  OpenClaw (multi) |  |
|  |  Use API          |  |  97M monthly SDK  |  |  Custom bots      |  |
|  |  Open Interpreter |  |  downloads        |  |  via MCP bridges  |  |
|  |  (Computer API)   |  |                   |  |                   |  |
|  +-------------------+  +-------------------+  +-------------------+  |
+-----------------------------------------------------------------------+
```

2026 年最重要的洞察是：這些類別正在彼此收斂。Claude Code 是在本地執行的雲端代理；OpenClaw 是會連接雲端 LLM 的本地代理；Cursor 是帶有雲端 Background Agents 的 IDE 代理。界線正變得模糊，真正重要的是底層的 **architecture pattern**（下一章會介紹）。

---

<a id="category-taxonomy"></a>
## 分類法

<a id="1-local-agents-self-hosted-user-controlled"></a>
### 1. 本地代理（Self-Hosted、User-Controlled）

這類代理運行在使用者自己的硬體上。LLM 呼叫可以送往雲端，但代理程序、記憶體與工具執行都留在本地。

**關鍵特性：**
- 可完整存取使用者機器上的檔案系統
- 持久記憶儲存在本地（SQLite、JSON、Markdown）
- 使用者擁有全部資料，不受 vendor lock-in 限制
- 安全責任完全由操作方承擔

**範例：** OpenClaw、Open Interpreter、本地部署的 OpenHands

<a id="2-cloud-agents-vendor-hosted-api-driven"></a>
### 2. 雲端代理（Vendor-Hosted、API-Driven）

這類代理運行在供應商管理的雲端環境中。程式碼執行會發生在沙箱化的 VM 或 container 內。

**關鍵特性：**
- 沙箱化執行（Docker、Firecracker VM、E2B）
- 無法存取本地檔案系統（在 clone 下來的 repo 上工作）
- 供應商負責擴展性、安全與基礎設施
- 採按使用量計費或訂閱制

**範例：** Claude Code（cloud mode）、OpenAI Codex、Google Jules、OpenHands Cloud

<a id="3-ide-agents-editor-integrated-context-aware"></a>
### 3. IDE 代理（Editor-Integrated、Context-Aware）

這類代理直接嵌入程式碼編輯器，對專案結構、開啟中的檔案與編輯器狀態有深入理解。

**關鍵特性：**
- 與編輯器 UI 緊密整合（inline diff、tab completion）
- 透過 embeddings 或 AST parsing 進行 codebase indexing
- 可在 branch 上非同步工作的 Background Agents
- 為開發者工作流最佳化，而非通用自動化

**範例：** Cursor（Agent Mode + Background Agents）、Windsurf（Cascade）、Cline、GitHub Copilot

<a id="4-computer-use-agents-vision-based-gui-driven"></a>
### 4. Computer-Use 代理（Vision-Based、GUI-Driven）

這類代理以人類操作軟體的方式與系統互動──也就是看著螢幕截圖再進行點擊。

**關鍵特性：**
- 模型查看螢幕截圖，決定滑鼠／鍵盤動作
- 可搭配任何應用程式使用（不需要 API）
- 延遲較高（每一步 screenshot-action loop 約 1–3 秒）
- 為了安全必須使用沙箱環境（VM + VNC）

**範例：** Claude Computer Use API、Open Interpreter Computer API

---

<a id="openclaw"></a>
<a id="openclaw-the-viral-personal-ai-agent"></a>
## OpenClaw：爆紅的個人 AI 代理

<a id="what-it-is"></a>
### 它是什麼

OpenClaw 是由奧地利開發者 Peter Steinberger 建立的 self-hosted 開源個人 AI 助理。它最早在 2025 年 11 月以「Clawdbot」之名發布，並於 2026 年 1 月更名為 OpenClaw。它在不到五個月內從 0 成長到 346,000 顆 GitHub stars，並於 2026 年 3 月 3 日超越 React，成為 GitHub 上 star 數最高的軟體專案。

**數字一覽（2026 年 5 月）：**
- 346,000+ GitHub stars
- 320 萬活躍使用者
- 500,000+ 個執行中的實例
- ClawHub 上有 44,000+ 個社群 skills
- 專案網站每月 3,800 萬訪客
- 24+ 個訊息平台整合

<a id="how-it-works"></a>
### 運作方式

OpenClaw 的架構由六個核心元件構成：

```
+-------------------------------------------------------------------+
|                      OpenClaw Architecture                        |
+-------------------------------------------------------------------+
|                                                                   |
|  +-----------+     +-----------+     +----------+                 |
|  |  Gateway  |---->|  LLM      |---->| PI Agent |                 |
|  |           |     |  (Brain)  |     | (Exec)   |                 |
|  +-----------+     +-----------+     +----------+                 |
|       ^                  |                |                       |
|       |                  v                v                       |
|  +-----------+     +-----------+     +----------+                 |
|  | Channels  |     | SOUL.md   |     | Skills   |                 |
|  | (24+)     |     | (Identity)|     | (44K+)   |                 |
|  +-----------+     +-----------+     +----------+                 |
|                          |                                        |
|                          v                                        |
|                    +-----------+                                  |
|                    | Memories  |                                  |
|                    | (Persist) |                                  |
|                    +-----------+                                  |
+-------------------------------------------------------------------+
```

**1. Gateway**：訊息進出層。可連接 WhatsApp（透過 Baileys）、Telegram、Discord、Slack、Signal、iMessage、Microsoft Teams、Matrix，以及另外 16+ 個平台。支援私訊與群組對話，並可用 mention 觸發。

**2. LLM（The Brain）**：設計上與模型無關。支援 GPT-4o、Claude、Gemini、DeepSeek，或透過 Ollama 使用本地模型。模型由使用者自行選擇；架構本身不受限。

**3. PI Agent（Process Interactor）**：一個小型 runtime，讓 LLM 可以在宿主系統上建立、編輯、執行與刪除檔案。LLM 產生程式碼，PI Agent 將其儲存後再執行。這就是代理的「手」。

**4. `SOUL.md`（Identity Layer）**：一個純 Markdown 檔案，用來定義代理的人格、溝通風格、價值觀與行為護欄。它會在 session 開始時載入，並注入 system prompt。每個代理實例都會先讀取 `SOUL.md`──也就是「先讀懂自己，才開始存在」。

**5. Skills（Plugin System）**：為代理提供新能力的擴充系統。ClawHub 上已有超過 44,000 個社群 skills。Skills 遵循 AgentSkills 規格，可被打包、放在 workspace 本地，或全域安裝。

**6. Memories（Persistent Context）**：儲存在本地的長期記憶。代理會在多次對話間持續累積對使用者的理解。配合 `SOUL.md`，每個代理都能在所有訊息平台上維持一致的人格。

<a id="workspace-files"></a>
### Workspace 檔案

| 檔案 | 用途 |
|------|------|
| `SOUL.md` | 代理人格、語氣、價值觀、護欄 |
| `AGENTS.md` | 操作指令、工具設定 |
| `HEARTBEAT.md` | 排程式自主動作（類 cron） |
| `Memories/` | 橫跨多次對話的持久上下文 |

<a id="security-concerns"></a>
### 安全疑慮

OpenClaw 的高速成長已經超越其安全實務的成熟速度。截至 2026 年 5 月，已有超過 135,000 個實例暴露在公網上，其中許多仍使用預設設定。ClawHub skills 市集幾乎沒有安全審查：skills 是附帶選用 TypeScript 的 Markdown，容易建立、容易安裝，也容易被濫用。對任何想在正式環境部署 OpenClaw 的人來說，這都是關鍵設計考量。

---

<a id="openhands"></a>
<a id="openhands-autonomous-developer-agent"></a>
## OpenHands：自主開發者代理

<a id="what-it-is-1"></a>
### 它是什麼

OpenHands（前身為 OpenDevin）是一個開源的自主 AI 軟體工程師。它採 MIT 授權，能修改程式碼、執行指令、瀏覽網頁並與 API 互動。不同於只提供程式碼片段建議的工具，OpenHands 會 clone repository、執行終端指令、跑測試，並在沙箱化 Docker container 內除錯錯誤。

<a id="architecture-event-stream-sandboxed-runtime"></a>
### 架構：Event-Stream + Sandboxed Runtime

```
+-------------------------------------------------------------------+
|                     OpenHands Architecture                        |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |   User / API     |                                             |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+     +------------------+                    |
|  |  Agent Controller |<--->|  Event Stream    |                   |
|  |  (CodeAct 1.0)   |     |  Hub             |                   |
|  +--------+---------+     +--------+---------+                    |
|           |                        |                              |
|           v                        v                              |
|  +--------+---------+     +--------+---------+                    |
|  |  Action Dispatch  |     |  Observation     |                   |
|  |                   |     |  Collector       |                   |
|  |  - CmdRunAction   |     |                  |                   |
|  |  - FileWriteAction|     |  - CmdOutput     |                   |
|  |  - BrowseURLAction|     |  - FileContent   |                   |
|  |  - CodeAction     |     |  - BrowserState  |                   |
|  +--------+---------+     +------------------+                    |
|           |                                                       |
|           v                                                       |
|  +--------+--------------------------------------------------+    |
|  |              Docker Sandbox (Per Session)                 |    |
|  |                                                           |    |
|  |  +----------+  +----------+  +----------+                |    |
|  |  | Terminal  |  |  Python  |  | Browser  |                |    |
|  |  | (bash)   |  | (stateful)|  | (BrowserGym)             |    |
|  |  +----------+  +----------+  +----------+                |    |
|  +-----------------------------------------------------------+    |
+-------------------------------------------------------------------+
```

**關鍵架構決策：**
- **Event-stream architecture**：所有代理與環境的互動都會以型別化事件流經中央 hub。Agent 會分析對話狀態並產生 Actions；sandbox 產生 Observations。
- **每個 session 一個 Docker container**：每個 session 都有自己獨立隔離的 container，並具備完整 OS 能力。container 與宿主機隔離。
- **CodeAct 1.0**：預設的代理範本。它把 LLM 推理嵌入統一的 coding control plane，並維持 session 等級的專案上下文。
- **BrowserGym 整合**：代理可以透過宣告式 primitive（DOM 操作、導覽）進行瀏覽器自動化。
- **SDK composability**：OpenHands SDK 是一個 Python 函式庫。你可以用程式碼定義代理、在本地執行，或擴展到雲端上的數千個代理。

**近期更新（v1.6.0，2026 年 3 月）：**
- 支援以 Kubernetes 協調代理 session
- Planning Mode beta，可做多步任務拆解
- 188+ 位貢獻者帶來 2,100+ 次貢獻

---

<a id="open-interpreter"></a>
<a id="open-interpreter-local-code-execution"></a>
## Open Interpreter：本地程式碼執行

<a id="what-it-is-2"></a>
### 它是什麼

Open Interpreter 是一個本地程式碼執行代理，提供類似 ChatGPT 的終端介面。它不是只顯示程式碼並要求你自己執行，而是會先請求授權，再直接在你的機器上執行，並完整存取你的本地檔案。

<a id="architecture"></a>
### 架構

```
+-------------------------------------------------------------------+
|                  Open Interpreter Architecture                    |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  Terminal UI      |                                            |
|  |  (ChatGPT-like)  |                                            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+     +------------------+                    |
|  |  Core Engine      |<--->|  LLM Provider   |                   |
|  |                   |     |  (100+ models)  |                    |
|  |  - NL to Code     |     |  GPT, Claude,   |                   |
|  |  - Permission     |     |  Ollama, LM     |                   |
|  |    Gate            |     |  Studio, etc.   |                   |
|  +--------+---------+     +------------------+                    |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  Code Executor    |                                            |
|  |                   |                                            |
|  |  - Python         |                                            |
|  |  - JavaScript     |                                            |
|  |  - Shell/Bash     |                                            |
|  |  - AppleScript    |                                            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  Computer API     |                                            |
|  |  (GUI Control)    |                                            |
|  |                   |                                            |
|  |  - Screen capture |                                            |
|  |  - Mouse/Keyboard |                                            |
|  |  - Icon detection |                                            |
|  +-------------------+                                            |
+-------------------------------------------------------------------+
```

**關鍵特性：**
- **模型彈性**：可搭配 100+ 個 LLM。若要最高能力，可用 GPT-4o 或 Claude；若重視隱私，也可透過 Ollama 與 LM Studio 完全離線運行。
- **Permission gate**：每次程式碼執行都需要使用者核准（對受信任工作流可關閉）。
- **Computer API**：除了執行程式碼外，Open Interpreter 也能看見你的螢幕、辨識 UI 元素並控制滑鼠與鍵盤──使它從 code interpreter 升級為 computer automation agent。
- **預設不沙箱化**：它直接在宿主機上執行。這是為了最大能力而刻意做出的設計選擇，但也表示錯誤的 LLM 輸出可能損害你的系統。Docker sandboxing 屬於可選項。

<a id="when-to-use-open-interpreter"></a>
### 何時使用 Open Interpreter

它最適合資料分析、檔案操作與系統管理這類任務，尤其當你想用對話式介面操作本地機器時。不適合正式環境部署，也不適合不受信任的環境。

---

<a id="claude-computer-use"></a>
<a id="claude-computer-use-vision-based-automation"></a>
## Claude Computer Use：基於視覺的自動化

<a id="what-it-is-3"></a>
### 它是什麼

Claude Computer Use 是 Anthropic API 的一項功能，讓 Claude 能透過螢幕截圖、滑鼠移動、鍵盤輸入與應用程式互動來控制桌面。它在 2024 年 10 月以 beta 形式推出，之後能力顯著演進。截至 2026 年 5 月，Sonnet 4.6 在 OSWorld-Verified 上達到 72.5%，相較於剛推出時的 14.9% 大幅提升，而 Opus 4.7 在 agentic coding benchmarks 上又更進一步（64.3% SWE-bench Pro）。

<a id="the-vision-action-loop"></a>
### Vision-Action Loop

```
+-------------------------------------------------------------------+
|              Claude Computer Use: Vision-Action Loop               |
+-------------------------------------------------------------------+
|                                                                   |
|  Step 1: OBSERVE          Step 2: REASON          Step 3: ACT    |
|  +----------------+       +----------------+      +------------+ |
|  |  Take          |       |  Analyze       |      |  Execute   | |
|  |  Screenshot    |------>|  Screenshot    |----->|  Action    | |
|  |  (base64 PNG)  |       |  + Task Goal   |      |  (click,   | |
|  |                |       |  + History      |      |  type,     | |
|  +----------------+       +----------------+      |  scroll)   | |
|                                                    +------+-----+ |
|                                                           |       |
|          +------------------------------------------------+       |
|          |                                                        |
|          v                                                        |
|  +-------+--------+                                               |
|  |  Wait + Take   |                                               |
|  |  New Screenshot|-------> (Loop back to Step 1)                 |
|  +----------------+                                               |
|                                                                   |
+-------------------------------------------------------------------+
```

<a id="available-tools"></a>
### 可用工具

| 工具 | 能力 | 備註 |
|------|------|------|
| `computer` | 滑鼠、鍵盤、截圖 | 完整桌面 GUI 控制 |
| `bash` | 執行 shell 指令 | session 可跨多輪持續 |
| `text_editor` | 讀取／寫入／編輯檔案 | 支援 view、create、str_replace |

<a id="2026-enhancements"></a>
### 2026 年強化項目

- **Zoom Action**：在點擊前以高解析度檢查微小 UI 元素，可降低密集介面中的誤點率。
- **可於 Claude Cowork 與 Claude Code 使用**：目前為 Pro 與 Max 使用者提供研究預覽，且破壞性操作前需要人工確認。
- **Sandboxing 最佳實務**：務必在沙箱化 VM 中執行（Docker + VNC，或 E2B cloud）。絕對不要讓 computer-use 存取未沙箱化的宿主機。

<a id="performance-trajectory"></a>
### 效能演進軌跡

| 日期 | OSWorld 分數 | 關鍵里程碑 |
|------|--------------|------------|
| 2024 年 10 月 | 14.9% | Beta 推出（Claude 3.5 Sonnet） |
| 2025 年中 | ~40% | Claude 3.7 改進 |
| 2026 年 Q1 | 72.5% | Sonnet 4.6、Zoom Action |

---

<a id="claude-code"></a>
<a id="claude-code-the-terminal-agent"></a>
## Claude Code：終端代理

<a id="what-it-is-4"></a>
### 它是什麼

Claude Code 是 Anthropic 的 agentic coding 工具，直接活在終端機裡。它會讀取你的 codebase、編輯檔案、執行指令，並整合開發工具。它於 2025 年 5 月公開推出，並在 2026 年 2 月前突破 25 億美元 ARR。

<a id="architecture-1"></a>
### 架構

Claude Code 是一個 TypeScript 終端代理，會在三個階段中反覆循環：

```
+-------------------------------------------------------------------+
|                   Claude Code Agent Loop                          |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  1. GATHER       |  Read files, grep codebase, glob search,   |
|  |     CONTEXT      |  check git status, analyze structure        |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  2. TAKE         |  Edit files, run bash, write new files,     |
|  |     ACTION       |  create commits, spawn subagents            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  3. VERIFY       |  Run tests, check build, review diffs,     |
|  |     RESULTS      |  validate output                            |
|  +--------+---------+                                             |
|           |                                                       |
|           +--------> (Loop back to Step 1 if not done)            |
|                                                                   |
+-------------------------------------------------------------------+

Built-in Tools: bash, read, write, edit, glob, grep, browser,
                subagent, notebook, web_search, web_fetch
```

**關鍵架構特性：**
- 單一代理迴圈，搭配豐富的工具盤
- 透過 slash commands 與 `CLAUDE.md` 按需載入 skills
- 長 session 的上下文壓縮（1M+ token context）
- 可啟動 subagent 以平行處理不同工作流
- 使用 worktree 隔離平行 branch 執行
- 權限治理（工具 allow/deny 規則）
- 具依賴圖的 task system
- 可掛接自訂自動化 hooks（pre/post commit、檔案變更）

---

<a id="ide-agents"></a>
<a id="ide-agents-cursor-windsurf-cline"></a>
## IDE 代理：Cursor、Windsurf、Cline

<a id="cursor"></a>
### Cursor

Cursor 是一個深度整合 AI 的 VS Code fork。2.0 版（2026 年初）引入了：
- **Agent Mode**：使用放大 20 倍的 reinforcement learning 進行多檔編輯
- **Background Agents**：在雲端 VM 中 clone 你的 repo、自主工作，完成後開 PR
- **Mission Control**：用來管理平行代理工作流的儀表板
- **市場表現**：20 億美元年化營收、200 萬+ 使用者、100 萬+ 付費客戶，並被半數 Fortune 500 採用

<a id="windsurf"></a>
### Windsurf

Windsurf（原名 Codeium，於 2025 年 7 月被 Cognition 以 2.5 億美元收購）具備：
- **Cascade**：多步驟 AI 代理，可分析專案結構、協調跨檔案變更，並從錯誤中自行恢復
- **專有模型**：SWE-1.5（比 Sonnet 4.5 快 13 倍）與 Fast Context
- **Codemaps**：AI 驅動的視覺化程式碼導覽
- **跨 IDE 外掛**：可用於 40+ 個 IDE（JetBrains、Vim、NeoVim、XCode）

<a id="cline"></a>
### Cline

Cline 是一個 VS Code 擴充套件，它以完整代理而非 autocomplete 工具的方式運作。它會執行一連串步驟、評估結果、修正自己的錯誤，然後繼續前進。相較於 Cursor 或 Windsurf，它更自主，但打磨程度較低。

<a id="ide-agent-architecture-comparison"></a>
### IDE 代理架構比較

```
+-------------------------------------------------------------------+
|                 IDE Agent Architecture Patterns                   |
+-------------------------------------------------------------------+
|                                                                   |
|  Cursor:                                                          |
|  [Editor] --> [Agent Mode] --> [Multi-file RL] --> [Apply Diffs]  |
|                    |                                               |
|                    +--> [Background Agent] --> [Cloud VM] --> [PR] |
|                                                                   |
|  Windsurf:                                                        |
|  [Editor] --> [Cascade Agent] --> [RAG Codebase] --> [Apply Edits]|
|                    |                                               |
|                    +--> [SWE-1.5 Model] --> [Fast Context]        |
|                                                                   |
|  Cline:                                                           |
|  [Editor] --> [Agent Loop] --> [Evaluate] --> [Self-Fix] --> [Act]|
|                    |                                               |
|                    +--> [Any LLM Provider] --> [Tool Calls]       |
+-------------------------------------------------------------------+
```

---

<a id="comparison-matrix"></a>
## 比較矩陣

| 功能 | OpenClaw | OpenHands | Open Interpreter | Claude Computer Use | Claude Code | Cursor |
|------|----------|-----------|------------------|---------------------|-------------|--------|
| **類型** | 本地代理 | 開發代理 | 本地程式碼執行 | 視覺自動化 | 終端代理 | IDE 代理 |
| **授權** | AGPL-3.0 | MIT | AGPL-3.0 | 專有 API | 專有 | 專有 |
| **GitHub Stars** | 346K | 51K+ | 58K+ | N/A（API） | 42K+ | N/A |
| **沙箱化** | 否（宿主機） | 是（Docker） | 否（宿主機） | 需要 VM | 可設定 | 是（BG agents） |
| **LLM 支援** | 任意（model-agnostic） | 任意 | 100+ 模型 | 僅 Claude | 僅 Claude | 多模型 |
| **GUI 控制** | 否 | 是（BrowserGym） | 是（Computer API） | 是（原生） | 透過 computer-use | 否 |
| **程式碼執行** | 是（PI Agent） | 是（container） | 是（本地） | 是（`bash` 工具） | 是（`bash`） | 是（terminal） |
| **訊息整合** | 24+ 平台 | Web UI / API | 終端 | API | 終端 / IDE | 編輯器 |
| **記憶體** | 持久化（本地） | 依 session | 依 session | 每段對話 | Session + `CLAUDE.md` | 專案範圍 |
| **MCP 支援** | 社群 skills | 有限 | 否 | 透過 Claude | 原生 | 持續成長中 |
| **最適合** | 個人助理 | 自主開發 | 資料分析 | GUI 自動化 | 專業開發 | IDE 工作流 |
| **風險等級** | 高（未沙箱化） | 低（已沙箱化） | 高（未沙箱化） | 中（需要 VM） | 中 | 低 |

---

<a id="market-trends"></a>
<a id="market-trends-and-adoption-2026"></a>
## 市場趨勢與採用情況（2026）

<a id="the-numbers"></a>
### 數字面

- **MCP 生態系**：10,000+ 個活躍 server、每月 9,700 萬次 SDK 下載
- **Gartner 預測**：到 2026 年底，40% 的企業應用程式將納入 AI agents（高於 2025 年初不到 5%）
- **OpenClaw**：史上最快達到 300K GitHub stars 的專案（不到 5 個月）
- **Claude Code**：到 2026 年 2 月達成 25 億美元 ARR──史上最快達到 10 億美元的企業軟體產品
- **Cursor**：20 億美元年化營收，打入半數 Fortune 500

<a id="key-trends"></a>
### 關鍵趨勢

**1. 代理類型正在收斂**：本地、雲端與 IDE 代理之間的邊界正在消失。Claude Code 在本地執行，但使用雲端模型。Cursor 的 Background Agents 跑在雲端。OpenClaw 可連到任何 LLM。整體趨勢正走向能在任意環境運作的通用代理架構。

**2. MCP 成為通用工具層**：MCP 已成為工具整合標準，並被 Anthropic、OpenAI、Google，以及數百個工具供應商採用。2026 年 roadmap 聚焦於 enterprise readiness：identity propagation、tool budgeting、結構化錯誤語意與 audit trails。

**3. Sandboxing 變成不可妥協的要求**：OpenClaw 的安全危機（135,000 個暴露實例）推動整個產業走向預設沙箱化的架構。新一代代理預期必須開箱即提供隔離能力。

**4. 成本最佳化成為一級議題**：Plan-and-Execute 模式（由能力較強的模型規劃、較便宜的模型執行）可降低 90% 成本。這相當於代理世界的雲端成本最佳化。

**5. 背景化與非同步代理**：Cursor 的 Background Agents 與 Claude Code 的 subagent spawning，代表系統正從同步、互動式代理，轉向完成後主動通知你的自主非同步工作者。

---

<a id="interview-angle"></a>
<a id="system-design-interview-angle"></a>
## 系統設計面試切入點

當你在系統設計面試中被問到 tool-use 代理時，請聚焦以下維度：

**1. 安全模型**：執行環境是否沙箱化？憑證如何管理？如果 LLM 產生惡意程式碼會發生什麼事？（OpenClaw 的 AGPL 授權與未沙箱化執行，對比 OpenHands 的 Docker 隔離，是很好的比較點。）

**2. 狀態管理**：代理如何在多次工具呼叫之間維持上下文？是 session-based（OpenHands）、persistent memory（OpenClaw），還是 file-based（Claude Code 的 `CLAUDE.md`）？

**3. 工具探索**：是靜態 manifest（舊方法）、透過 MCP 動態探索，還是 skill marketplace（OpenClaw ClawHub）？

**4. 延遲預算**：Function calling（每次工具呼叫 50–200ms）對比 vision-based automation（每次 screenshot-action loop 1–3 秒）。這會如何影響 UX？

**5. 失敗處理**：工具呼叫失敗時會怎樣？重試？fallback？human-in-the-loop？在放棄之前要重試幾次？

---

<a id="interview-questions"></a>
## 面試題

<a id="q-your-team-wants-to-build-an-internal-ai-assistant-should-you-build-on-openclaw-openhands-or-build-custom-with-claude-code-mcp"></a>
### Q：你的團隊想打造內部 AI 助理。你應該基於 OpenClaw、OpenHands，還是用 Claude Code + MCP 自行打造？

**有力回答：**
這取決於使用情境與安全需求。OpenClaw 針對具備訊息整合的個人助理做了最佳化──如果目標是打造 Slack/Teams bot，且需要持久人格，它很理想。但它未沙箱化的執行方式與 AGPL 授權，會帶來企業層面的疑慮。OpenHands 更適合自主開發任務──它的 Docker 沙箱與 MIT 授權都對企業較友善。若要打造客製化內部工具，Claude Code 搭配 MCP servers 會提供最高控制力：你可以精確定義可用工具、在自己的基礎設施中執行它們，並享有 MCP 標準化 discovery 與 auth 的優勢。決策樹大致是：訊息優先？OpenClaw。開發自動化？OpenHands。企業級客製工具？MCP + 你自己的 agent loop。

<a id="q-how-would-you-design-a-system-that-lets-non-technical-users-automate-desktop-tasks-using-ai"></a>
### Q：你會如何設計一套系統，讓非技術使用者能用 AI 自動化桌面任務？

**有力回答：**
我會採用 vision-based computer-use 模式（Claude Computer Use 或類似方案）。關鍵設計決策有五點：(1) 一律在沙箱化 VM 中執行，避免代理損害使用者真正的機器。(2) 任何破壞性動作前都實作 Human-in-the-Loop 確認步驟──例如刪除檔案、送出表單、購買。(3) 使用 Zoom Action 模式降低密集 UI 上的誤點。(4) 設定 token／成本上限，以避免 runaway loop。(5) 將所有動作記錄為 audit trail。主要權衡是延遲──每一步 screenshot-action 大約需 1–3 秒──但這種方式不需要 API，就能操作任何應用程式。若要加速工作流，可在具備 API 的應用中將 computer-use 與 function calling 結合。

<a id="q-why-did-openclaw-grow-faster-than-any-open-source-project-in-history-what-does-this-tell-you-about-the-market"></a>
### Q：為什麼 OpenClaw 成長速度比歷史上任何開源專案都快？這反映了什麼市場訊號？

**有力回答：**
有三個因素。(1) **零摩擦上手**：OpenClaw 可直接連接人們本來就在用的訊息平台（WhatsApp、Telegram）。使用者不需要學習新的介面。(2) **`SOUL.md` 個人化**：讓代理擁有自訂人格的能力，會帶來情感連結與病毒式傳播──人們會分享自己的代理。(3) **模型無關的架構**：使用者不會被綁在單一 LLM 供應商上，因此成本更低、彈性更高。這個市場訊號是：代理的「介面」比底層模型更重要。人們想要的是能出現在自己既有場景中的代理（訊息 App，而不是 Web UI）。另一面則是：若高速成長卻沒有同步投入安全建設，就會演變成 135,000 個暴露實例這類危機，這對任何開源代理專案都是警示。

<a id="q-compare-sandboxed-vs-unsandboxed-execution-for-ai-agents-when-would-you-choose-each"></a>
### Q：請比較 AI 代理的沙箱化與未沙箱化執行。你會在什麼情況選擇各自方案？

**有力回答：**
沙箱化（Docker/VM）：適合不受信任的程式碼執行、多租戶系統，或任何正式環境部署。OpenHands 在這方面做得很好──每個 session 都有自己的 Docker container。代價是建置複雜度與效能負擔。未沙箱化（直接存取宿主機）：只適合單一使用者、可信任、且有人盯著的環境。Open Interpreter 與 OpenClaw 為了最大能力採用了這種做法。風險在於錯誤的 LLM 輸出可能破壞宿主系統。2026 年的共識是預設沙箱化，並為進階使用者保留 escape hatch。在面試中，一定要指出 sandbox boundary 是安全決策，而不只是便利性決策。

---

<a id="references"></a>
## 參考資料

- OpenClaw GitHub Repository 與文件（2025–2026）
- OpenHands 文件與 SDK 參考（2025–2026）
- Open Interpreter GitHub Repository（2024–2026）
- Anthropic，《Computer Use Tool Documentation》（2024–2026）
- Anthropic，《Claude Code Overview》（2025–2026）
- MCP Specification 2025-11-25 與 2026 Roadmap
- Gartner，《AI Agent Adoption Projections》（2025–2026）
- Cursor、Windsurf 與 Cline 官方文件（2025–2026）

---

*下一篇：[Architecture Patterns for Tool-Use Agents](02-architecture-patterns.md)*
