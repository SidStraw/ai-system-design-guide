<a id="architecture-patterns-for-tool-use-agents"></a>
# 工具使用代理的架構模式

2026 年的每一個工具使用代理——從 OpenClaw、Claude Code 到 Cursor 的 Background Agents——都建立在少數幾種核心架構模式之一上。理解這些模式，能讓你從第一原理設計代理，而不是只複製特定工具。本章以詳細圖解、程式碼範例、取捨分析，以及何時該選用哪一種模式的指引，逐一拆解這些模式。

<a id="table-of-contents"></a>
## 目錄

- [模式 1：函式／工具呼叫](#pattern-1-functiontool-calling)
- [模式 2：視覺式自動化](#pattern-2-vision-based-automation)
- [模式 3：本機程式碼執行](#pattern-3-local-code-execution)
- [模式 4：多代理工具協調](#pattern-4-multi-agent-tool-orchestration)
- [沙箱化 vs. 非沙箱化執行](#sandboxed-vs-unsandboxed-execution)
- [跨工具呼叫的狀態管理](#state-management-across-tool-calls)
- [錯誤處理與重試模式](#error-handling-and-retry-patterns)
- [MCP 整合模式](#mcp-integration-patterns)
- [架構決策樹](#architecture-decision-tree)
- [系統設計面試切入角度](#system-design-interview-angle)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="pattern-1-functiontool-calling"></a>
## 模式 1：函式／工具呼叫

這是目前在正式環境中部署最廣泛的模式。LLM 決定要呼叫哪個工具以及使用哪些參數；框架執行該呼叫；結果再被回饋到對話中，供下一步推理使用。

<a id="architecture"></a>
### 架構

```
+-------------------------------------------------------------------+
|              Function/Tool Calling Pattern                        |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  User Message     |                                            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+     +------------------+                    |
|  |  LLM Reasoning   |---->|  Tool Selection  |                   |
|  |                   |     |                  |                    |
|  |  "I need to look  |     |  tool: search_db |                   |
|  |   up the order"  |     |  args: {id: 42}  |                    |
|  +-------------------+     +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  Tool Executor   |                   |
|                            |  (Framework)     |                   |
|                            |                  |                   |
|                            |  Validates args  |                   |
|                            |  Calls function  |                   |
|                            |  Returns result  |                   |
|                            +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  Result Injected |                   |
|                            |  into Context    |                   |
|                            |                  |                   |
|                            |  {status: "shipped",                 |
|                            |   tracking: "1Z..."} |               |
|                            +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  LLM Generates   |                   |
|                            |  Final Response  |                   |
|                            +------------------+                   |
+-------------------------------------------------------------------+
```

<a id="the-three-steps-in-detail"></a>
### 三個步驟的細節

**步驟 1 -- Schema Presentation**：模型會接收到一份 JSON schema，用來描述可用工具。到了 2026 年，最佳實務是使用 Dynamic Manifests，依據使用者意圖只載入相關工具，而不是一開始就把所有工具 schema 全部載入。

**步驟 2 -- Intent and Extraction**：模型輸出結構化的工具呼叫。這不是自由文字，而是包含 `tool_name` 與 `arguments` 的 JSON 物件，讓框架可以做確定性的解析。

**步驟 3 -- Execution and Contextualization**：框架會驗證參數（使用 Pydantic、Zod 或類似工具）、呼叫函式，並把結果以角色為 `tool` 的新訊息注入回對話內容中。

<a id="code-example-mcp-server-client"></a>
### 程式碼範例：MCP Server + Client

```python
# MCP Server: defines a tool with strict schema
from mcp.server import Server
from pydantic import BaseModel, Field

server = Server("order-service")

class OrderLookup(BaseModel):
    """Look up an order by ID. DO NOT use for cancelled orders."""
    order_id: str = Field(..., description="The order UUID")

@server.tool()
async def lookup_order(args: OrderLookup) -> dict:
    order = await db.orders.find_one({"id": args.order_id})
    if not order:
        return {"error": "Order not found", "suggestion": "Check order ID format"}
    return {"status": order["status"], "tracking": order.get("tracking_number")}
```

```python
# MCP Client: agent discovers tools dynamically, calls them, feeds results back
tools = await mcp_client.list_tools()
response = client.messages.create(model="claude-sonnet-4-6", tools=tools,
    messages=[{"role": "user", "content": "Where is my order ORD-12345?"}])

if response.stop_reason == "tool_use":
    tool_call = response.content[0]
    result = await mcp_client.call_tool(tool_call.name, tool_call.input)
    # Feed result back as a tool_result message for the next LLM turn
```

<a id="when-to-use-this-pattern"></a>
### 何時使用這個模式

- API 整合（資料庫、SaaS 工具、內部服務）
- 結構化資料擷取與修改
- 任何能事先定義工具介面的工作流程
- 需要稽核軌跡與輸入驗證的正式環境系統

<a id="trade-offs"></a>
### 取捨

| 優點 | 缺點 |
|-----------|--------------|
| 可確定性的執行 | 需要預先定義工具 schema |
| 容易稽核與記錄 | 無法與任意 UI 互動 |
| 快速（每次工具呼叫 50-200ms） | 模型可能幻想出不存在的工具名稱／參數 |
| 適用於任何支援 tool use 的 LLM | 工具太多時會有 schema 過載問題 |

---

<a id="pattern-2-vision-based-automation"></a>
## 模式 2：視覺式自動化

模型會看到螢幕截圖，推理接下來該做什麼，並輸出低階動作（click、type、scroll）。環境執行動作後，再拍攝新的截圖，然後重複這個循環。這就是 Claude Computer Use 與 Open Interpreter 的 Computer API 的運作方式。

<a id="architecture-1"></a>
### 架構

```
+-------------------------------------------------------------------+
|              Vision-Based Automation Pattern                      |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  Task Goal        |  "Fill out the expense form with           |
|  |  (NL instruction) |   last week's receipts"                    |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+--------------------------------------------------+    |
|  |                    VISION-ACTION LOOP                      |    |
|  |                                                            |    |
|  |   +------------+    +-------------+    +------------+     |    |
|  |   |  OBSERVE   |    |  REASON     |    |  ACT       |     |    |
|  |   |            |    |             |    |            |     |    |
|  |   | Screenshot |--->| Analyze     |--->| Emit action|     |    |
|  |   | (base64)   |    | screenshot  |    | {type:     |     |    |
|  |   |            |    | + goal      |    |  "click",  |     |    |
|  |   |            |    | + history   |    |  x: 450,   |     |    |
|  |   |            |    | + prev acts |    |  y: 320}   |     |    |
|  |   +-----^------+    +-------------+    +------+-----+     |    |
|  |         |                                      |          |    |
|  |         +--------------------------------------+          |    |
|  |                    (Loop until done)                       |    |
|  +-----------------------------------------------------------+    |
|                            |                                      |
|                            v                                      |
|  +-------------------------+------------------------------+       |
|  |         Sandboxed Environment (VM / Docker + VNC)      |       |
|  |                                                        |       |
|  |   +----------+  +----------+  +----------+            |       |
|  |   | Desktop  |  | Browser  |  | Apps     |            |       |
|  |   | (Xfce)   |  | (Chrome) |  | (any)    |            |       |
|  |   +----------+  +----------+  +----------+            |       |
|  +--------------------------------------------------------+       |
+-------------------------------------------------------------------+
```

<a id="the-observe-reason-act-cycle"></a>
### Observe-Reason-Act 週期

**Observe**：擷取目前螢幕狀態的截圖。在 Claude Computer Use 中，這通常是以 base64 編碼的 PNG，作為影像內容區塊傳送。Zoom Action（2026 年新功能）可擷取特定區域的高解析裁切畫面，適合密集型 UI。

**Reason**：多模態 LLM 會結合任務目標與動作歷史來分析截圖，並決定下一步動作。這一步通常消耗最多 token。

**Act**：模型會輸出結構化動作：
- `left_click(x, y)` -- 在指定座標點擊
- `type(text)` -- 輸入字串
- `key(key_combo)` -- 按下鍵盤快捷鍵
- `scroll(direction, amount)` -- 捲動畫面
- `screenshot()` -- 在不執行動作的情況下重新截圖
- `zoom(x0, y0, x1, y1)` -- 以高解析度檢視某個區域

<a id="code-example-computer-use-loop"></a>
### 程式碼範例：Computer Use 迴圈

```python
tools = [
    {"type": "computer_20250124", "name": "computer",
     "display_width_px": 1280, "display_height_px": 800},
    {"type": "bash_20250124", "name": "bash"},
    {"type": "text_editor_20250124", "name": "str_replace_based_edit_tool"}
]
messages = [{"role": "user", "content": "Open the browser and go to GitHub."}]

while True:  # The vision-action loop
    response = client.messages.create(
        model="claude-sonnet-4-6", max_tokens=4096, tools=tools, messages=messages)
    if response.stop_reason == "end_turn":
        break
    for block in response.content:
        if block.type == "tool_use":
            result = sandbox.execute_action(block.name, block.input)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": [
                {"type": "tool_result", "tool_use_id": block.id, "content": result}]})
```

<a id="when-to-use-this-pattern-1"></a>
### 何時使用這個模式

- 自動化沒有 API 的舊式應用程式
- 圖形介面的端對端測試
- 需要與多個應用程式互動的任務
- 以自然語言描述任務的非開發者使用者

<a id="trade-offs-1"></a>
### 取捨

| 優點 | 缺點 |
|-----------|--------------|
| 可搭配任何 GUI 應用程式使用 | 慢（每個動作步驟 1-3 秒） |
| 不需要 API 或額外整合 | token 成本高（截圖很大） |
| 能處理動態 UI | 在密集介面上有誤點風險 |
| 對非技術使用者友善 | 為了安全需要沙箱化 VM |

---

<a id="pattern-3-local-code-execution"></a>
## 模式 3：本機程式碼執行

使用者以自然語言描述任務。LLM 產生程式碼。程式碼在本機（或沙箱中）執行。系統觀察輸出後，LLM 會繼續產生更多程式碼，或直接提供最終答案。這就是 Open Interpreter 與 Claude Code 部分能力的運作方式。

<a id="architecture-2"></a>
### 架構

```
User (NL): "Analyze the CSV and plot the top 10 products"
  |
  v
[LLM Generates Code] --> Python/Bash/JS
  |
  v
[Permission Gate] --> "Run this code? [y/N]" (auto-approve, always-ask, or rules-based)
  |
  v
[Code Executor] --> Execute, capture stdout/stderr/return value
  |
  v
[Output Observer] --> Error? Feed back to LLM for fix. Success? Present to user.
  |
  v
[LLM Decides] --> Done? Return result. Need more? Generate next code block. (Loop)
```

<a id="the-nl-code-execute-observe-cycle"></a>
### NL-Code-Execute-Observe 週期

**1. Natural Language to Code**：LLM 會把使用者意圖轉成可執行程式碼。使用哪種語言取決於任務——資料分析用 Python、系統操作用 bash、網頁任務用 JavaScript。

**2. Permission Gate**：在執行前，會先要求使用者核准。這是非沙箱化環境中最關鍵的安全機制。常見實作包括：
- **Always ask**（Open Interpreter 預設）：每個程式碼區塊都必須明確核准
- **Auto-approve**（trusted mode）：危險但快速
- **Rules-based**（Claude Code 模式）：在設定中定義允許／拒絕規則。例如：允許 `git` 指令，拒絕 `rm -rf`

**3. Execute and Capture**：程式碼會在具備完整（或受限）系統權限的執行環境中運作。系統會擷取 stdout、stderr、回傳值，以及任何產生的檔案。

**4. Observe and Iterate**：LLM 會看到執行輸出。如果出現錯誤，它會產生修正；若輸出只是部分結果，它會產生下一步。這形成一個可自我修正的迴圈。

<a id="code-example-code-execution-agent"></a>
### 程式碼範例：Code Execution Agent

```python
class CodeExecutionAgent:
    def __init__(self, llm_client, sandbox=None):
        self.llm = llm_client
        self.sandbox = sandbox  # None = unsandboxed (host)
        self.history = []

    async def run(self, task: str) -> str:
        self.history.append({"role": "user", "content": task})
        for iteration in range(10):  # Max 10 code-execute cycles
            response = await self.llm.generate(messages=self.history)
            code = extract_code_block(response)
            if not code:
                return response  # No code = final answer
            if not self.sandbox and not await user_approves(code):
                return "Execution cancelled by user."
            result = await (self.sandbox or LocalExecutor()).run(code, timeout=30)
            self.history.append({"role": "assistant", "content": response})
            self.history.append({"role": "user",
                "content": f"stdout: {result.stdout}\nstderr: {result.stderr}"})
        return "Max iterations reached."
```

<a id="when-to-use-this-pattern-2"></a>
### 何時使用這個模式

- 資料分析與視覺化任務
- 系統管理與 DevOps 自動化
- 檔案處理與轉換
- 使用者只描述「做什麼」，由代理決定「怎麼做」的任務

<a id="trade-offs-2"></a>
### 取捨

| 優點 | 缺點 |
|-----------|--------------|
| 極度彈性 | 若未沙箱化則有安全風險 |
| 可透過 observe 迴圈自我修正 | 模型可能產生危險程式碼 |
| 可搭配本機模型離線運作 | 需要使用者評估程式碼（或選擇信任） |
| 有需要時可取得完整系統存取權 | 非確定性（相同提示，產生不同程式碼） |

---

<a id="pattern-4-multi-agent-tool-orchestration"></a>
## 模式 4：多代理工具協調

與其使用一個擁有許多工具的代理，不如使用多個各自負責一部分工具的專門代理。協調器會把任務路由到正確的代理。這可說是代理世界的「微服務革命」。

<a id="architecture-3"></a>
### 架構

```
  [User Request]
       |
       v
  [ORCHESTRATOR] (Frontier model: Claude Opus, GPT-4o)
  Analyzes task, selects agent, routes and waits
       |
  +----+----+----+
  |         |         |
  v         v         v
[Code Agent]  [Data Agent]  [Web Agent]
 bash, edit,   SQL, plot,    fetch, scrape,
 git           csv            browse
  |         |         |
  v         v         v
[Sandbox]  [Sandbox]  [Sandbox]
(Docker)   (Docker)   (Docker)
```

<a id="orchestration-strategies"></a>
### 協調策略

**1. Router-Based（最簡單）**：協調器本身是分類器。它查看使用者訊息、選出正確的專家代理，並把整個任務轉交出去。不需要代理彼此溝通。

**2. Plan-and-Execute**：規劃模型（frontier-class）會把任務拆成子任務，並指派給合適的專家。最後再由規劃者彙整子任務結果。基準測試顯示，相較於序列式 ReAct，這種方式能達到 92% 的任務完成率與 3.6 倍速度提升。

**3. Hierarchical**：高階代理把工作分派給低階代理，而低階代理還可以再往下委派。這很像組織結構，特別適合複雜專案。

**4. Collaborative（Peer-to-Peer）**：代理可以彼此直接溝通、分享觀察，並互相請求協助。這是最複雜的模式，但很適合處理具湧現性的任務。

<a id="cost-optimization-the-plan-and-execute-advantage"></a>
### 成本最佳化：Plan-and-Execute 的優勢

```
Traditional: [Frontier Model] handles all steps       Cost: $1.00/task

Plan-and-Execute:
  [Frontier Model] plans (1 call)                     Cost: $0.05
  [Small Model] executes steps 1-3                    Cost: $0.03
  [Frontier Model] aggregates (1 call)                Cost: $0.05
                                                      Total: $0.13/task
                                                      Savings: ~87%
```

2026 年的趨勢，是把代理成本最佳化視為一級設計考量，就像微服務時代裡雲端成本最佳化變得不可或缺一樣。

---

<a id="sandboxed-vs-unsandboxed-execution"></a>
## 沙箱化 vs. 非沙箱化執行

對任何工具使用代理來說，這都是影響最深遠的架構決策。

<a id="comparison"></a>
### 比較

```
  UNSANDBOXED (Host Access)              SANDBOXED (Isolated)
  +------------------------+             +------------------------+
  | LLM output executes    |             | LLM output executes    |
  | directly on host OS    |             | inside Docker/VM/E2B   |
  |                        |             |                        |
  | Risk: rm -rf /         |             | Isolated filesystem,   |
  | Risk: data exfiltration|             | network, processes     |
  |                        |             |                        |
  | Used by: OpenClaw,     |             | Used by: OpenHands,    |
  | Open Interpreter,      |             | OpenAI Codex, Jules,   |
  | Claude Code (default)  |             | Cursor Background Agents|
  +------------------------+             +------------------------+
```

<a id="sandbox-implementation-options"></a>
### 沙箱實作選項

| 技術 | 隔離等級 | 啟動時間 | 使用情境 |
|------------|----------------|-------------|----------|
| Docker | 程序 + FS | 1-5 秒 | 多數代理沙箱（OpenHands） |
| Firecracker | 完整 VM（microVM） | 約 125ms | 高安全、多租戶 |
| gVisor | 核心層級 | 約 200ms | Google Cloud Run |
| E2B | 雲端沙箱 | 2-3 秒 | 遠端代理執行 |
| WebAssembly | 語言層級 | <50ms | 瀏覽器內執行 |

<a id="the-2026-consensus"></a>
### 2026 年共識

預設採用沙箱，必要時再提供 escape hatch。OpenClaw 的安全危機（135,000 個暴露在公網上的實例）讓整個產業開始嚴肅看待這件事。新的正式環境代理預期都應該預設啟用沙箱；非沙箱化執行則保留給單一使用者、有人監督的環境。

---

<a id="state-management-across-tool-calls"></a>
## 跨工具呼叫的狀態管理

代理需要在工具呼叫之間維持狀態。應採用哪種策略，取決於代理生命週期與使用情境。

<a id="state-management-patterns"></a>
### 狀態管理模式

| 模式 | 生命週期 | 儲存位置 | 使用者 |
|---------|-----------|---------|---------|
| **Conversation State** | 短暫（單次對話） | 訊息陣列 | 多數 API 型代理 |
| **Session State** | 每個 session（工作目錄、開啟檔案） | Docker container / temp dir | OpenHands、Claude Code |
| **Persistent State** | 跨 session（數天、數週） | DB、檔案、Markdown | OpenClaw（Memories/）、CLAUDE.md |
| **Environment State** | 外部（事實來源） | Git repo、database、FS | Claude Code（`git status`）、CI/CD |

<a id="implementation-session-state"></a>
### 實作：Session State

```python
class AgentSession:
    """Manages state across tool calls within a single session."""
    def __init__(self):
        self.conversation: list[dict] = []
        self.working_dir: str = tempfile.mkdtemp()
        self.open_files: dict[str, str] = {}  # path -> content cache
        self.tool_call_count: int = 0

    def add_tool_result(self, tool_name: str, args: dict, result: dict):
        self.tool_call_count += 1
        self.conversation.append({"role": "tool", "tool_name": tool_name,
            "args": args, "result": result, "timestamp": time.time()})
        # Update derived state from side effects
        if tool_name == "write_file":
            self.open_files[args["path"]] = args["content"]

    def get_context_for_llm(self, max_tokens: int = 100_000) -> list[dict]:
        """Return conversation history, compressed if over budget."""
        if estimate_tokens(self.conversation) < max_tokens:
            return self.conversation
        return self._compress_history(max_tokens)  # Summarize old results
```

---

<a id="error-handling-and-retry-patterns"></a>
## 錯誤處理與重試模式

工具呼叫會失敗。網路會逾時。API 會回傳錯誤。程式碼會丟出例外。正式環境代理需要系統化的錯誤處理機制。

<a id="error-taxonomy"></a>
### 錯誤分類

| 錯誤類型 | 範例 | 策略 |
|-----------|----------|----------|
| **Transient** | 網路逾時、rate limit、503 | 指數退避重試（最多 3 次） |
| **Input** | 參數無效、格式錯誤 | 把錯誤回饋給 LLM，讓它修正參數 |
| **Permission** | 驗證失敗、存取遭拒 | 回報給使用者，**不要**重試 |
| **Logic** | 工具選錯、操作不可能完成 | 把錯誤回饋給 LLM，讓它重新規劃 |
| **Catastrophic** | OOM、沙箱崩潰、無限迴圈 | 中止、回報並清理資源 |

<a id="retry-pattern-implementation"></a>
### 重試模式實作

```python
class ToolExecutor:
    MAX_RETRIES = 3

    async def execute_with_retry(self, tool_name: str, args: dict) -> dict:
        for attempt in range(self.MAX_RETRIES):
            try:
                result = await self.call_tool(tool_name, args)
                if not result.get("error"):
                    return result  # Success
                error_type = classify_error(result["error"])
                if error_type == "transient":
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff
                    continue
                elif error_type == "input":
                    return {"error": result["error"], "fix_hint": "Adjust args"}
                elif error_type == "permission":
                    return {"error": result["error"], "action": "Report to user"}
                else:  # catastrophic
                    await self.cleanup_sandbox()
                    return {"error": "Fatal error. Task aborted."}
            except TimeoutError:
                if attempt < self.MAX_RETRIES - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
        return {"error": f"Failed after {self.MAX_RETRIES} retries"}
```

<a id="the-self-correction-loop"></a>
### 自我修正迴圈

這是 2026 年最強大的錯誤處理模式。代理會觀察自己的失敗，並自主修正：

```
LLM generates code/tool call
  --> Execute --> Success? -- YES --> Return result
                     |
                     NO
                     |
                     v
              Feed error + stderr to LLM --> LLM generates fix --> Execute again
              (max 5 corrections to prevent infinite loops)
```

這就是 Claude Code、OpenHands 與 Cline 處理測試失敗的方式：執行測試、看見失敗、編輯程式碼、重新跑測試，反覆進行直到全部變綠。

---

<a id="mcp-integration-patterns"></a>
## MCP 整合模式

MCP 已成為 2026 年工具整合的標準協定。以下是把 MCP 整合進代理架構時的幾個關鍵模式。

<a id="pattern-a-direct-mcp-connection"></a>
### 模式 A：直接連線到 MCP

```
[Agent (Client)] <-- stdio / HTTP --> [MCP Server]
```
最簡單的模式。一個代理、一個伺服器。適用於單一用途工具（資料庫、檔案系統）。

<a id="pattern-b-multi-server-fan-out"></a>
### 模式 B：多伺服器 Fan-Out

```
                  +--> [GitHub MCP]
[Agent (Client)]--+--> [Postgres MCP]
                  +--> [Slack MCP]
```
代理會同時連到多個 MCP 伺服器。工具 schema 會被合併為一份 manifest。Claude Code 與多工具助理都採用這種方式。

<a id="pattern-c-mcp-gateway-enterprise"></a>
### 模式 C：MCP Gateway（企業版）

```
[Agent 1] --+                          +--> [GitHub MCP]
[Agent 2] --+--> [MCP Gateway]  --+--> [Postgres MCP]
[Agent 3] --+    (Auth, Rate Limit,    +--> [Slack MCP]
                  Audit, Route)
```
中央 gateway 會處理 auth、rate limiting 與 audit logging。代理只需要向 gateway 驗證。這種方式常見於企業與多租戶部署。

<a id="mcp-roadmap-gaps"></a>
### MCP 路線圖缺口

目前的 MCP 規格（截至 2026 年 5 月）仍缺少三個關鍵的正式環境原語：

1. **Identity Propagation**：缺乏從 client 經由 server 傳遞使用者身分的標準化方式。gateway 模式是目前的變通方案。
2. **Adaptive Tool Budgeting**：協定層沒有提供限制單次工具呼叫 token／成本消耗的能力。
3. **Structured Error Semantics**：沒有標準錯誤碼或錯誤分類。每個 server 都定義自己的錯誤格式。

這些項目都在 2026 路線圖中，但尚未正式核准。

---

<a id="architecture-decision-tree"></a>
## 架構決策樹

使用這個決策樹為你的使用情境選出正確模式：

```
Does the target system have an API?
 +-- YES --> Pattern 1 (Tool Calling). Wrap as MCP server. Fastest, most reliable.
 +-- NO  --> Does the task require GUI interaction?
              +-- YES --> Pattern 2 (Vision-Based). Sandbox in VM. Accept latency.
              +-- NO  --> Is the task primarily code/data work?
                           +-- YES --> Pattern 3 (Code Exec). Sandbox if multi-tenant.
                           +-- NO  --> Complex enough for multiple specialists?
                                        +-- YES --> Pattern 4 (Multi-Agent Orch.)
                                        +-- NO  --> Pattern 1 with custom tool.
```

<a id="hybrid-architectures"></a>
### 混合式架構

在實務上，正式環境系統會混合使用多種模式。Claude Code 會使用：
- 模式 1（工具呼叫）處理檔案操作與 git
- 模式 2（視覺式）處理 computer use 功能
- 模式 3（程式碼執行）處理 bash 與測試執行
- 模式 4（多代理）處理 subagent 啟動

關鍵原則是：預設先使用最簡單的模式（function calling），只有在使用情境真的需要時才增加複雜度。

---

<a id="system-design-interview-angle"></a>
## 系統設計面試切入角度

在面試中討論工具使用架構時，可以用以下五個維度來組織你的回答：

<a id="1-pattern-selection"></a>
### 1. 模式選擇

先辨識適合的模式：「目標系統有 REST API，所以我會使用 function/tool calling 模式，並用 MCP server 包住 API。」這能展現你理解整個決策樹。

<a id="2-sandbox-boundary"></a>
### 2. 沙箱邊界

一定要談安全性：「若是多租戶部署，我會把每位使用者的代理 session 放進 Docker container 中執行，並切斷它對內部服務的網路存取。MCP server 則跑在沙箱外，負責中介所有外部呼叫。」

<a id="3-state-strategy"></a>
### 3. 狀態策略

說明狀態如何管理：「我會在 Docker container 內使用 session state 來管理工作檔案，並把 environment state（git repo）作為事實來源。這個使用情境不需要持久化的代理記憶。」

<a id="4-error-budget"></a>
### 4. 錯誤預算

討論失敗模式：「工具呼叫可能因 transient error 失敗（以 backoff 重試）、因 input error 失敗（讓 LLM 自我修正），或因 permission error 失敗（直接呈現給使用者）。在升級處理前，我會把自我修正嘗試上限設為 5 次。」

<a id="5-cost-model"></a>
### 5. 成本模型

說明經濟面：「對 orchestrator 來說，我會用 Plan-and-Execute 模式：由 Opus 規劃任務，再由 Haiku 執行每個步驟。與全部都用 Opus 相比，成本大約可降低 87%。」

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-design-a-system-that-lets-a-customer-support-agent-answer-questions-using-data-from-zendesk-salesforce-and-an-internal-knowledge-base"></a>
### 問：設計一個系統，讓客服代理能使用 Zendesk、Salesforce 與內部知識庫的資料回答問題。

**較強的回答：**
採用模式 1（function/tool calling），並為每個資料來源各設一個 MCP server。使用 Multi-Server Fan-Out 模式，再搭配 dynamic manifests，讓每次查詢只載入相關工具。若要上正式環境，再加入 MCP Gateway 來處理各資料來源的 OAuth、rate limiting（尤其 Salesforce API 限額很關鍵）與 audit logging。狀態採短暫型即可——客服情境不需要跨 session 記憶。

<a id="q-how-would-you-prevent-an-ai-agent-from-causing-damage-through-tool-calls"></a>
### 問：你會如何防止 AI 代理透過工具呼叫造成破壞？

**較強的回答：**
採用五層式深度防禦：（1）Schema 約束與 deny-patterns（例如用 regex 拒絕 `DROP TABLE` 等）。（2）對破壞性操作設置 permission gate——Claude Code 的 allow/deny 規則就是很好的範例。（3）沙箱隔離（Docker 搭配唯讀掛載、禁止對外網路）。（4）token 與成本上限，避免失控迴圈。（5）透過 MCP Gateway 模式建立 audit trail。任何單一層都不夠——模型可能幻想出能通過驗證的參數（所以需要沙箱），而沙箱也無法阻止透過允許路徑進行資料外洩（所以需要 audit logging）。

<a id="q-explain-the-trade-offs-between-vision-based-computer-use-and-api-based-tool-calling"></a>
### 問：說明 vision-based computer use 與 API-based tool calling 之間的取捨。

**較強的回答：**
API-based 方式更快（每步 50-200ms vs. 1-3 秒）、更便宜（文字 token vs. 圖像 token）、更可靠（確定性 vs. 座標點擊），也更容易測試。只要有 API，就應優先使用。vision-based 則是沒有 API 的應用程式、legacy systems 或跨多個應用的工作流程之後備方案。2026 年的 Zoom Action 能降低密集 UI 上誤點的問題。最佳實務是：對具備 API 支援的 80% 任務使用 API 呼叫，對剩下的 20% 使用 vision-based。

---

<a id="references"></a>
## 參考資料

- Anthropic.「Computer Use Tool Documentation」(2024-2026)
- Anthropic.「Model Context Protocol Specification」(2025-2026)
- MCP 2026 Roadmap.「Transport Evolution, Agent Communication, Governance」(2026)
- IBM Developer.「MCP Architecture Patterns for Multi-Agent AI Systems」(2026)
- Google Cloud.「Choose a Design Pattern for Your Agentic AI System」(2025-2026)
- Microsoft Azure.「AI Agent Orchestration Patterns」(2025-2026)
- OpenHands Documentation.「Runtime Architecture」(2025-2026)
- OpenClaw Documentation.「Architecture and SOUL.md Guide」(2025-2026)
- Open Interpreter GitHub Repository (2024-2026)
- ArXiv 2603.13417.「Design Patterns for Deploying AI Agents with MCP」(2026)

---

*上一節：[工具使用與電腦代理版圖](01-tool-use-landscape.md)*
*下一章：[案例研究](../16-case-studies/)*
