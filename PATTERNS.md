<a id="ai-design-patterns-quick-reference"></a>
# AI 設計模式快速參考

常見模式的快速參考。詳細實作請參閱各章節。

---

<a id="retrieval-patterns"></a>
## 檢索模式

| 模式 | 使用情境 | 主要取捨 |
|---------|----------|--------------|
| **Basic RAG** | 文件上的簡單 Q&A | 容易實作，但精確度有限 |
| **Hybrid Search** | 結合語意搜尋 + 關鍵字搜尋 | 更好的召回率，但更複雜 |
| **Reranking** | 高精確度檢索 | 準確度與延遲的取捨 |
| **Query Expansion** | 模糊查詢 | 更好的召回率，但 token 更多 |
| **HyDE** | 預期沒有直接匹配 | 具有創造性，但可能產生 hallucination |
| **Parent-Child Chunking** | 需要周圍脈絡 | 記憶體額外負擔 |

```
Query → Embed → Vector Search → Rerank → Top-K → Generate
              ↓
         BM25 Search ─────────┘ (hybrid)
```

---

<a id="generation-patterns"></a>
## 生成模式

| 模式 | 使用情境 | 主要取捨 |
|---------|----------|--------------|
| **Zero-Shot** | 簡單任務 | 快速，但可靠性較低 |
| **Few-Shot** | 需要控制輸出格式 | Token 成本 |
| **Chain-of-Thought** | 推理任務 | 延遲增加，且會暴露推理過程 |
| **Self-Consistency** | 高風險答案 | 3-5 倍成本 |
| **Structured Output** | API 回應 | 創造力受限 |

---

<a id="agent-patterns"></a>
## Agent 模式

| 模式 | 使用情境 | 複雜度 |
|---------|----------|------------|
| **ReAct** | 會使用工具的 agents | 中 |
| **Plan-and-Execute** | 多步驟任務 | 高 |
| **Multi-Agent Debate** | 交叉驗證 | 高 |
| **Human-in-the-Loop** | 高風險操作 | 中 |
| **Swarm / Handoff** | 專門化 sub-agents 協作 | 高 |

```
┌─────────────────────────────────────────┐
│              REACT LOOP                  │
│                                         │
│  Observe → Think → Act → Observe → ...  │
│              ↓                          │
│         [Tool Call]                     │
│              ↓                          │
│         [Result]                        │
└─────────────────────────────────────────┘
```

---

<a id="agentic-coding-patterns-2026"></a>
## Agentic Coding 模式（2026）

| 模式 | 使用情境 | 主要工具 |
|---------|----------|----------|
| **Scaffold → Implement → Verify** | 完整功能開發流程 | Claude Code / OpenHands |
| **Read-Plan-Edit** | 重構既有程式碼時 | Claude Code text_editor |
| **Test-Driven Agent** | 高可靠性程式碼 | 由 Agent 先撰寫測試 |
| **Shadow Review** | PR 品質關卡 | 在合併前由 agent 審查 diff |
| **CLAUDE.md Manifest** | 專案脈絡注入 | 使用 Claude Code 的 CLAUDE.md 檔案 |
| **Sub-Agent Parallelism** | 大型程式碼庫變更 | 各模組使用多個 agents |

```
┌────────────────────────────────────────────────────────┐
│              AGENTIC CODING LOOP                        │
│                                                        │
│  Understand → Plan → Implement → Run Tests → Fix       │
│      ↑             (bash + text_editor tools)    │     │
│      └──────────── Iterate until tests pass ────┘     │
│                                                        │
│  [CLAUDE.md injects: coding style, test commands,     │
│   forbidden patterns, architecture decisions]          │
└────────────────────────────────────────────────────────┘
```

**何時該使用哪種工具：**
```
需要完整自主能力 + CLI → Claude Code
需要 open-source + 任意 LLM → OpenHands / Cline
需要緊密的 IDE 整合 → Cursor / Windsurf
需要可重現的 pipelines → 在 Docker CI 中使用 OpenHands
```

---

<a id="reliability-patterns"></a>
## 可靠性模式

