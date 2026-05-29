<a id="opencoder-ai-coding-agents-landscape"></a>
# OpenCoder：AI Coding Agents 全景

AI coding agent 生態已經快速爆發。本指南涵蓋 open-weight coding model、agentic IDE、開源代理，以及如何為你的工程工作流程選擇合適工具。

<a id="table-of-contents"></a>
## 目錄

- [AI Coding 生態（2026）](#landscape)
- [Open-Weight Coding Models](#models)
- [AI-Native IDE](#ides)
- [開源程式編碼代理](#agents)
- [Benchmark 深入解析](#benchmarks)
- [成本比較](#costs)
- [選型指南](#selection)
- [正式環境架構](#production)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="landscape"></a>
## AI Coding 生態（2026）

Coding AI 生態可分成三個明確層次：

```
┌─────────────────────────────────────────────────────────────┐
│                    AI CODING STACK (2026)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LAYER 3: CODING AGENTS (Autonomous, multi-turn)           │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │  Claude Code │ │  OpenHands │ │  Cline / Aider     │   │
│  │  (Anthropic) │ │  (Open)    │ │  (Open)            │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
│  LAYER 2: AI IDEs (Completion + editing, developer-in-loop)│
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │    Cursor    │ │  Windsurf  │ │  GitHub Copilot    │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
│  LAYER 1: CODING MODELS (The brains behind everything)     │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │  Claude 3.7  │ │    o3      │ │ Qwen2.5-Coder-32B  │   │
│  │  GPT-4o      │ │ DeepSeek-R1│ │ StarCoder2-15B     │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

<a id="models"></a>
## Open-Weight Coding Models

這些模型可以自行託管、微調，並在不依賴任何 API 的情況下部署。

<a id="qwen25-coder-alibaba"></a>
### Qwen2.5-Coder（Alibaba）

截至 2026 年 3 月，這是最強的開源 coding model 家族：

| 模型 | 參數量 | 上下文 | HumanEval+ | 備註 |
|-------|------------|---------|------------|-------|
| Qwen2.5-Coder-32B-Instruct | 32B | 128K | 88.2% | 最佳開放式 coding model |
| Qwen2.5-Coder-7B-Instruct | 7B | 128K | 79.3% | 極佳的小型模型 |
| Qwen2.5-Coder-1.5B | 1.5B | 32K | 65.8% | Edge/on-device 使用 |

**優勢：**
- 在許多 coding benchmark 上可匹敵 GPT-4o
- 支援 100+ 種程式語言
- 具備優秀的 fill-in-the-middle（FIM）補全能力
- Apache 2.0 授權——可完整商業使用

```python
# Self-hosted with vLLM
from vllm import LLM

model = LLM(
    model="Qwen/Qwen2.5-Coder-32B-Instruct",
    tensor_parallel_size=2,  # 2× A100 80GB
)
response = model.generate("def fibonacci(n: int) -> list[int]:")
```

<a id="deepseek-coder-v2-deepseek"></a>
### DeepSeek-Coder-V2（DeepSeek）

| 模型 | 參數量 | 架構 | HumanEval+ |
|-------|------------|-------------|------------|
| DeepSeek-Coder-V2-Instruct | 236B (MoE) | MoE | 90.2% |
| DeepSeek-Coder-V2-Lite | 16B (MoE) | MoE | 81.1% |

**優勢：**
- MoE 架構 → 每個 token 僅啟用 21B 參數（高效率）
- 在競賽程式設計（CodeForces 題目）表現強
- 開放權重；中文支援強

<a id="starcoder2-bigcode-hugging-face"></a>
### StarCoder2（BigCode / Hugging Face）

| 模型 | 參數量 | 上下文 | 備註 |
|-------|------------|---------|-------|
| StarCoder2-15B | 15B | 16K | 最佳中型開源 coding LM |
| StarCoder2-7B | 7B | 16K | 高效率，支援 80+ 種語言 |
| StarCoder2-3B | 3B | 16K | 輕量，可 on-device |

**優勢：**
- 完全開放（BigCode OpenRAIL-M license）
- 非常適合 IDE 補全（低延遲）
- 在 Stack Overflow / GitHub 資料上表現強

<a id="deepseek-r1-distill-for-coding"></a>
### DeepSeek-R1-Distill（用於 coding）

| 模型 | 參數量 | 數學/程式碼 | 備註 |
|-------|------------|-----------|-------|
| DeepSeek-R1-Distill-Qwen-32B | 32B | 極佳 | 將推理能力蒸餾到較小模型 |
| DeepSeek-R1-Distill-Llama-8B | 8B | 不錯 | 小型推理模型 |

**使用情境**：當你需要具備推理品質的程式碼生成，同時又必須採用 self-hosted 規模化部署時。

<a id="open-model-selection-guide"></a>
### 開放模型選型指南

```
Simple completions (< 100ms latency needed)?
  → StarCoder2-3B or Qwen2.5-Coder-1.5B (local, fast)

Best quality self-hosted?
  → Qwen2.5-Coder-32B-Instruct (2× A100)

Budget < 1× A100 GPU?
  → Qwen2.5-Coder-7B-Instruct (1× RTX 4090 sufficient)

Need reasoning + coding?
  → DeepSeek-R1-Distill-Qwen-32B

Competitive programming / algorithmic?
  → DeepSeek-Coder-V2 or DeepSeek-R1
```

---

<a id="ides"></a>
## AI-Native IDE

<a id="cursor"></a>
### Cursor

**網站：** cursor.sh | **基底：** VS Code fork | **價格：** $20/月 Pro

Cursor 是目前領先的 AI-native IDE。核心能力如下：

| 功能 | 說明 |
|---------|-------------|
| **Composer** | 多檔案 agentic 編輯（相當於 Cursor 版的 Claude Code） |
| **Ctrl+K** | 行內程式碼生成 |
| **Tab** | 預測式補全（比 Copilot 更聰明） |
| **@-mentions** | 將檔案、URL、文件附加到上下文 |
| **.cursorrules** | 專案層級的 AI 指令（類似 CLAUDE.md） |
| **Model choice** | GPT-4o、Claude 3.7 Sonnet、o3、Gemini 2.0 Flash |

**最適合**：想在熟悉 GUI 內使用 agentic 編輯的 frontend/full-stack 開發者。

**限制**：Closed-source；你的程式碼會送到 Cursor 的伺服器（他們提供 Privacy Mode）。

<a id="windsurf-by-codeium"></a>
### Windsurf（by Codeium）

**網站：** codeium.com/windsurf | **基底：** VS Code fork | **價格：** 免費方案 + $15/月 Pro

Windsurf 的差異化特色在於 **Flows**（不要和 CrewAI Flows 混淆）：

| 功能 | 說明 |
|---------|-------------|
| **Cascade** | Windsurf 的 agentic 編輯模式 |
| **Flows** | 決定性的 agentic 序列（代理與使用者協作一致） |
| **Model choice** | 任意：GPT-4o、Claude 3.7、Gemini 2.0、DeepSeek |
| **Free tier** | 慷慨的免費額度 |

**最適合**：想要類似 Cursor 體驗、同時需要免費方案與模型彈性的團隊。

<a id="github-copilot-microsoft-openai"></a>
### GitHub Copilot（Microsoft/OpenAI）

| 功能 | 狀態（2026 年 3 月） |
|---------|---------------------|
| Completions | ✅ 依安裝基數仍是市場龍頭 |
| Copilot Workspace | ✅ 多檔案 agentic 編輯（已 GA） |
| Model | GPT-4o（預設）、Claude 3.5（可用） |
| Enterprise features | ✅ IP 保護、org policy、可關閉 code referencing |

**最適合**：已經在 Microsoft/GitHub 生態中的企業團隊。

**2026 年現實**：對多數開發者而言，Copilot 的補全品質已被 Cursor/Windsurf 超越，但其企業功能與 GitHub 整合仍使其在大型組織中保持主導地位。

---

<a id="agents"></a>
## 開源程式編碼代理

<a id="openhands-formerly-opendevin"></a>
### OpenHands（前身為 OpenDevin）

**GitHub：** github.com/All-Hands-AI/OpenHands | **授權：** MIT

目前領先的開源自主 coding agent：

```bash
# Run with Docker
docker pull docker.all-hands.dev/all-hands-ai/openhands:latest
docker run -it --rm \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:latest \
  -e LLM_API_KEY=$ANTHROPIC_API_KEY \
  -e LLM_MODEL=claude-3-7-sonnet-20250219 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 3000:3000 \
  docker.all-hands.dev/all-hands-ai/openhands:latest
# Access at http://localhost:3000
```

**架構：**
```
User request
    ↓
OpenHands Controller
    ├── CodeActAgent (main strategy)
    ├── Docker Sandbox (isolated execution)
    ├── File editor (str_replace_editor)
    └── Browser (playwright for web tasks)
```

**核心特性：**
- **Any LLM**：可搭配 Claude 3.7、GPT-4o、Gemini、local Ollama
- **Docker sandbox**：代理在隔離 container 中執行
- **Web UI**：聊天式介面；會顯示代理的推理過程
- **API access**：提供用於 CI 整合的 REST API
- **SWE-bench 分數**：~55-60%（依後端模型而定）

<a id="aider"></a>
### Aider

**GitHub：** github.com/paul-gauthier/aider | **授權：** Apache 2.0

以終端機為優先、git-native 的 coding agent：

```bash
pip install aider-chat

# Works directly with your git repo
aider --model claude-3-7-sonnet-20250219

# Add files to context
/add src/auth.py src/models.py

# Give task
> Add JWT authentication to the User model
```

**Aider 的差異化之處：**
- **Git-native**：在進行過程中持續提交變更；維持乾淨的 git 歷史
- **Context maps**：維護整個程式碼庫的地圖（即使檔案不在當前上下文中）
- **Voice mode**：可用語音說出任務  
- **Architecture mode**：在動手改程式前先討論設計

```bash
# Benchmark (March 2026)
# Aider + claude-3-7-sonnet → SWE-bench Verified: ~55%
# Aider + o3 → SWE-bench Verified: ~60%
```

<a id="cline-vs-code-extension"></a>
### Cline（VS Code Extension）

**GitHub：** github.com/cline/cline | **授權：** Apache 2.0

適用於自主程式編碼的開源 VS Code extension：

```
VS Code
  └── Cline Extension
        ├── Any model (Claude, GPT, Gemini, Ollama)
        ├── File system access (read/write any file)
        ├── Terminal (bash commands)
        ├── Browser (playwright)
        └── MCP servers (any MCP tool)
```

**關鍵差異點：**
- **MCP-native**：開箱即用的完整 MCP 支援
- **逐動作授權**：每個 shell 命令與檔案編輯都需要使用者批准
- **模型彈性**：支援任何 OpenAI-compatible API endpoint（包含 local Ollama）
- **免費**：開源，無訂閱費

**最適合**：想免費獲得類似 Cursor 體驗，且需要完整模型彈性的開發者。

---

<a id="benchmarks"></a>
## Benchmark 深入解析

<a id="swe-bench-verified-march-2026"></a>
### SWE-bench Verified（2026 年 3 月）

這是 agentic software engineering 的黃金標準，用來衡量解決真實 GitHub issue 的能力。

| 代理 / 系統 | 分數 | 後端模型 | 備註 |
|---------------|-------|---------------|-------|
| Devin 2.0（商業版） | 55-65% | Claude 3.7 | 付費服務 |
| Claude Code | ~70% | Claude 3.7 Sonnet | Anthropic 官方 |
| OpenHands（最佳設定） | ~55% | Claude 3.7 Sonnet | 開源 |
| Aider | ~55% | o3 / Claude 3.7 | 開源 CLI |
| SWE-agent | ~38% | GPT-4o | Princeton 研究 |

> [!NOTE]
> SWE-bench 分數對後端模型極度敏感。相同代理搭配 claude-3-7-sonnet 時，通常會比搭配 GPT-4o 高出 10-15%。

<a id="humaneval-open-models"></a>
### HumanEval+（開放模型）

| 模型 | HumanEval+ 分數 |
|-------|-----------------|
| Claude 3.7 Sonnet | 93.6% |
| GPT-4o | 90.2% |
| Qwen2.5-Coder-32B-Instruct | 88.2% |
| DeepSeek-Coder-V2-Instruct | 90.2% |
| StarCoder2-15B | 73.3% |

<a id="livecodebench-runtime-evaluation-stronger-signal"></a>
### LiveCodeBench（執行期評估，訊號更強）

LiveCodeBench 使用全新的競賽程式設計題目（不在訓練資料中）：

| 模型 | LiveCodeBench 分數 |
|-------|---------------------|
| o3（high） | 68.1% |
| Claude 3.7 Sonnet | 54.2% |
| GPT-4.5 | 38.7% |
| Qwen2.5-Coder-32B | 43.2% |
| DeepSeek-R1 | 57.0% |

**洞察**：LiveCodeBench 分數遠低於 HumanEval，因為它測試的是新穎問題。o3 與 DeepSeek-R1 因其推理能力而領先。

---

<a id="costs"></a>
## 成本比較

<a id="closed-api-vs-open-self-hosted"></a>
### Closed API vs. Open Self-Hosted

**情境：每天 1,000 個 coding 任務，每個任務平均 5K tokens**

| 方案 | 每月成本 | 品質 | 延遲 |
|----------|-------------|---------|---------|
| Claude 3.7 Sonnet（API） | ~$9,000 | ★★★★★ | 中等 |
| GPT-4o（API） | ~$7,500 | ★★★★ | 中等 |
| o3-mini（API） | ~$3,300 | ★★★★★（推理） | 慢 |
| Qwen2.5-Coder-32B（4×A100） | ~$4,000（infra） | ★★★★ | 快 |
| DeepSeek-V3（Together AI） | ~$1,350 | ★★★★ | 中等 |

**關鍵洞察**：相較於 Claude API，self-hosting Qwen2.5-Coder-32B 在每天約 500+ 個任務時就開始具備成本競爭力。若少於每天 200 個任務，把工程維運成本算進去後，API 幾乎總是更便宜。

---

<a id="selection"></a>
## 選型指南

<a id="quick-decision-tree"></a>
### 快速決策樹

```
What is your primary need?

├─ IDE coding assistance (completions + chat)?
│  ├─ Microsoft ecosystem / enterprise? → GitHub Copilot
│  ├─ Want best quality? → Cursor (Pro)
│  └─ Want free + model choice? → Windsurf or Cline
│
├─ Autonomous agent for standalone coding tasks?
│  ├─ Best quality, don't mind proprietary? → Claude Code
│  ├─ Need open-source? → OpenHands
│  ├─ CLI-first, git-native? → Aider
│  └─ VS Code embedded, MCP-native? → Cline
│
├─ Self-hosted model for custom deployment?
│  ├─ Best quality? → Qwen2.5-Coder-32B
│  ├─ Need reasoning? → DeepSeek-R1-Distill-32B
│  ├─ Fast completions? → Qwen2.5-Coder-7B or StarCoder2-7B
│  └─ Edge/on-device? → Qwen2.5-Coder-1.5B or StarCoder2-3B
│
└─ CI/CD pipeline integration?
   ├─ Best results? → Claude Code SDK (headless)
   ├─ Open-source? → OpenHands REST API
   └─ Git-native? → Aider CLI in GitHub Actions
```

<a id="comparison-matrix"></a>
### 比較矩陣

| 維度 | Claude Code | Cursor | OpenHands | Aider | Cline |
|-----------|-------------|--------|-----------|-------|-------|
| 自主性 | 完整 | 中等 | 完整 | 完整 | 完整 |
| 模型鎖定 | Claude | 任意 | 任意 | 任意 | 任意 |
| 開源 | ❌ | ❌ | ✅ | ✅ | ✅ |
| CI/Headless | ✅ | ❌ | ✅ | ✅ | ❌ |
| GUI | CLI | 完整 IDE | Web UI | Terminal | VS Code |
| MCP | ✅ | ✅ | 部分 | ❌ | ✅ |
| Git-native | 部分 | 部分 | ✅ | ✅ | 部分 |
| 價格 | API 成本 | $20/月 | 免費 + API | 免費 + API | 免費 + API |

---

<a id="production"></a>
## 正式環境架構

<a id="enterprise-coding-agent-platform"></a>
### 企業級 Coding Agent 平台

以下是建置內部 AI coding 平台的方法：

```
┌────────────────────────────────────────────────────────────┐
│             ENTERPRISE CODING AGENT PLATFORM                │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Developer                                                 │
│     ↓ (Jira ticket / PR description)                      │
│  ┌──────────────────────────────────┐                      │
│  │        TASK INTAKE LAYER         │                      │
│  │  • Parse task from Jira/GitHub   │                      │
│  │  • Classify: simple/complex      │                      │
│  │  • Route to appropriate agent    │                      │
│  └──────────────┬───────────────────┘                      │
│                 │                                          │
│    Simple fix   │   Complex feature                        │
│        ↓        │        ↓                                 │
│  ┌──────────┐   │  ┌──────────────────┐                    │
│  │  Aider   │   │  │   Claude Code    │                    │
│  │ (cheap)  │   └→ │  SDK (headless)  │                    │
│  └────┬─────┘      └────────┬─────────┘                    │
│       │                     │                              │
│       └─────────────────────┘                              │
│                 ↓                                          │
│  ┌──────────────────────────────────┐                      │
│  │         REVIEW LAYER             │                      │
│  │  • Git diff → PR creation        │                      │
│  │  • Auto-run CI tests             │                      │
│  │  • Human review (required)       │                      │
│  └──────────────────────────────────┘                      │
│                 ↓                                          │
│         Merge to main (human approved)                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

<a id="key-production-decisions"></a>
### 關鍵正式環境決策

| 決策 | 選項 | 建議 |
|----------|---------|----------------|
| 代理使用的模型 | Claude 3.7、GPT-4o、open model | Claude 3.7 Sonnet 以獲得最佳結果 |
| 任務入口 | 手動、Jira webhook、GitHub label | 用 GitHub label 觸發 Actions workflow |
| 程式碼執行 | 本機、Docker、E2B | Docker（可重現、隔離） |
| 人工審查 | PR、Slack 核准、自動化 | 必須 PR review，絕不 auto-merge |
| 成本控制 | Max turns、model routing | `max_turns=20`，簡單任務用 Haiku |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-choose-between-claude-code-cursor-and-openhands"></a>
### Q：你如何在 Claude Code、Cursor 與 OpenHands 之間做選擇？

**強答案：**
這取決於三個軸線：

1. **介面需求**：如果開發者想要 GUI（可在上下文中看到變更），用 Cursor 或 Windsurf。若任務是腳本化/無頭執行（CI 中的 bug 修復、測試生成），就用 Claude Code SDK 或 OpenHands。

2. **模型控制權**：如果你需要使用任意模型（或自家微調模型），就用 OpenHands 或 Aider。如果你能接受僅用 Anthropic，且想要同級最佳結果，就用 Claude Code。

3. **開源需求**：企業資安團隊常要求可稽核的開源工具。OpenHands（MIT）與 Aider（Apache 2.0）就是答案。

對典型的新創團隊，我的建議是：日常開發用 Cursor、批次任務（由 GitHub issue 產生 PR）用 Claude Code，而 self-hosted 的 CI pipeline 則用 OpenHands。

<a id="q-why-are-open-weight-coding-models-like-qwen25-coder-important-for-enterprise"></a>
### Q：為什麼像 Qwen2.5-Coder 這類 open-weight coding model 對企業很重要？

**強答案：**
三個原因：

1. **資料隱私**：送到 closed API 的程式碼，可能被用於訓練或暴露給第三方。對醫療（HIPAA）、金融（SOX）與政府團隊而言，專有程式碼不能離開內網。把 Qwen2.5-Coder-32B 跑在 on-prem 就能解決這個問題。

2. **規模化成本**：當每月有 100 萬次以上程式碼生成請求時，self-hosting 會比 API 計價便宜 40-60%，尤其是在補全（相對於 agentic 任務）場景。

3. **微調能力**：開放權重可以做領域特化。法律科技公司可以針對內部 DSL（domain-specific language）做微調，API 則做不到。

Qwen2.5-Coder-32B 與 Claude 3.7 Sonnet 之間仍有品質差距，但正在縮小。對補全與較簡單的任務來說，open model 通常已經「夠用」。

<a id="q-how-would-you-design-the-testing-strategy-for-an-ai-coding-agent-in-ci"></a>
### Q：你會如何為 CI 中的 AI coding agent 設計測試策略？

**強答案：**
我會採用三層評估：

**1. 功能測試**（自動化、每次執行都跑）：
```
Agent output → Run pytest → Pass rate metric
```

**2. 與 ground truth 比對**（每週）：
```
Known bug → Agent fix → Compare to expert fix
Metric: Semantic similarity of diff (not byte-exact)
```

**3. 人工評估**（抽樣 5% 的 agent PR）：
```
Senior engineer rates: Correctness, Style, Safety, 1-5 scale
```

我也會追蹤**回歸率**——如果代理修復造成新的測試失敗，那就是硬性失敗。代理應執行完整測試套件，且只有在通過率提升或至少維持不變時才算成功。

---

<a id="references"></a>
## 參考資料

- Qwen2.5-Coder: https://qwenlm.github.io/blog/qwen2.5-coder/
- DeepSeek-Coder-V2: https://github.com/deepseek-ai/DeepSeek-Coder-V2
- StarCoder2: https://huggingface.co/blog/starcoder2
- OpenHands: https://github.com/All-Hands-AI/OpenHands
- Aider: https://aider.chat/
- Cline: https://github.com/cline/cline
- Cursor: https://cursor.sh/
- Windsurf: https://codeium.com/windsurf
- SWE-bench Leaderboard: https://www.swebench.com/
- LiveCodeBench: https://livecodebench.github.io/

---

*上一篇：[Claude Code](09-claude-code.md) | 下一篇：[Framework Selection Guide](08-framework-selection-guide.md)*
