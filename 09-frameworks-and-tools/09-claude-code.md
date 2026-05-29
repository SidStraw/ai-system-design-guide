<a id="claude-code-the-autonomous-coding-agent"></a>
# Claude Code：自主式程式編碼代理

Claude Code 是 Anthropic 的**終端機原生自主程式編碼代理**。不同於只會提供程式碼補全建議的 IDE 外掛，Claude Code 更像是一位全端軟體工程師：它會讀取你的程式碼庫、編輯檔案、執行命令、跑測試，並持續迭代直到任務完成。

<a id="table-of-contents"></a>
## 目錄

- [Claude Code 是什麼](#what-it-is)
- [核心架構](#architecture)
- [核心工具](#tools)
- [CLAUDE.md 清單模式](#claude-md)
- [執行 Claude Code](#running)
- [子代理與平行化](#subagents)
- [自訂 MCP 整合](#mcp-integration)
- [安全與權限模型](#safety)
- [正式環境用法：CI Pipeline](#production)
- [比較：Claude Code 與替代方案](#comparison)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="what-it-is"></a>
## Claude Code 是什麼

Claude Code 由 Anthropic 於 2025 年初發布，其定位如下：

- **CLI 工具**：終端機中的 `claude` 指令
- **MCP-native 代理**：使用 bash、text_editor 與 computer 工具
- **SDK**：可嵌入 Python/TypeScript 應用程式
- **不只是聊天機器人**：它會自主規劃、實作與驗證

```
# Install
pip install claude-code  # or: npm install -g @anthropic-ai/claude-code

# Run interactively
claude

# Run headlessly (for CI)
claude -p "Add unit tests for all functions in src/utils.py" --output-format json
```

**與 Copilot/Cursor 的關鍵差異：**
- Copilot/Cursor：提供建議程式碼，由你接受或拒絕
- Claude Code：**自主完成整個任務**，並透過執行測試來驗證

---

<a id="architecture"></a>
## 核心架構

```
┌─────────────────────────────────────────────────────────┐
│                   CLAUDE CODE ARCHITECTURE               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User Request                                           │
│       ↓                                                 │
│  ┌─────────────┐    ┌──────────────┐                   │
│  │  Claude 3.7 │    │  CLAUDE.md   │                   │
│  │   Sonnet    │ ←  │  (manifest)  │                   │
│  │ (Extended   │    └──────────────┘                   │
│  │  Thinking)  │                                       │
│  └──────┬──────┘                                       │
│         │ Tool calls                                    │
│         ↓                                               │
│  ┌──────────────────────────────────────┐              │
│  │           TOOL LAYER                 │              │
│  │  ┌─────────┐ ┌───────────┐ ┌──────┐ │              │
│  │  │  bash   │ │text_editor│ │  MCP │ │              │
│  │  └────┬────┘ └─────┬─────┘ └──┬───┘ │              │
│  └───────┼────────────┼──────────┼─────┘              │
│          │            │          │                      │
│   Shell cmds     File edits    Custom tools             │
│   (test, lint,   (read/write)  (DB, APIs,               │
│    git, build)                  internal)               │
└─────────────────────────────────────────────────────────┘
```

Claude Code 以 **Claude 3.7 Sonnet** 作為核心模型，且在複雜規劃任務中預設啟用 Extended Thinking。

---

<a id="tools"></a>
## 核心工具

Claude Code 內建三個原生工具，並支援自訂 MCP 工具：

<a id="1-bash-shell-execution"></a>
### 1. `bash` — Shell 執行

```python
# Claude calls this internally:
bash(command="pytest tests/ -v --tb=short", timeout=60)
# Returns: stdout, stderr, exit_code
```

**Claude 會用它來：**
- 執行測試套件（`pytest`、`jest`、`cargo test`）
- 進行 Git 操作（`git diff`、`git commit`、`git log`）
- 執行建置命令（`npm build`、`make`、`docker build`）
- 安裝套件（`pip install`、`npm install`）

bash 工作階段在**跨回合之間會持續保留**——環境變數與工作目錄會在同一個 session 中延續。

<a id="2-text-editor-file-operations"></a>
### 2. `text_editor` — 檔案操作

```python
# Read a file
text_editor(command="view", path="/project/src/auth.py")

# Find in file
text_editor(command="view", path="/project/src/auth.py", view_range=[1, 50])

# Edit (surgical replacement)
text_editor(
    command="str_replace",
    path="/project/src/auth.py",
    old_str="def authenticate(user, password):",
    new_str="def authenticate(user: str, password: str) -> AuthResult:"
)

# Create new file
text_editor(command="create", path="/project/tests/test_auth.py", file_text="...")
```

**為什麼精準替換優於整段重寫：**
- 保留檔案上下文
- 降低幻覺風險（只改動需要變更的部分）
- 讓 diff 更原子化且易於審查

<a id="3-computer-gui-automation-optional"></a>
### 3. `computer` — GUI 自動化（選用）

提供完整桌面控制能力（截圖、滑鼠、鍵盤）——用於瀏覽器測試與 UI 驗證。需要沙箱化環境。

---

<a id="claude-md"></a>
## CLAUDE.md 清單模式

`CLAUDE.md` 檔案是高效使用 Claude Code 時**最重要的模式**。它會把持久性的專案上下文注入到每一次 Claude Code session 中。

```markdown
# CLAUDE.md — Project: E-Commerce API

## Architecture
- Python 3.11 FastAPI backend
- PostgreSQL 15 with Alembic migrations
- Redis for session caching
- All API responses must be Pydantic models

## Test Commands
- Run all tests: `pytest tests/ -v`
- Run single test: `pytest tests/test_auth.py::test_login -v`
- Lint: `ruff check . --fix`
- Type check: `mypy src/`

## Coding Standards
- Always add type hints
- Never use `global` variables
- All database queries through SQLAlchemy ORM, never raw SQL
- New features require tests with >80% coverage

## Forbidden Patterns
- Do NOT use `os.system()` — use `subprocess.run()` instead
- Do NOT commit secrets — use environment variables
- Do NOT modify `alembic/versions/` — create new migrations

## Architecture Decisions
- Auth: JWT tokens, 1hr expiry, refresh token pattern
- Errors: Always return RFC 7807 Problem Details format
- Logging: structlog with JSON output, always include request_id
```

**巢狀 CLAUDE.md 檔案：**
```
project/
  CLAUDE.md          # global project rules
  src/
    auth/
      CLAUDE.md      # auth-specific rules (stricter security)
    payments/
      CLAUDE.md      # payment-specific rules (PCI compliance notes)
```

Claude 會在某個目錄內工作時，自動讀取最近的 CLAUDE.md。

---

<a id="running"></a>
## 執行 Claude Code

<a id="interactive-mode"></a>
### 互動模式

```bash
# Start session (reads CLAUDE.md automatically)
claude

# With specific model
claude --model claude-3-7-sonnet-20250219

# With MCP config
claude --mcp-config .claude/mcp.json
```

<a id="headless-mode-for-scripting"></a>
### 無頭模式（用於腳本）

```bash
# Single task, JSON output
claude -p "Fix all type errors in src/" \
  --output-format json \
  --max-turns 20

# Pipe from file
echo "Refactor src/utils.py to use async/await" | claude -p -

# Stream output
claude -p "Add logging to all API endpoints" --output-format stream-json
```

<a id="python-sdk"></a>
### Python SDK

```python
import asyncio
from claude_code_sdk import query, ClaudeCodeOptions

async def run_coding_task(task: str) -> str:
    options = ClaudeCodeOptions(
        max_turns=30,
        allowed_tools=["bash", "str_replace_based_edit_tool"],
        system_prompt_suffix="Always run tests after making changes.",
    )
    
    messages = []
    async for message in query(prompt=task, options=options):
        messages.append(message)
    
    return messages[-1].content[0].text

result = asyncio.run(run_coding_task(
    "Add input validation to all POST endpoints in src/api/"
))
```

---

<a id="subagents"></a>
## 子代理與平行化

Claude Code 支援在大型程式碼庫中**派發子代理**：

```
Main Claude Code session
    ↓
"This codebase has 5 modules. I'll spawn sub-agents for each."
    ├── Sub-agent 1: Fix auth module tests
    ├── Sub-agent 2: Add type hints to utils/
    ├── Sub-agent 3: Migrate payments to async
    └── Sub-agent 4: Update API documentation
```

每個子代理會平行執行，之後再由主代理審查並合併結果。

**適合使用子代理的時機：**
- 程式碼庫超過 50K 行
- 可平行進行的獨立變更（沒有共享狀態）
- 以模組為單位的重構任務

---

<a id="mcp-integration"></a>
## 自訂 MCP 整合

Claude Code 會從 `~/.claude/config.json` 或 `.claude/mcp.json` 讀取 MCP server：

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"],
      "description": "Live library documentation"
    },
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres"],
      "env": {"DATABASE_URL": "postgresql://localhost/myapp"},
      "description": "Direct DB access for schema inspection"
    },
    "jira": {
      "command": "uvx",
      "args": ["mcp-server-jira"],
      "env": {"JIRA_URL": "https://company.atlassian.net"},
      "description": "Task tracking integration"
    }
  }
}
```

有了這份設定後，Claude Code 可以：
1. 在寫程式前先查詢最新的函式庫文件（Context7）
2. 在撰寫 SQL 前先讀取真實的 DB schema（postgres MCP）
3. 在完成實作後把 Jira ticket 標記為完成（jira MCP）

---

<a id="safety"></a>
## 安全與權限模型

Claude Code 採用**分層式權限模型**：

```
Permission Level    Who approves       What it covers
────────────────────────────────────────────────────────
Auto               Claude (no prompt)  Read files, run tests
Ask per-turn       User confirms       Shell command execution
Explicit allow     User pre-approves   Specific commands/dirs
Blocked            Never runs          Network calls outside allowlist
```

<a id="configuration"></a>
### 設定

```json
{
  "permissions": {
    "allow": [
      "bash(pytest*)",           // Always allow test runs
      "bash(ruff*)",             // Always allow linting
      "bash(git diff*)",         // Always allow git reads
      "str_replace_based_edit_tool"  // Always allow file edits
    ],
    "deny": [
      "bash(rm -rf*)",           // Block destructive deletions
      "bash(curl https://external*)", // Block external network
      "bash(pip install*)"       // Block package installs without approval
    ]
  }
}
```

<a id="production-safety-rules"></a>
### 正式環境安全規則

1. **務必使用沙箱**：在 Docker container 或 E2B cloud VM 中執行
2. **Git 隔離**：開始前先建立 feature branch；合併前先審查 diff
3. **人工檢查點**：正式環境部署必須由人工審查最終 diff
4. **Secret 掃描**：對每個 Claude Code 輸出執行 `truffleHog` 或 `git-secrets`
5. **速率限制**：設定 `max_turns` 以避免失控迴圈（建議：20-30）

---

<a id="production"></a>
## 正式環境用法：CI Pipeline

<a id="github-actions-integration"></a>
### GitHub Actions 整合

```yaml
# .github/workflows/ai-fix.yml
name: AI Bug Fix
on:
  issues:
    types: [labeled]

jobs:
  ai-fix:
    if: github.event.label.name == 'ai-fix'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Claude Code
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          pip install claude-code
          
          ISSUE_BODY="${{ github.event.issue.body }}"
          
          claude -p "Fix the following bug: $ISSUE_BODY
          
          Rules:
          - Read the relevant files first
          - Make minimal changes
          - Run tests and verify they pass
          - Do not change unrelated code
          " --output-format json --max-turns 15 > result.json
      
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v5
        with:
          title: "AI Fix: ${{ github.event.issue.title }}"
          body: "Automated fix by Claude Code"
          branch: "ai-fix/${{ github.event.issue.number }}"
```

<a id="cost-model-for-ci"></a>
### CI 成本模型

| 任務類型 | 平均回合數 | 平均 Token 數 | 預估成本 |
|-----------|-----------|------------|----------------|
| Bug 修復（小） | 8 | 15K | $0.23 |
| 測試生成 | 12 | 25K | $0.38 |
| 功能實作 | 20 | 50K | $0.75 |
| 大型重構 | 30 | 100K | $1.50 |

*若每天執行 100 次 CI：依任務組合不同，約為 ~$75-150/天。*

---

<a id="comparison"></a>
## 比較：Claude Code 與替代方案

| 功能 | Claude Code | Cursor/Windsurf | Cline | OpenHands |
|---------|-------------|-----------------|-------|-----------|
| **介面** | CLI + SDK | IDE（VS Code fork） | VS Code extension | Web UI + CLI |
| **模型** | 僅 Claude | 任意（GPT、Claude、Gemini） | 任意 | 任意 |
| **自主性** | 完整 | 中等（需要點擊） | 完整 | 完整 |
| **CI/Headless** | ✅ 原生支援 | ❌ | ✅ | ✅ |
| **MCP 支援** | ✅ 原生支援 | ✅ | ✅ | ✅ |
| **CLAUDE.md** | ✅ | ❌（相似概念：.cursorrules） | ❌ | ❌ |
| **開源** | ❌ | ❌ | ✅ | ✅ |
| **最適合** | 後端開發者、CI/CD | UI/frontend 開發者、視覺化工作流 | 任何開發者 | 自行託管團隊 |

<a id="swe-bench-verified-scores-march-2026"></a>
### SWE-bench Verified 分數（2026 年 3 月）

| 代理 | 分數 | 備註 |
|-------|-------|-------|
| Claude Code (claude-3-7-sonnet) | ~70% | Anthropic 官方代理 |
| OpenHands + claude-3-7-sonnet | ~60% | 開源框架 |
| Devin（商業版） | ~45% | Cognition AI 產品 |
| SWE-agent + GPT-4o | ~38% | Princeton 研究 |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-does-claude-code-differ-from-github-copilot"></a>
### Q：Claude Code 與 GitHub Copilot 有何不同？

**強答案：**
Copilot 是**補全工具**——你在輸入時，它會預測接下來幾行程式碼。Claude Code 則是**自主代理**——你交給它一個任務（例如「為這個 API 加上驗證」），它會讀取程式碼庫、規劃實作、編輯多個檔案、執行測試、修正失敗，並且只有在測試通過時才算完成。兩者體驗本質上不同：Copilot 幫你更快寫程式；Claude Code 則在你審查輸出的同時替你寫程式。

<a id="q-what-is-claude-md-and-why-is-it-critical"></a>
### Q：CLAUDE.md 是什麼？為什麼它如此關鍵？

**強答案：**
CLAUDE.md 很像是專門寫給 AI 同事看的 `README`。沒有它時，Claude Code 只會把你的專案視為一般的 Python/JS 專案。有了它，Claude 就知道：你的精確測試指令、禁止模式（不可用 raw SQL，必須使用 ORM）、架構決策（JWT auth、特定錯誤格式），以及程式碼標準。它能把通用型代理轉變成**專案專家**。在一份寫得好的 CLAUDE.md 幫助下，我看過任務完成速度快 2-3 倍、錯誤減少 60%。

<a id="q-how-do-you-safely-run-claude-code-in-production-ci"></a>
### Q：你會如何在正式環境 CI 中安全地執行 Claude Code？

**強答案：**
三層做法：
1. **Sandbox**：讓 Claude Code 在沒有外部網路存取的 Docker container 中執行。只有 git repo 與測試執行器可存取。
2. **權限 allow-list**：利用 permissions 設定，精確白名單允許的 bash 命令（測試執行器、linter），並封鎖破壞性操作（rm -rf、未審查的 pip install）。
3. **人工關卡**：Claude Code 輸出的是帶有 diff 的 branch。由人工在 PR 中審查 diff 並合併。Claude 永遠不會直接 merge 到 main。這能讓最終決策仍保有人類判斷。

<a id="q-how-do-you-handle-the-cost-of-claude-code-for-high-volume-ci"></a>
### Q：在高流量 CI 場景下，你如何控制 Claude Code 的成本？

**強答案：**
我會從三個方面最佳化：
1. **任務範圍控制**：Claude Code 對於獨立且邊界明確的任務（bug 修復、測試生成）很划算。我不會把它用在開放式探索，因為那種情況人工通常更便宜。
2. **最大回合數**：設定 `max_turns=15` 能避免失控工作在循環推理上燒掉超過 $10。
3. **模型路由**：針對簡單 bug 修復（語法錯誤、明顯拼字錯誤），我會透過 SDK 使用 Claude 3.5 Haiku——成本低 5 倍。對於架構級重構，則使用啟用 Extended Thinking 的 Claude 3.7 Sonnet。

---

<a id="references"></a>
## 參考資料

- Anthropic. "Claude Code: Building Agentic Coding Experiences" (2025) — https://docs.anthropic.com/claude-code
- Anthropic. "Claude Code SDK Documentation" — https://github.com/anthropics/claude-code
- Anthropic. "CLAUDE.md Best Practices" — https://docs.anthropic.com/claude-code/settings#claudemd
- SWE-bench Verified Leaderboard — https://www.swebench.com/

---

*下一篇：[OpenCoder / AI Coding Agents 全景](10-opencoderguide.md)*
