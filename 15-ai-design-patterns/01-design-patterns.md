<a name="ai-design-patterns"></a>
# AI 設計模式

本章整理了建構 AI 系統的常見模式，類似於軟體工程中的設計模式。每種模式均包含使用時機、實作指南與取捨說明。

<a name="table-of-contents"></a>
## 目錄

- [RAG 模式](#rag-patterns)
- [代理人模式](#agent-patterns)
- [最佳化模式](#optimization-patterns)
- [可靠性模式](#reliability-patterns)
- [成本模式](#cost-patterns)
- [面試題目](#interview-questions)
- [參考資料](#references)

---

<a name="rag-patterns"></a>
## RAG 模式

<a name="pattern-naive-rag"></a>
### 模式：樸素 RAG

最簡單的 RAG 實作：

```
Query → Embed → Search → Top K → Stuff into prompt → Generate
```

**使用時機：**
- MVP 與原型開發
- 簡單問答
- 當檢索品質已足夠時

**限制：**
- 無重排序
- 無查詢增強
- 可能檢索到不相關的區塊

---

<a name="pattern-advanced-rag"></a>
### 模式：進階 RAG

具有多個階段的強化管道：

```
Query → Rewrite → Embed → Hybrid Search → Rerank → Filter → Generate
```

```python
class AdvancedRAG:
    async def query(self, user_query: str) -> str:
        # Step 1: Query rewriting
        enhanced_query = await self.rewrite_query(user_query)
        
        # Step 2: Hybrid retrieval
        semantic_results = await self.vector_search(enhanced_query, top_k=50)
        keyword_results = await self.bm25_search(enhanced_query, top_k=50)
        
        # Step 3: Fusion
        combined = self.reciprocal_rank_fusion(semantic_results, keyword_results)
        
        # Step 4: Reranking
        reranked = await self.rerank(enhanced_query, combined[:20])
        
        # Step 5: Generation with top results
        context = self.format_context(reranked[:5])
        return await self.generate(user_query, context)
```

**使用時機：**
- 生產系統
- 精確度至關重要時
- 複雜文件集

---

<a name="pattern-parent-child-retrieval"></a>
### 模式：父子檢索

檢索小型區塊，回傳較大的父區塊：

```
Document
    └── Parent chunk (2000 tokens)
            ├── Child chunk (200 tokens) ← Retrieve on this
            ├── Child chunk (200 tokens)
            └── Child chunk (200 tokens)
```

```python
class ParentChildRetriever:
    def __init__(self, vector_store):
        self.vector_store = vector_store
    
    async def retrieve(self, query: str, top_k: int = 5) -> list[str]:
        # Search on child chunks (more precise)
        child_results = await self.vector_store.search(
            query, 
            collection="child_chunks",
            top_k=top_k * 3
        )
        
        # Get unique parent chunks
        parent_ids = set(r.metadata["parent_id"] for r in child_results)
        
        # Return parent chunks (more context)
        parents = await self.get_parents(list(parent_ids)[:top_k])
        return parents
```

**使用時機：**
- 需要精確的檢索
- 需要生成時的上下文
- 文件結構具有層次性

---

<a name="pattern-self-rag"></a>
### 模式：Self-RAG

模型自行決定何時及要檢索什麼：

```python
class SelfRAG:
    async def generate(self, query: str) -> str:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = await self.assess_retrieval_need(query)
        
        if needs_retrieval:
            # Step 2: Retrieve
            context = await self.retrieve(query)
            
            # Step 3: Assess relevance
            relevant_context = await self.filter_relevant(query, context)
            
            # Step 4: Generate with context
            response = await self.generate_with_context(query, relevant_context)
            
            # Step 5: Self-critique
            is_supported = await self.check_support(response, relevant_context)
            if not is_supported:
                response = await self.regenerate(query, relevant_context)
        else:
            response = await self.generate_without_context(query)
        
        return response
```

**使用時機：**
- 混合知識（參數式 + 檢索式）
- 希望模型具有選擇性
- 研究與實驗

---

<a name="pattern-corrective-rag-crag"></a>
### 模式：修正式 RAG（CRAG）

評估並修正檢索品質：

```python
class CorrectiveRAG:
    async def query(self, user_query: str) -> str:
        # Initial retrieval
        docs = await self.retrieve(user_query)
        
        # Grade each document
        graded = []
        for doc in docs:
            grade = await self.grade_relevance(user_query, doc)
            graded.append((doc, grade))
        
        # Categorize results
        relevant = [d for d, g in graded if g == "relevant"]
        ambiguous = [d for d, g in graded if g == "ambiguous"]
        
        if len(relevant) >= 3:
            # Enough relevant docs
            context = relevant
        elif len(relevant) + len(ambiguous) >= 2:
            # Refine ambiguous docs
            refined = await self.refine_search(user_query, ambiguous)
            context = relevant + refined
        else:
            # Web search fallback
            web_results = await self.web_search(user_query)
            context = relevant + web_results
        
        return await self.generate(user_query, context)
```

**使用時機：**
- 文件語料庫不可靠
- 需要高精確度
- 可接受品質檢查帶來的延遲

---

<a name="agent-patterns"></a>
## 代理人模式

<a name="pattern-react"></a>
### 模式：ReAct

交錯式推理與行動：

```
Thought → Action → Observation → Thought → Action → Observation → Answer
```

請參閱 [代理人架構](../07-agentic-systems/01-agent-architectures.md) 以了解實作細節。

**使用時機：**
- 通用型代理人
- 可解釋的決策流程
- 中等複雜度任務

---

<a name="pattern-plan-and-execute"></a>
### 模式：計畫與執行

先建立計畫，再逐步執行：

```python
class PlanAndExecuteAgent:
    async def run(self, task: str) -> str:
        # Step 1: Create plan
        plan = await self.create_plan(task)
        
        # Step 2: Execute each step
        results = []
        for step in plan.steps:
            result = await self.execute_step(step, results)
            results.append(result)
            
            # Re-plan if needed
            if result.needs_replanning:
                plan = await self.replan(task, results)
        
        # Step 3: Synthesize final answer
        return await self.synthesize(task, results)
    
    async def create_plan(self, task: str) -> Plan:
        prompt = f"""
        Create a step-by-step plan to accomplish this task: {task}
        
        Return as JSON:
        {{
            "steps": [
                {{"id": 1, "description": "...", "tool": "..."}},
                ...
            ]
        }}
        """
        return await self.llm.generate(prompt)
```

**使用時機：**
- 複雜的多步驟任務
- 需要可見的執行計畫
- 任務可從分解中受益

---

<a name="pattern-criticverifier"></a>
### 模式：評論者／驗證者

一個代理人負責生成，另一個負責評論：

```python
class CriticPattern:
    async def generate_with_critique(self, task: str, max_iterations: int = 3) -> str:
        response = await self.generator.generate(task)
        
        for i in range(max_iterations):
            # Critique the response
            critique = await self.critic.evaluate(task, response)
            
            if critique.is_acceptable:
                break
            
            # Regenerate with feedback
            response = await self.generator.regenerate(
                task, 
                previous=response, 
                feedback=critique.feedback
            )
        
        return response
```

**使用時機：**
- 品質至關重要
- 可接受額外的延遲
- 任務具有明確的成功標準

---

<a name="pattern-hierarchical-agents"></a>
### 模式：階層式代理人

管理者將任務委派給專業工作者：

```python
class ManagerAgent:
    def __init__(self):
        self.workers = {
            "research": ResearchAgent(),
            "coding": CodingAgent(),
            "writing": WritingAgent()
        }
    
    async def run(self, task: str) -> str:
        # Decompose task
        subtasks = await self.decompose(task)
        
        # Assign to workers
        results = {}
        for subtask in subtasks:
            worker = self.workers[subtask.worker_type]
            results[subtask.id] = await worker.execute(subtask)
        
        # Synthesize results
        return await self.synthesize(task, results)
```

**使用時機：**
- 跨領域的複雜任務
- 不同子任務需要不同工具
- 具有平行化機會

---

<a name="optimization-patterns"></a>
## 最佳化模式

<a name="pattern-cascading-models"></a>
### 模式：層疊模型

路由至成本最低且足夠勝任的模型：

```python
class ModelCascade:
    def __init__(self):
        self.models = [
            ("gpt-4o-mini", 0.15),     # Cheapest
            ("gpt-4o", 2.50),           # Mid-tier
            ("claude-3.5-sonnet", 3.00) # Most capable
        ]
    
    async def generate(self, query: str) -> str:
        # Classify complexity
        complexity = await self.classify_complexity(query)
        
        if complexity == "simple":
            return await self.call_model("gpt-4o-mini", query)
        elif complexity == "medium":
            return await self.call_model("gpt-4o", query)
        else:
            return await self.call_model("claude-3.5-sonnet", query)
```

**使用時機：**
- 高查詢量
- 查詢複雜度變化大
- 成本最佳化優先

---

<a name="pattern-speculative-execution"></a>
### 模式：推測性執行

以小型模型起草，再以大型模型驗證：

```python
class SpeculativeExecution:
    async def generate(self, prompt: str, n_tokens: int = 5) -> str:
        output = []
        
        while len(output) < max_tokens:
            # Draft with small model
            draft = await self.draft_model.generate(
                prompt + "".join(output),
                n_tokens=n_tokens
            )
            
            # Verify with large model
            verified = await self.target_model.verify(
                prompt + "".join(output),
                draft
            )
            
            # Accept verified tokens
            output.extend(verified.accepted_tokens)
            
            if verified.is_complete:
                break
        
        return "".join(output)
```

**使用時機：**
- 對延遲敏感的應用
- 擁有對齊的起草模型
- 可預測的生成模式

---

<a name="pattern-caching-layers"></a>
### 模式：快取層

多層快取策略：

```python
class CachingLLM:
    def __init__(self):
        self.exact_cache = ExactMatchCache()
        self.semantic_cache = SemanticCache(threshold=0.95)
    
    async def generate(self, query: str) -> str:
        # Level 1: Exact match
        cached = await self.exact_cache.get(query)
        if cached:
            return cached
        
        # Level 2: Semantic similarity
        similar = await self.semantic_cache.get_similar(query)
        if similar:
            return similar
        
        # Cache miss: Generate
        response = await self.llm.generate(query)
        
        # Store in caches
        await self.exact_cache.set(query, response)
        await self.semantic_cache.set(query, response)
        
        return response
```

**使用時機：**
- 重複的相似查詢
- 降低成本優先
- 可容忍一定程度的資料陳舊

---

<a name="reliability-patterns"></a>
## 可靠性模式

<a name="pattern-retry-with-fallback"></a>
### 模式：重試並備援

```python
class RetryWithFallback:
    async def generate(self, query: str) -> str:
        providers = [
            ("openai", "gpt-4o"),
            ("anthropic", "claude-3.5-sonnet"),
            ("google", "gemini-1.5-pro")
        ]
        
        for provider, model in providers:
            try:
                return await self.call(provider, model, query)
            except RateLimitError:
                continue
            except ServiceError:
                continue
        
        # All providers failed
        raise AllProvidersUnavailable()
```

---

<a name="pattern-circuit-breaker"></a>
### 模式：斷路器

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: int = 60):
        self.failures = 0
        self.state = "closed"
        self.last_failure = None
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
    
    async def call(self, func, *args):
        if self.state == "open":
            if time.time() - self.last_failure > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()
        
        try:
            result = await func(*args)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise
```

---

<a name="pattern-bulkhead"></a>
### 模式：隔艙

隔離元件之間的故障：

```python
class BulkheadExecutor:
    def __init__(self, max_concurrent: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrent)
    
    async def execute(self, func, *args):
        async with self.semaphore:
            return await func(*args)

# Separate bulkheads for different operations
rag_bulkhead = BulkheadExecutor(max_concurrent=20)
agent_bulkhead = BulkheadExecutor(max_concurrent=5)
```

---

<a name="cost-patterns"></a>
## 成本模式

<a name="pattern-token-budget"></a>
### 模式：Token 預算

```python
class TokenBudget:
    def __init__(self, max_input: int, max_output: int):
        self.max_input = max_input
        self.max_output = max_output
    
    def constrain_input(self, messages: list[dict]) -> list[dict]:
        total_tokens = 0
        constrained = []
        
        for msg in reversed(messages):
            tokens = count_tokens(msg["content"])
            if total_tokens + tokens > self.max_input:
                break
            constrained.insert(0, msg)
            total_tokens += tokens
        
        return constrained
```

---

<a name="pattern-cost-tracking-decorator"></a>
### 模式：成本追蹤裝飾器

```python
def track_cost(model: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            start_tokens = get_token_count()
            result = await func(*args, **kwargs)
            end_tokens = get_token_count()
            
            cost = calculate_cost(model, end_tokens - start_tokens)
            metrics.record("llm_cost", cost, tags={"model": model})
            
            return result
        return wrapper
    return decorator

@track_cost("gpt-4o")
async def generate_response(query: str):
    return await llm.generate(query)
```

---

<a name="interview-questions"></a>
## 面試題目

<a name="q-describe-three-rag-patterns-and-when-to-use-each"></a>
### 問：描述三種 RAG 模式及各自的使用時機。

**優質回答範例：**

「我將介紹樸素 RAG、進階 RAG 與父子檢索。

**樸素 RAG** 是最簡單的：對查詢進行嵌入、搜尋向量、將前 K 筆結果填入提示、生成回答。我在 MVP 開發或檢索品質已足夠時使用它。實作快速，但沒有重排序或查詢增強。

**進階 RAG** 加入了多個階段：查詢改寫、混合搜尋（語意 + 關鍵字）、重排序與過濾。我在精確度至關重要的生產環境中使用它。重排序帶來的額外延遲（100–200 毫秒）相對於 10–15% 的精確度提升是值得的。

**父子檢索** 以小型區塊進行精確匹配，但回傳較大的父區塊以提供上下文。我在文件具有層次結構，且需要兼顧檢索精確度與生成上下文時使用它。

我選擇哪種模式取決於精確度要求、延遲預算與文件特性。我通常從樸素 RAG 建立基準，再迭代至進階 RAG。」

<a name="q-what-reliability-patterns-would-you-use-for-a-production-llm-system"></a>
### 問：在生產 LLM 系統中，您會採用哪些可靠性模式？

**優質回答範例：**

「我會實作多層可靠性機制：

**指數退避重試**，用於處理暫時性故障。LLM API 的速率限制與暫時性錯誤很常見。

**多供應商備援**，若 OpenAI 出現問題，自動路由至 Anthropic 或 Google。這需要抽象化 LLM 介面。

**斷路器**，避免持續請求故障中的服務。在 N 次失敗後開路，立即路由至備援，讓主要服務有時間恢復。

**優雅降級**，當所有供應商都失敗時。回傳快取的回應、顯示備援訊息，或排入佇列稍後處理，而非直接報錯。

**隔艙隔離**，防止某個元件的故障蔓延。代理人工作負載與 RAG 工作負載使用各自獨立的執行緒池。

**各層皆設超時**。LLM 呼叫可能會掛起；我設定積極的超時並妥善處理。

關鍵在於假設故障必然發生，並為此而設計，而非寄望於不會發生。」

---

<a name="references"></a>
## 參考資料

- Gao et al. "Retrieval-Augmented Generation for Large Language Models: A Survey" (2024)
- Yao et al. "ReAct: Synergizing Reasoning and Acting in Language Models" (2023)
- Microsoft Patterns for AI: https://learn.microsoft.com/azure/architecture/patterns/

---

*下一章：[反模式](02-anti-patterns.md)*
