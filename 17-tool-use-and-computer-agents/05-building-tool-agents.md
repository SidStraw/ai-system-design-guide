<a id="building-tool-use-agents"></a>
# 建構 Tool-Use Agents

本章涵蓋 Tool-Use Agents 的實務工程：設計讓 LLM 能可靠呼叫的工具 schema、建置用來託管這些工具的 MCP server、將工具組合成工作流程，以及測試整個系統。這些模式正是區分示範與正式上線部署的關鍵。

<a id="table-of-contents"></a>
## 目錄

- [設計給 LLM 的 Tool Schemas](#designing-tool-schemas-for-llms)
- [建立 MCP Server](#mcp-server-creation)
- [工具註冊與探索](#tool-registration-and-discovery)
- [輸入驗證與輸出格式化](#input-validation-and-output-formatting)
- [工具組合：串接工具](#tool-composition-chaining-tools)
- [建構自訂 Agent Skills](#building-custom-agent-skills)
- [建立 Function-Calling Endpoints](#creating-function-calling-endpoints)
- [測試 Tool-Use Agents](#testing-tool-use-agents)
- [Tool Use 的可觀測性](#observability-for-tool-use)
- [常見錯誤與 Anti-Patterns](#common-mistakes-and-anti-patterns)
- [工具版本管理與向後相容性](#tool-versioning-and-backwards-compatibility)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="designing-tool-schemas-for-llms"></a>
## 設計給 LLM 的 Tool Schemas

Tool schema 是 LLM 與你的系統之間的契約。設計良好的 schema 能減少幻覺參數、避免誤用，並讓模型在選擇工具時更可靠。

<a id="anatomy-of-a-good-tool-definition"></a>
### 良好 Tool Definition 的結構

```json
{
  "name": "search_customers",
  "description": "Search for customers by name, email, or account ID. Returns up to 10 matching customer records. Use this when the user asks about a specific customer. Do NOT use this for aggregate queries like 'how many customers do we have'.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search term: customer name, email address, or account ID (e.g., 'john@acme.com' or 'ACC-12345')"
      },
      "limit": {
        "type": "integer",
        "description": "Max results to return (1-10). Default: 5",
        "default": 5,
        "minimum": 1,
        "maximum": 10
      }
    },
    "required": ["query"]
  }
}
```

<a id="schema-design-rules"></a>
### Schema 設計規則

**1. 精確命名**：使用 `verb_noun` 格式。用 `search_customers`，不要用 `search` 或 `customer_tool`。

**2. 描述何時不要使用**：模型需要反例。與其只列出可用情境，加入「Do NOT use for aggregate queries」這類說明，更能防止誤用。

**3. 提供參數範例**：在 description 字串中包含範例值。模型會利用這些範例來校準輸出。

**4. 限制範圍**：使用 `minimum`、`maximum`、`enum` 與 `pattern`，在 schema 層就防止無效參數，而不是等到 handler 才處理。

**5. 讓工具保持原子性**：一個工具只做一件事。避免建立同時負責建立、讀取、更新、刪除的 `manage_customer` 工具，應拆成四個工具。

**6. 使用 `strict: true`**：Anthropic 的 strict mode 可保證模型輸出與 schema 完全一致。正式環境中務必啟用。

```
Good Tool Design:                    Bad Tool Design:

+-------------------+                +-------------------+
| search_customers  |                | customer_tool     |
| - query (string)  |                | - action (string) |
| - limit (int 1-10)|                | - data (object)   |
+-------------------+                | - options (any)   |
| create_customer   |                +-------------------+
| - name (string)   |                "action" can be
| - email (string)  |                "search", "create",
+-------------------+                "update", "delete"
| update_customer   |                => model confused,
| - id (string)     |                   schema too loose,
| - fields (object) |                   hard to validate
+-------------------+
```

---

<a id="mcp-server-creation"></a>
## 建立 MCP Server

MCP server 是一個獨立程序，會向任何相容 MCP 的 client（Claude、GPT、Llama-based agents）暴露 tools、resources 與 prompts。你只需要撰寫一次 server，任何 LLM 都能使用它。

<a id="mcp-architecture"></a>
### MCP 架構

```
+------------------+          JSON-RPC           +------------------+
|                  |  ========================>  |                  |
|   MCP Client     |                             |   MCP Server     |
|   (AI App)       |  <========================  |   (Your Code)    |
|                  |                             |                  |
|  - Claude Code   |  Transport:                 |  Exposes:        |
|  - Custom Agent  |  - stdio (local)            |  - Tools         |
|  - IDE Plugin    |  - Streamable HTTP (remote)  |  - Resources     |
|                  |                             |  - Prompts       |
+------------------+                             +------------------+
```

<a id="typescript-mcp-server"></a>
### TypeScript MCP Server

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "customer-service", version: "1.0.0" });

server.tool(
  "search_customers",
  "Search customers by name, email, or ID. Returns up to 10 matches.",
  {
    query: z.string().describe("Search term: name, email, or account ID"),
    limit: z.number().min(1).max(10).default(5).describe("Max results"),
  },
  async ({ query, limit }) => ({
    content: [{ type: "text",
      text: JSON.stringify(await db.customers.search(query, limit), null, 2) }],
  })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

<a id="python-mcp-server-fastmcp"></a>
### Python MCP Server (FastMCP)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("customer-service")

@mcp.tool()
async def search_customers(query: str, limit: int = 5) -> str:
    """Search customers by name, email, or ID. Returns up to 10 matches.
    Args:
        query: Search term - customer name, email, or account ID
        limit: Max results to return (1-10, default 5)
    """
    return json.dumps(await db.customers.search(query, limit), indent=2)
```

這兩個 SDK 都遵循相同模式：建立 server、以型別化 schema 註冊工具，然後連接 transport。TypeScript SDK 使用 Zod 做驗證；Python 則使用 type hints 與 docstrings。

<a id="deployment-modes"></a>
### 部署模式

| 模式 | Transport | 使用情境 |
|------|-----------|----------|
| 本機（stdio） | stdin/stdout pipe | 桌面工具、IDE 外掛 |
| 遠端（Streamable HTTP） | HTTP + SSE | 雲端服務、共享 servers |
| 混合 | Both | 本機開發、遠端部署 |

---

<a id="tool-registration-and-discovery"></a>
## 工具註冊與探索

在正式環境中，agent 需要動態探索可用工具，而不是把它們硬編碼進去。

<a id="static-registration"></a>
### 靜態註冊

在設定檔（例如 `claude_desktop_config.json`）中宣告 MCP servers。每個項目會把 server 名稱對應到 command、args 與可選的 env vars。這種方式簡單，但不夠靈活——不論是否相關，所有 server 都會在啟動時載入。

<a id="dynamic-discovery-tool-search"></a>
### 動態探索（Tool Search）

Anthropic 的 Tool Search（2025）解決了 schema overload 問題。與其把 200 個工具 schema 全部載入 context（這會降低推理品質），agent 會先送出輕量搜尋查詢，只接收 3-5 個最相關的工具 schema。這能讓 context window 聚焦在推理，而不是解析未使用的 schemas。

<a id="mcp-discovery-protocol"></a>
### MCP Discovery Protocol

MCP clients 會透過標準 JSON-RPC methods 探索能力：`tools/list` 回傳可用工具、`resources/list` 回傳資料資源、`prompts/list` 回傳 prompt templates。這讓系統在執行期能探索能力，而不需要硬編碼。

---

<a id="input-validation-and-output-formatting"></a>
## 輸入驗證與輸出格式化

<a id="input-validation-layers"></a>
### 輸入驗證層次

```
+---------------------+
|  Schema Validation   |  <-- JSON Schema / Zod / Pydantic
|  (type, range, enum) |      Catches: wrong types, out-of-range
+----------+----------+
           |
           v
+---------------------+
|  Business Validation |  <-- Your handler code
|  (exists, permitted) |      Catches: invalid IDs, unauthorized
+----------+----------+
           |
           v
+---------------------+
|  Execution           |  <-- Actual operation
+---------------------+
```

務必在兩層都做驗證。Schema validation 會攔下格式錯誤的輸入；Business validation 會攔下語意上無效的輸入。

```python
@mcp.tool()
async def transfer_funds(
    from_account: str,
    to_account: str,
    amount: float
) -> str:
    """Transfer funds between accounts."""
    # Schema already enforced types via type hints

    # Business validation
    if amount <= 0:
        return "Error: Amount must be positive."
    if amount > 10000:
        return "Error: Transfers over $10,000 require manual approval."
    if from_account == to_account:
        return "Error: Cannot transfer to the same account."

    from_acct = await db.accounts.get(from_account)
    if not from_acct:
        return f"Error: Account {from_account} not found."

    # Execute
    result = await db.transfers.execute(from_account, to_account, amount)
    return f"Transferred ${amount:.2f}. Confirmation: {result.id}"
```

<a id="output-formatting"></a>
### 輸出格式化

當模型還需要對結果進一步推理時，請回傳結構化資料。當結果已是最終答案時，則回傳人類可讀的文字。

```python
# Good: structured for further reasoning
return json.dumps({
    "customers": [
        {"id": "ACC-123", "name": "Jane Smith", "email": "jane@acme.com"},
        {"id": "ACC-456", "name": "John Doe", "email": "john@acme.com"}
    ],
    "total_matches": 2,
    "has_more": False
})

# Bad: unstructured blob
return "Found Jane Smith (ACC-123, jane@acme.com) and John Doe (ACC-456, john@acme.com)"
```

---

<a id="tool-composition-chaining-tools"></a>
## 工具組合：串接工具

真實任務通常需要依序呼叫多個工具。常見有兩種組合模式：

<a id="pattern-1-llm-orchestrated-chaining"></a>
### 模式 1：由 LLM 協調的串接

LLM 會根據前一步結果，決定接下來要呼叫哪個工具：

```
User: "Find customer Jane Smith and create a high-priority ticket for her billing issue"

Turn 1:  LLM -> search_customers("Jane Smith")
         Result: {"id": "ACC-123", "name": "Jane Smith", ...}

Turn 2:  LLM -> create_ticket("ACC-123", "Billing issue", "...", "high")
         Result: "Ticket TK-789 created."

Turn 3:  LLM -> "I found Jane Smith (ACC-123) and created ticket TK-789."
```

每次工具呼叫都是一次獨立的 API round-trip。模型會在每次呼叫之間根據結果進行推理。

<a id="pattern-2-programmatic-tool-calling"></a>
### 模式 2：程式化 Tool Calling

Anthropic 的 programmatic tool calling（2025）讓模型能直接撰寫程式碼來串接工具，而不需要多次 round-trip：

```
LLM generates code:
  customer = search_customers("Jane Smith")
  if customer.results:
    ticket = create_ticket(customer.results[0].id, ...)
    return f"Created {ticket.id} for {customer.results[0].name}"
  else:
    return "Customer not found"
```

這會以單次 API 呼叫完成，把延遲從 3 次 round-trip 降到 1 次。

<a id="pattern-3-server-side-composition"></a>
### 模式 3：Server-Side Composition

把工具組合邏輯放在 MCP server 內部——單一的 `resolve_customer_issue` 工具會在內部呼叫 search 與 create_ticket，將多步驟邏輯對 LLM 隱藏起來。這適用於固定、明確定義的工作流程，且 LLM 不需要在步驟間進行推理的情境。

<a id="when-to-use-each"></a>
### 何時使用哪一種

| 模式 | 延遲 | 彈性 | 最適合 |
|---------|---------|-------------|----------|
| LLM-orchestrated | 高（N 次 round-trips） | 很高 | 複雜、分支型邏輯 |
| Programmatic | 低（1 次 round-trip） | 高 | 線性串接、批次處理 |
| Server-side | 最低 | 低 | 固定且常見的工作流程 |

---

<a id="building-custom-agent-skills"></a>
## 建構自訂 Agent Skills

Agent Skills（Anthropic, 2025）是由 instructions、tools 與 resources 打包而成、可由 agent 動態載入的集合。一個 skill 就是一個資料夾：

```
my-skill/
  SKILL.md          # Instructions the agent loads into system prompt
  tools/            # MCP tool implementations
  resources/        # Data files, templates, schemas
  tests/            # Evaluation cases
```

在執行期，SkillManager 會註冊可用 skills，並在需要時啟用——把 skill 的 instructions 注入 system prompt，並將其工具加入可用工具集合。這讓基礎 agent 保持輕量，同時仍能支援深度專精。

---

<a id="creating-function-calling-endpoints"></a>
## 建立 Function-Calling Endpoints

若要讓你的 API 能被任何 LLM 呼叫，可透過搭配 Pydantic models 的 FastAPI 來暴露它。自動產生的 OpenAPI spec（`/openapi.json`）也能兼作 function calling 的 tool schema。或者，你也可以把相同邏輯包裝成 MCP server，直接整合到 Claude、GPT 或其他相容 MCP 的 clients。

---

<a id="testing-tool-use-agents"></a>
## 測試 Tool-Use Agents

<a id="three-testing-layers"></a>
### 三層測試

```
+---------------------------+
|   Eval Suites             |  End-to-end: does the agent
|   (Agent + LLM + Tools)  |  complete the task?
+-------------+-------------+
              |
+-------------v-------------+
|   Integration Tests       |  Does tool X work correctly
|   (Tool + Dependencies)   |  with real DB / API?
+-------------+-------------+
              |
+-------------v-------------+
|   Unit Tests              |  Does validation logic
|   (Tool Logic Only)       |  handle edge cases?
+---------------------------+
```

<a id="unit-tests-for-tools"></a>
### 工具的 Unit Tests

以 mocked dependencies 將每個 tool handler 獨立測試。要涵蓋：輸入驗證邊界情況（超出範圍的值、缺漏欄位）、錯誤訊息品質（是否有引導模型恢復？），以及輸出格式（有效 JSON、正確 schema）。

<a id="eval-suites-for-agent-behavior"></a>
### Agent 行為的 Eval Suites

建立一組包含 100+ 個真實查詢與預期結果的資料集：

```python
eval_cases = [
    {
        "input": "Find Jane Smith's account and check her last payment",
        "expected_tools": ["search_customers", "get_payment_history"],
        "max_tool_calls": 5,
    },
    {
        "input": "What is the meaning of life?",
        "expected_tools": [],  # Should NOT call any tools
        "max_tool_calls": 0,
    },
]
```

針對每個案例，衡量：工具選擇正確率（工具選對了嗎？）、參數品質（args 正確嗎？）、任務完成率，以及效率（工具呼叫次數）。每次 model version 變更與 tool schema 變更時，都要重新跑 evals。

---

<a id="observability-for-tool-use"></a>
## Tool Use 的可觀測性

每次工具呼叫都應記錄：trace/span IDs、timestamp、tool name、input args、output size、latency、status、使用的 model、token usage 與 session ID。

<a id="key-metrics"></a>
### 關鍵指標

| 指標 | 衡量內容 | 警示門檻 |
|--------|-----------------|-----------------|
| Tool call success rate | 回傳有效結果的呼叫百分比 | < 95% |
| Tool selection accuracy | 是否選對工具？ | < 90% |
| Avg tool calls per task | Tool Use 的效率 | > 2x baseline |
| Latency per tool call | Tool handler 的回應時間 | > 5s (p99) |
| Hallucinated arguments | 即使有 schema 仍出現無效 args | > 2% |
| Cost per task | LLM + 工具執行的總成本 | > budget |

<a id="tracing-architecture"></a>
### Tracing 架構

```
+-------------+     +----------------+     +--------------+
|  Agent      |---->|  Tool Handler  |---->|  Backend     |
|  (LLM call) |     |  (MCP Server)  |     |  (DB/API)    |
+------+------+     +--------+-------+     +------+-------+
       |                     |                     |
       v                     v                     v
+------+---------------------+---------------------+------+
|                    Trace Collector                       |
|              (OpenTelemetry / Langfuse)                  |
+---------------------------+------------------------------+
                            |
                            v
                   +--------+--------+
                   |   Dashboard     |
                   |   - Success %   |
                   |   - Latency     |
                   |   - Cost        |
                   +-----------------+
```

---

<a id="common-mistakes-and-anti-patterns"></a>
## 常見錯誤與 Anti-Patterns

| Anti-Pattern | 問題 | 修正方式 |
|-------------|---------|-----|
| Tool overload | 50+ 個工具會降低選擇正確率 | 動態探索，每回合只載入 5-10 個 |
| Vague descriptions | 「Handles customer operations」——太模糊 | 加入何時使用、何時不要使用、範例 |
| God tools | 一個帶有 `action` 參數的工具包辦所有事情 | 拆成原子工具，每個只負責一種操作 |
| Missing error context | 工具只回傳「Error」而沒有細節 | 使用可操作訊息：「ACC-999 not found. Use search_customers...」 |
| Unstructured output | 工具回傳需要模型自行解析的 prose | 以 JSON 回傳，利於結構化推理 |
| No idempotency | `create_ticket` 呼叫兩次就建立重複資料 | 接受 idempotency key，建立前先檢查 |
| Exposing internal IDs | 工具要求模型不可能知道的資料庫 UUIDs | 接受人類可讀識別子，並在內部解析 |
| Ignoring rate limits | Agent 連續打 100 次 API calls 而被限流 | 在 handlers 中實作 backoff，回傳「retry in X seconds」 |

---

<a id="tool-versioning-and-backwards-compatibility"></a>
## 工具版本管理與向後相容性

隨著工具演進，你必須維持與依賴它們的 agents 相容。

**規則：**
1. **Additive changes**（新增可選參數）：不需要升版。舊呼叫依然可用。
2. **Breaking changes**（重新命名、移除參數、改變語意）：請用新 schema 建立新的工具名稱。保留舊工具繼續運作，並在其 description 中加入「DEPRECATED: Use new_tool instead」。同時記錄每次 deprecated call 以供監控。
3. **在確認沒有任何活躍 agent 依賴前，絕不要移除工具**。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-you-need-to-give-an-llm-agent-access-to-200-internal-tools-how-do-you-handle-schema-overload"></a>
### Q: 你需要讓一個 LLM agent 存取 200 個內部工具。你會如何處理 schema overload？

**強答：**
我不會把 200 個工具 schema 全部載入 context。相反地，我會採用兩階段方法。第一階段是工具探索，agent 先描述它需要完成的事，再透過輕量搜尋（embedding similarity 或 keyword match）回傳 5-10 個最相關的工具 schema。第二階段是工具執行，只有被選中的工具會被放入 context，供實際的 LLM 呼叫使用。

這與 Anthropic 的 Tool Search 模式一致。探索步驟可以是一次獨立、成本更低的 LLM 呼叫，甚至可以是不使用 LLM 的搜尋。關鍵洞見在於：context window 若被不相關的工具 schemas 佔用，會直接降低模型的推理品質。我會把工具選擇正確率當作核心指標——如果 agent 在應該呼叫 `get_customer_by_id` 時卻呼叫了 `search_customers`，那就表示探索階段需要調校。

在 MCP 實作上，我會依領域把工具分組成不同 servers（customer-service、billing、analytics），並且只連線到與目前對話相關的 servers。

<a id="q-design-a-testing-strategy-for-a-tool-use-agent-that-handles-customer-support"></a>
### Q: 為處理客戶支援的 tool-use agent 設計一套測試策略。

**強答：**
我會在三個層次測試。第一，為每個 tool handler 撰寫 unit tests：驗證輸入邊界情況、錯誤訊息，以及輸出格式。這些測試會在每次 commit 時，以 mocked dependencies 在 CI 中執行。

第二，使用 integration tests 驗證工具能對真實（staging）資料庫正確運作。例如，`create_ticket` 真的會建立紀錄，而 `search_customers` 也真的能查到它。這能抓出工具與後端之間的 schema drift。

第三，建立測試完整 agent 的 eval suites——也就是 LLM 加上工具。我會建立一組包含 100+ 個真實客戶查詢的資料集，並定義預期的工具呼叫序列與輸出標準。Eval 會衡量工具選擇正確率（有沒有選對工具？）、參數品質（參數是否正確？）、任務完成率（是否真的解決問題？），以及效率（總共用了多少次工具呼叫？）。

每次 model version 變更與 tool schema 變更時，我都會重新跑 evals。如果 schema 變更後工具選擇正確率下降 2%，那代表該修的是 description，而不是模型。

---

<a id="references"></a>
## 參考資料

- Anthropic. "Tool Use with Claude" API Documentation (2025)
- Model Context Protocol. "Build an MCP Server" (2025)
- MCP TypeScript SDK: github.com/modelcontextprotocol/typescript-sdk
- MCP Python SDK: github.com/modelcontextprotocol/python-sdk
- Anthropic. "Introducing Advanced Tool Use" (2025)
- Anthropic. "Agent Skills" Beta Documentation (2025)

---

*上一篇：[Computer-Use Agents](04-computer-use-agents.md)*