| 模式 | 解決的問題 | 實作方式 |
|---------|----------------|----------------|
| **Retry with Backoff** | 暫時性故障 | 指數退避 |
| **Circuit Breaker** | 級聯失敗 | 超過門檻後快速失敗 |
| **Fallback Model** | 主要模型不可用時 | 次要模型 |
| **Timeout** | 回應太慢 | 取消 + fallback |
| **Bulkhead** | 資源隔離 | 分離資源池 |

```python
<a id="reliability-stack"></a>
# 可靠性堆疊
@circuit_breaker(failure_threshold=5)
@retry(max_attempts=3, backoff=exponential)
@timeout(seconds=30)
@fallback(model="gpt-4o-mini")
async def generate(prompt):
    return await primary_model.generate(prompt)
```

---

<a id="caching-patterns"></a>
## 快取模式

| 模式 | 命中率 | 使用情境 |
|---------|----------|----------|
| **Exact Match** | 低 | 相同查詢 |
| **Semantic Cache** | 中 | 相似查詢 |
| **KV Cache** | 高 | 相同前綴 |
| **Response Cache** | 視情況而定 | 決定性輸出 |

---

<a id="security-patterns"></a>
## 安全性模式

| 模式 | 威脅 | 實作方式 |
|---------|--------|----------------|
| **Input Validation** | Prompt injection | 清洗、偵測 |
| **Output Filtering** | 資料外洩 | PII 偵測、封鎖清單 |
| **Tenant Isolation** | 跨租戶存取 | 查詢時進行過濾 |
| **Rate Limiting** | 濫用 | 依使用者／租戶限制 |

```
Input → Validate → Sanitize → LLM → Filter → Validate → Output
```

---

<a id="evaluation-patterns"></a>
## 評估模式

| 模式 | 使用情境 | 指標 |
|---------|----------|---------|
| **Golden Set** | 回歸測試 | 通過率 |
| **LLM-as-Judge** | 品質評分 | 1–5 分量表 |
| **Human Eval** | Ground truth | 一致率 |
| **A/B Testing** | 正式環境中的比較 | 使用者指標 |

---

<a id="cost-optimization-patterns"></a>
## 成本最佳化模式

| 模式 | 節省幅度 | 取捨 |
|---------|---------|----------|
| **Model Routing** | 50-70% | 複雜度 |
| **Caching** | 20-40% | 內容過時風險 |
| **Prompt Compression** | 10-30% | 品質風險 |
| **Batch Processing** | 30-50% | 延遲 |

```
Query → Classify → Route → [Small Model] or [Large Model]
                      ↓
              [Cheap: 80%]  [Expensive: 20%]
```

---

<a id="anti-patterns-to-avoid"></a>
## 應避免的反模式

| 反模式 | 問題 | 更佳做法 |
|--------------|---------|-----------------|
| **Context Stuffing** | 浪費 token | 只檢索相關內容 |
| **Retry Forever** | 資源耗盡 | Circuit breaker |
| **Trust All Output** | Hallucination | 交叉驗證、grounding |
| **Single Model** | 單點故障 | Multi-provider |
| **No Observability** | 無從除錯 | 為所有環節建立 trace |
| **Infinite Agentic Loop** | Agent 持續迴圈卻沒有進展 | Max turns + Critic agent |
| **Over-trusting Computer-Use** | Agent 點錯 UI 元素 | Screenshot validation + HITL |
| **No CLAUDE.md / Manifest** | Agent 缺乏專案脈絡資訊 | 一律提供 coding manifest |
| **Thinking Mode Always On** | 成本高出 3–10 倍卻沒有收益 | 以 complexity classifier 控制 |

---

<a id="pattern-selection-guide"></a>
## 模式選擇指南

**開始新專案？**
1. 從 Basic RAG 開始
2. 當精確度重要時加入 reranking
3. 針對關鍵字密集內容加入 hybrid search

**需要可靠性？**
1. 從 retry + timeout 開始
2. 對外部呼叫加入 circuit breaker
3. 為關鍵路徑加入 fallback models

**在意成本？**
1. 先實作 semantic caching
2. 依查詢複雜度加入 model routing
3. 在延遲可接受時進行 batch

---

*詳細實作請參閱 [15-ai-design-patterns/](15-ai-design-patterns/)*
