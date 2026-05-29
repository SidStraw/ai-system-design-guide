<a id="openclaw-deep-dive-the-open-source-personal-ai-agent"></a>
# OpenClaw 深入解析：開源個人 AI Agent

OpenClaw 是一個**開源、自行託管的個人 AI agent**，透過訊息平台作為主要介面，使用 LLM 來執行任務。你可以透過 WhatsApp、Telegram、Slack、Discord 或 Signal 與它對話，而它會回應你——執行 shell 指令、控制瀏覽器、管理行事曆、處理電子郵件，以及編排多步驟工作流程。

<a id="table-of-contents"></a>
## 目錄

- [什麼是 OpenClaw](#what-is-openclaw)
- [歷史：從 Clawdbot 到 Moltbot 再到 OpenClaw](#history)
- [架構深入解析](#architecture)
- [AgentSkills 系統](#agentskills)
- [LLM 供應商設定](#llm-providers)
- [訊息平台整合](#messaging-integrations)
- [安全模型](#security-model)
- [部署模式](#deployment-patterns)
- [效能最佳化與擴展](#performance)
- [真實世界使用案例](#use-cases)
- [限制，以及何時**不要**使用 OpenClaw](#limitations)
- [與替代方案的比較](#comparison)
- [快速上手：快速設定指南](#getting-started)
- [系統設計面試切入角度](#system-design-interview)
- [參考資料](#references)

---

<a id="what-is-openclaw"></a>
## 什麼是 OpenClaw

OpenClaw 是：

- **個人 AI agent**：不是 chatbot——而是會代表你採取行動的自主 agent
- **自行託管**：可在你的機器、VPS 或 Raspberry Pi 上執行——你的資料由你掌控
- **訊息原生**：存在於你已經在使用的聊天 App 中（WhatsApp、Telegram、Slack、Discord、Signal、iMessage，以及其他 20+ 個平台）
- **LLM 無關**：可搭配 Claude、GPT-4、Gemini、DeepSeek 或本地模型使用
- **技能可擴充**：提供 100+ 個預先設定的 skills，也有簡單格式可撰寫自訂 skill
- **開源**：採 MIT 授權，截至 2026 年初 GitHub 星數超過 25 萬

```
# The simplest way to start
git clone https://github.com/openclaw/openclaw.git
cd openclaw
docker compose up -d

# Or via npm
npm install -g openclaw
openclaw start
```

**它與 chatbot 的關鍵差異：**
- ChatGPT/Claude.ai：你輸入，它回覆文字
- OpenClaw：你輸入，它會**真的做事**——執行指令、編輯檔案、寄送電子郵件、控制智慧家庭裝置、管理你的行事曆

---

<a id="history"></a>
## 歷史

<a id="the-naming-timeline"></a>
### 命名時間線

| 日期 | 名稱 | 事件 |
|------|------|-------|
| 2025 年 11 月 | **Clawdbot** | Peter Steinberger 發布第一個原型，約一小時內完成 |
| 2026 年 1 月 | 2,000 stars | 早期採用者發現了這個專案 |
| 2026 年 1 月 27 日 | **Moltbot** | 因 Anthropic 商標投訴而重新命名（保留龍蝦主題） |
| 2026 年 1 月 30 日 | **OpenClaw** | 再次更名——Steinberger 覺得「Moltbot」念起來很拗口 |
| 2026 年 2 月 | 145,000+ stars | 爆炸性成長，超越許多成熟的開源專案 |
| 2026 年 2 月 14 日 | -- | Steinberger 加入 OpenAI，表示這讓他能取得擴展所需的資源 |
| 2026 年 3 月 | 250,000+ stars | 在 GitHub 上超越 React；成為有史以來成長最快的 OSS 專案之一 |

<a id="the-creator"></a>
### 創作者

Peter Steinberger 是來自奧地利的軟體工程師，在 2024 年出售公司之前，他曾花 13 年打造 PSPDFKit——一套被全球開發者使用的 PDF 工具組。他形容自己是「vibe coder」，也以「我會發布自己沒讀過的程式碼」這句名言聞名——這體現了新的 AI-first 開發哲學：人類提供意圖，AI 提供實作。

<a id="why-it-went-viral"></a>
### 為什麼會爆紅

OpenClaw 之所以打中市場痛點，是因為它解決了一個真實問題：LLM 很強大，但沒有狀態。每一次對話都從零開始。OpenClaw 為 LLM 帶來了**持久性**（跨 session 記憶）、**能動性**（不只會說，還能做事），以及**觸及範圍**（整合你已在使用的 App）。此外，它是自行託管且開源，代表任何人都能在不把資料託付給第三方服務的情況下自行運行。

---

<a id="architecture"></a>
## 架構

<a id="high-level-overview"></a>
### 高階概觀

```
                         OPENCLAW ARCHITECTURE
 ============================================================

  Messaging Platforms              OpenClaw Gateway           LLM Providers
 ┌──────────────┐              ┌─────────────────────┐     ┌──────────────┐
 │  WhatsApp    │──┐           │                     │     │  Anthropic   │
 │  (Baileys)   │  │           │   GATEWAY            │     │  (Claude)    │
 ├──────────────┤  │  Channel  │   ┌──────────────┐  │     ├──────────────┤
 │  Telegram    │──┼──Adapters─┼──>│  Router      │  │     │  OpenAI      │
 │  (grammY)    │  │           │   │  (sessions,  │  │     │  (GPT-4)     │
 ├──────────────┤  │           │   │   bindings)  │  │     ├──────────────┤
 │  Slack       │──┤           │   └──────┬───────┘  │     │  Google      │
 │  (Bolt)      │  │           │          │          │     │  (Gemini)    │
 ├──────────────┤  │           │   ┌──────▼───────┐  │     ├──────────────┤
 │  Discord     │──┤           │   │ Agent Runtime│──┼────>│  DeepSeek    │
 │  (discord.js)│  │           │   │ (AI loop,    │  │     ├──────────────┤
 ├──────────────┤  │           │   │  tool calls, │  │     │  Local/      │
 │  Signal      │──┤           │   │  memory)     │  │     │  Ollama      │
 │  (signal-cli)│  │           │   └──────┬───────┘  │     └──────────────┘
 ├──────────────┤  │           │          │          │
 │  iMessage    │──┤           │   ┌──────▼───────┐  │     Tools & Skills
 │  (BlueBubbles│  │           │   │  Tool Layer  │  │     ┌──────────────┐
 ├──────────────┤  │           │   │  (skills,    │──┼────>│  Shell exec  │
 │  Teams       │──┘           │   │   browser,   │  │     │  Browser     │
 │  IRC, Matrix │              │   │   files,     │  │     │  File I/O    │
 │  20+ more... │              │   │   cron)      │  │     │  Calendar    │
 └──────────────┘              │   └──────────────┘  │     │  Email       │
                               │                     │     │  100+ more   │
                               │   ┌──────────────┐  │     └──────────────┘
                               │   │  Memory &    │  │
                               │   │  State       │  │     Storage
                               │   │  (sessions,  │──┼────>┌──────────────┐
                               │   │   workspace) │  │     │  ~/.openclaw/│
                               │   └──────────────┘  │     │  (state,     │
                               └─────────────────────┘     │   memory,    │
                                                           │   config)    │
                                localhost:18789             └──────────────┘
```

<a id="core-components"></a>
### 核心元件

**1. Gateway**

Gateway 是一個長時間執行的 WebSocket 伺服器（預設：`localhost:18789`），作為 sessions、routing 與 channel connections 的單一事實來源（single source of truth）。它負責：

- 透過 channel adapters 接受所有訊息平台的連線
- 將訊息路由到正確的 agent
- session 管理與狀態持久化
- 驗證與存取控制
- 熱重載設定變更

**2. Channel Adapters**

當任一平台傳來訊息時，channel adapter 會先將其正規化為標準內部格式。每個 adapter 都包裝了一個平台專用函式庫：

| 平台 | Adapter 函式庫 | 協定 |
|----------|----------------|----------|
| WhatsApp | Baileys | WebSocket（非官方） |
| Telegram | grammY | Bot API |
| Slack | Bolt | Events API |
| Discord | discord.js | Gateway API |
| Signal | signal-cli | D-Bus |
| iMessage | BlueBubbles | REST API |
| IRC | irc-framework | IRC 協定 |
| Matrix | matrix-js-sdk | Matrix 協定 |
| Microsoft Teams | Bot Framework | REST API |

**3. Agent Runtime**

Agent Runtime 就是 AI loop。對於每一則傳入訊息，它會：

1. 從 session 歷史、workspace 記憶與相關 skills 組裝 context
2. 將組裝好的 prompt 傳送給已設定的 LLM
3. 從模型接收 tool calls
4. 對系統能力執行 tool calls
5. 把結果回傳給模型進行下一輪迭代
6. 持久化更新後的狀態（記憶、檔案、session 歷史）

**4. Multi-Agent Routing**

OpenClaw 支援在單一 Gateway process 內執行多個 agents。每個 agent 都有自己的 workspace、agentDir、sessions 與 tool 設定。傳入訊息會透過 bindings 被路由到 agents：

```json
{
  "agents": {
    "list": [
      {
        "name": "work-assistant",
        "agentDir": "./agents/work",
        "channels": ["slack-work"]
      },
      {
        "name": "home-assistant",
        "agentDir": "./agents/home",
        "channels": ["whatsapp-personal", "telegram"]
      },
      {
        "name": "devops-bot",
        "agentDir": "./agents/devops",
        "channels": ["discord-infra"]
      }
    ]
  }
}
```

這代表你可以讓工作助理跑在 Slack、個人助理跑在 WhatsApp，而 DevOps bot 跑在 Discord——全都由同一個 Gateway 驅動，且記憶與權限彼此完全隔離。

---

<a id="the-agentskills-system"></a>
<a id="agentskills"></a>
## AgentSkills 系統

<a id="how-skills-work"></a>
### Skills 如何運作

Skills 是 OpenClaw 取得基本對話能力以外其他能力的機制。每個 skill 都是一個目錄，其中包含 `SKILL.md` 檔案，裡面有 YAML frontmatter（中繼資料）與 markdown instructions（行為說明）。

```
~/.openclaw/skills/
  weather/
    SKILL.md           # Required: metadata + instructions
    scripts/
      fetch_weather.py # Optional: executable scripts
    references/
      api_docs.md      # Optional: supplementary docs

  email-manager/
    SKILL.md
    scripts/
      process_inbox.py
```

<a id="skillmd-format"></a>
### SKILL.md 格式

```yaml
---
name: weather-lookup
description: >
  Fetch current weather and forecasts for any location.
  Responds to queries about temperature, rain, and conditions.
triggers:
  - weather
  - temperature
  - forecast
  - "is it going to rain"
tools:
  - web_search
  - bash
---

# Weather Lookup Skill

When the user asks about weather:

1. Use the web_search tool to find current conditions
2. Extract temperature, humidity, wind, and forecast
3. Present in a concise, readable format
4. Include both metric and imperial units

## Example Response Format

"Currently 72F (22C) and partly cloudy in San Francisco.
Forecast: Clear skies through Thursday, rain expected Friday."
```

<a id="skill-resolution-order"></a>
### Skill 解析順序

Skills 可以存在於多個位置。當名稱衝突時，越本地的副本優先：

```
Priority (highest first):
  1. <workspace>/skills/        # Project-specific skills
  2. ~/.openclaw/skills/        # User-global skills
  3. <installed-packages>/      # npm-installed skills
  4. <bundled>/skills/          # Ships with OpenClaw
```

<a id="selective-injection"></a>
### 選擇性注入

OpenClaw **不會**把每個 skill 都注入到每個 prompt 中。Runtime 只會根據 skill 描述與 trigger keywords，選擇性注入與目前這一輪相關的 skills。這能避免 prompt 膨脹，並維持良好的模型效能。

<a id="creating-a-custom-skill"></a>
### 建立自訂 Skill

```bash
# Create the skill directory
mkdir -p ~/.openclaw/skills/deploy-checker
cd ~/.openclaw/skills/deploy-checker

# Create the SKILL.md
cat > SKILL.md << 'EOF'
---
name: deploy-checker
description: >
  Monitor deployment status across staging and production.
  Checks health endpoints, recent commits, and CI status.
triggers:
  - deploy
  - deployment
  - "is staging up"
  - "prod status"
tools:
  - bash
  - web_search
---

# Deploy Checker

When asked about deployment status:

1. Run `curl -s https://staging.myapp.com/health` to check staging
2. Run `curl -s https://myapp.com/health` to check production
3. Check recent git log: `git log --oneline -5`
4. Report status in a clear format

## Response Format

Staging: [UP/DOWN] - version X.Y.Z - deployed 2h ago
Production: [UP/DOWN] - version X.Y.Z - deployed 1d ago
Last 3 commits: ...
