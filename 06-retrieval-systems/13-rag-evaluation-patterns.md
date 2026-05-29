<a id="rag-evaluation-patterns"></a>
# RAG 評估模式

評估是 RAG 中最難、也最未被完全解決的問題。你可以在一天內建好一條 retrieval pipeline；但要知道它是否真的有效，往往要花上數週。業界如今已逐漸收斂到分層式評估策略：用 RAG Triad 檢查正確性、用元件層級指標除錯，以及用自動化 regression testing 守住正式環境安全。Langfuse、LangWatch、Braintrust 與 Arize Phoenix 都已提供原生 RAG eval recipe；選型時可依部署模式（self-hosted 或 SaaS）與是否需要 eval-gated 的 CI/CD blocking 來決定。

<a id="table-of-contents"></a>
## 目錄

- [RAG Triad](#the-rag-triad)
- [RAGAS framework 與指標](#ragas-framework)
- [元件層級評估](#component-level-evaluation)
- [以 LLM-as-Judge 評估 RAG](#llm-as-judge)
- [建立 Golden Test Set](#golden-test-sets)
- [自動化 Regression Testing](#regression-testing)
- [正式環境監控](#production-monitoring)
- [大規模評估成本](#cost-at-scale)
- [工具比較](#tools-comparison)
- [系統設計面試角度](#system-design-interview-angle)
- [參考資料](#references)

---

<a id="the-rag-triad"></a>
## RAG Triad

RAG Triad 是評估 RAG 系統的基礎框架。它把正確性拆成三個彼此獨立的維度，每一個維度都對應不同的失敗模式。

```
                          User Query
                              |
                              v
                    +-------------------+
                    |    RETRIEVER      |
                    +-------------------+
                              |
                   (1) Context Relevance
                    "Did we retrieve the
                     right documents?"
                              |
                              v
                    +-------------------+
                    |    GENERATOR      |
                    +-------------------+
                         /         \
            (2) Groundedness      (3) Answer Relevance
            "Is the answer         "Does the answer
             supported by           address the actual
             the context?"          question?"
                  |                       |
                  v                       v
             No hallucination       No tangential answers
```

<a id="dimension-1-context-relevance"></a>
### 維度 1：Context Relevance

**問題**：每個被取回的 chunk，是否真的與使用者查詢有關？

**可捕捉的問題**：錯誤檢索——向量搜尋回傳了錯主題的文件，或 query 本身有歧義，導致 retriever 猜錯方向。

**如何衡量**：
- 對每個 retrieved chunk 問：「這個 chunk 是否與回答該 query 有關？」
- 分數 =（相關 chunk 數量）/（取回 chunk 總數）
- 若分數為 0.3，代表 70% 的 retrieved context 都是雜訊，迫使 LLM 在無關內容中找針。

**為何重要**：低 context relevance 是大多數 RAG 失敗的根源。即使 generator 再完美，面對無關 context 也無法產出好答案。

<a id="dimension-2-groundedness-faithfulness"></a>
### 維度 2：Groundedness（Faithfulness）

**問題**：生成答案中的每個主張，是否都能被 retrieved context 支持？

**可捕捉的問題**：幻覺——LLM 產生了看似合理、但其實不在 retrieved documents 中的內容。

**如何衡量**：
- 將答案拆成個別 claim／statement。
- 對每個 claim，在 retrieved context 中尋找支持證據。
- 分數 =（有證據支持的 claim 數量）/（claim 總數）
- 若分數為 0.7，表示答案中有 30% 是 hallucinated。

**為何重要**：這是企業客戶最在意的指標。若 RAG 不忠於證據，還不如不用 RAG，因為它會產生聽起來很有把握、還帶著假引用的錯誤答案。

<a id="dimension-3-answer-relevance"></a>
### 維度 3：Answer Relevance

**問題**：最終答案是否真的回答了使用者的問題？

**可捕捉的問題**：答非所問——檢索沒有問題、答案也有根據，但就是沒有回答到問題本身。這在 retriever 找到「相關但不匹配」的內容時很常見。

**如何衡量**：
- 產生 N 個「如果這個答案是好答案，使用者可能問什麼」的假設問題。
- 衡量這些假設問題與原始 query 的語意相似度。
- 相似度高表示答案有對題。

**為何重要**：系統可能取回了相關 context，也忠實地摘要了它，但依然沒回答到重點。Answer relevance 就是抓這種問題。

<a id="triad-failure-modes"></a>
### Triad 失敗模式

| 失敗模式 | Context Relevance | Groundedness | Answer Relevance | 根因 |
|----------------|-------------------|-------------|-----------------|------------|
| 好的 RAG | 高 | 高 | 高 | 系統正常運作 |
| 錯誤檢索 | **低** | 高 | 低 | Embedding 或搜尋設定錯誤 |
| 幻覺 | 高 | **低** | 高 | LLM 忽略 context、prompt 有問題 |
| 答非所問 | 高 | 高 | **低** | Query 歧義、索引錯誤 |
| 完全失敗 | **低** | **低** | **低** | Pipeline 的根本性問題 |

---

<a id="ragas-framework-and-metrics"></a>
## RAGAS Framework 與指標

RAGAS（Retrieval Augmented Generation Assessment）是目前最廣泛採用的開源 RAG 評估框架，提供不需要 ground-truth answer 的 reference-free 指標。

<a id="core-ragas-metrics"></a>
### 核心 RAGAS 指標

```
  RAGAS Metric Suite (v0.2+)
  |
  +-- Retrieval Metrics
  |     +-- Context Precision: Are relevant docs ranked higher?
  |     +-- Context Recall: Did we find all relevant docs?
  |     +-- Context Entities Recall: Did we capture key entities?
  |     +-- Context Relevance: Is retrieved context pertinent?
  |
  +-- Generation Metrics
  |     +-- Faithfulness: Are claims supported by context?
  |     +-- Answer Relevance: Does the answer address the query?
  |     +-- Answer Correctness: Does the answer match ground truth?
  |     +-- Answer Similarity: Semantic overlap with reference answer
  |
  +-- Noise & Robustness
  |     +-- Noise Sensitivity: How much does irrelevant context hurt?
  |
  +-- Multi-Modal (2025+)
        +-- Multimodal Faithfulness: Claims supported by images + text?
        +-- Multimodal Relevance: Are retrieved images relevant?
```

<a id="how-ragas-faithfulness-works-under-the-hood"></a>
### RAGAS Faithfulness 底層如何運作

```
Step 1: Claim Extraction
  Answer: "Revenue grew 15% in Q3, driven by APAC expansion
           and the new enterprise tier launched in July."

  Claims:
    c1: "Revenue grew 15% in Q3"
    c2: "Growth was driven by APAC expansion"
    c3: "Growth was driven by the new enterprise tier"
    c4: "The enterprise tier was launched in July"

Step 2: Evidence Matching (per claim)
  c1: Found in Context chunk 3 --> SUPPORTED
  c2: Found in Context chunk 1 --> SUPPORTED
  c3: Not found in any context --> UNSUPPORTED
  c4: Context says "August" not "July" --> CONTRADICTED

Step 3: Score Calculation
  Faithfulness = supported / total = 2/4 = 0.50
```

<a id="how-ragas-context-precision-works"></a>
### RAGAS Context Precision 如何運作

```
  Retrieved chunks ranked by retriever score:
    Rank 1: Chunk about Q3 revenue    --> Relevant (v_1 = 1)
    Rank 2: Chunk about company history --> Not relevant (v_2 = 0)
    Rank 3: Chunk about Q3 expenses   --> Relevant (v_3 = 1)
    Rank 4: Chunk about office locations --> Not relevant (v_4 = 0)

  Context Precision@K:
    Precision@1 = 1/1 = 1.0
    Precision@2 = 1/2 = 0.5
    Precision@3 = 2/3 = 0.67
    Precision@4 = 2/4 = 0.5

  Average Precision = (1.0*1 + 0.5*0 + 0.67*1 + 0.5*0) / 2
                    = (1.0 + 0.67) / 2 = 0.835
```

<a id="ragas-vs-ground-truth-metrics"></a>
### RAGAS 與 Ground-Truth 指標比較

| 指標 | 需要 Ground Truth 嗎？ | 衡量內容 |
|--------|-------------------|------------------|
| Faithfulness | 否 | claim 是否被 context 支持 |
| Context Relevance | 否 | retrieved chunk 的相關性 |
| Answer Relevance | 否 | 答案是否回應 query |
| Context Recall | **是** | 是否覆蓋 reference answer |
| Answer Correctness | **是** | 是否與 reference answer 相符 |
| Answer Similarity | **是** | 與 reference 的語意重疊 |

**洞見**：一開始應先用 reference-free 指標（faithfulness、context relevance、answer relevance）快速迭代。等有 golden test set 後，再加入 ground-truth 指標做 regression testing。

---

<a id="component-level-evaluation"></a>
## 元件層級評估

RAG Triad 是端到端評估；元件層級評估則是把各個階段切開，精準找出失敗點。

<a id="retriever-evaluation"></a>
### Retriever 評估

```
  Query Set (100+ queries with known relevant documents)
        |
        v
  Run Retriever --> Retrieved docs per query
        |
        v
  Compare against ground truth relevance labels
        |
        v
  Metrics:
    +-- Recall@K: What fraction of relevant docs are in the top K?
    +-- MRR (Mean Reciprocal Rank): How high is the first relevant doc?
    +-- NDCG@K: Quality-weighted ranking metric
    +-- Precision@K: What fraction of top K are relevant?
```

**關鍵 Retriever 基準**：

| 指標 | 最低門檻 | 良好 | 優秀 |
|--------|------------------|------|-----------|
| Recall@10 | 0.70 | 0.85 | 0.95+ |
| MRR | 0.50 | 0.70 | 0.85+ |
| NDCG@10 | 0.50 | 0.70 | 0.85+ |
| Precision@5 | 0.40 | 0.60 | 0.80+ |

<a id="generator-evaluation"></a>
### Generator 評估

固定檢索 context，只變動 generation，將 generator 單獨抽出來評估。

```
  Fixed Context (known relevant chunks)
  + Query
        |
        v
  Run Generator --> Answer
        |
        v
  Metrics:
    +-- Faithfulness (RAGAS): Does it stay grounded?
    +-- Completeness: Does it cover all relevant info in context?
    +-- Conciseness: Is it appropriately brief?
    +-- Format Compliance: Does it follow the expected output format?
    +-- Citation Accuracy: Do citations point to the right chunks?
```

<a id="reranker-evaluation"></a>
### Reranker 評估

```
  Query + Initial retrieval results (e.g., top 100 from BM25)
        |
        v
  Run Reranker --> Reranked results
        |
        v
  Metrics:
    +-- NDCG improvement: Did reranking move relevant docs up?
    +-- Recall preservation: Did reranking lose any relevant docs?
    +-- Latency: What did reranking add to query time?
```

---

<a id="llm-as-judge-for-rag"></a>
## 以 LLM-as-Judge 評估 RAG

用一個 LLM 來評估另一個 LLM 的輸出，已成為目前主流的評估方式。它能擴展到人類無法負擔的規模，但也有已知偏誤。

<a id="how-it-works"></a>
### 運作方式

```
  Evaluation Prompt Template:
  +------------------------------------------------------------------+
  | You are evaluating a RAG system. Given:                           |
  | - User Query: {query}                                             |
  | - Retrieved Context: {context}                                    |
  | - Generated Answer: {answer}                                      |
  |                                                                    |
  | Rate the following on a scale of 1-5:                             |
  | 1. Faithfulness: Are all claims in the answer supported by        |
  |    the context? (1=hallucinated, 5=fully grounded)                |
  | 2. Relevance: Does the answer address the user's question?        |
  |    (1=off-topic, 5=directly answers)                              |
  | 3. Completeness: Does the answer cover all relevant info?         |
  |    (1=missing key info, 5=comprehensive)                          |
  |                                                                    |
  | Provide scores and brief justifications in JSON.                  |
  +------------------------------------------------------------------+
```

<a id="known-biases-and-mitigations"></a>
### 已知偏誤與緩解方式

| 偏誤 | 描述 | 緩解方式 |
|------|-------------|------------|
| **冗長偏誤** | LLM judge 偏好較長答案 | 依答案長度正規化分數；加入 conciseness penalty |
| **自我偏好** | GPT-4 較容易給 GPT-4 產出的答案高分 | judge model 與 generator 使用不同模型 |
| **位置偏誤** | A/B 比較時，第一個選項較常被評高 | 隨機化呈現順序 |
| **迎合偏誤** | judge 傾向同意被評估系統 | 使用具體標準的結構化 rubric |
| **寬鬆偏誤** | LLM 很少給低於 3/5 的分數 | 改用二元判定（pass/fail）而非 Likert 尺度 |

<a id="best-practices-for-llm-as-judge"></a>
### LLM-as-Judge 最佳實務

1. **用二元判定取代分數量表**：「這個 claim 是否被支持？YES/NO」比「請為支持程度打 1–5 分」更可靠。
2. **拆成原子評估**：一次只評估一個 claim 或一個維度。
3. **要求證據**：強制 judge 指出支持／反駁各 claim 的具體 context 片段。
4. **用人工一致性做校準**：拿 100+ 個案例同時給 LLM 與人工評審，計算 Cohen's Kappa，目標 > 0.7。
5. **使用手上最強的模型**：用 Claude Opus 或 GPT-4o 做 judge；不要用同一個產生答案的模型來評自己。

---

<a id="building-golden-test-sets"></a>
## 建立 Golden Test Set

Golden test set 是經過整理與版本化的（query, expected_context, expected_answer）三元組集合，可作為 regression testing 的 ground truth。

<a id="building-process"></a>
### 建立流程

```
  Step 1: Seed Collection
  +-------------------------------------------------------+
  | Source production queries (logs, support tickets)       |
  | Target: 200-500 diverse queries                        |
  | Coverage: all topics, question types, difficulty levels |
  +-------------------------------------------------------+
            |
            v
  Step 2: Synthetic Augmentation
  +-------------------------------------------------------+
  | Use RAGAS or DataMorgana to generate additional queries |
  | from your corpus:                                       |
  |   - Simple factual questions (40%)                     |
  |   - Multi-hop reasoning questions (25%)                |
  |   - Conditional/comparative questions (20%)            |
  |   - Adversarial/edge cases (15%)                       |
  +-------------------------------------------------------+
            |
            v
  Step 3: Human Annotation
  +-------------------------------------------------------+
  | For each query, annotate:                               |
  |   - Expected relevant document IDs (for retrieval eval) |
  |   - Reference answer (for generation eval)              |
  |   - Difficulty label (easy / medium / hard)             |
  |   - Category tags (topic, question type)                |
  +-------------------------------------------------------+
            |
            v
  Step 4: Versioning and Freezing
  +-------------------------------------------------------+
  | Store in version control (golden_set_v3.json)           |
  | FREEZE the set for each evaluation cycle                |
  | Never modify a frozen set -- create a new version       |
  +-------------------------------------------------------+
```

<a id="golden-set-composition-guidelines"></a>
### Golden Set 組成指南

| 問題類型 | 佔比 | 目的 |
|--------------|-----------|---------|
| 簡單事實題 | 40% | 基線：理論上應全部通過 |
| Multi-hop 推理 | 25% | 測試跨文件檢索 |
| 比較型問題 | 15% | 測試多份相關文件檢索 |
| 時間性問題 | 10% | 測試版本化／帶日期內容的處理 |
| 對抗型問題 | 10% | 測試魯棒性（不可回答、超出範圍） |

<a id="synthetic-test-generation-with-ragas"></a>
### 用 RAGAS 產生合成測試資料

```python
# Pseudocode: Generate synthetic test queries from your corpus
from ragas.testset.generator import TestsetGenerator
from ragas.testset.evolutions import simple, reasoning, multi_context

generator = TestsetGenerator.from_langchain(
    generator_llm=ChatOpenAI(model="gpt-4o"),
    critic_llm=ChatOpenAI(model="gpt-4o"),
)
testset = generator.generate_with_langchain_docs(
    documents=load_documents("./knowledge_base/"),
    test_size=200,
    distributions={simple: 0.4, reasoning: 0.35, multi_context: 0.25}
)
# CRITICAL: Always human-review synthetic data before using as ground truth
testset.to_pandas().to_csv("golden_set_draft_v4.csv")
```

**警告**：合成測試集只是起點，不是終點。一定要經過人工審查，避免你最終是在測試生成模型本身的偏差，而非系統品質。

---

<a id="automated-regression-testing"></a>
## 自動化 Regression Testing

每次 RAG pipeline 變更（新 embedding、chunk size 調整、prompt 修改、reranker 更換）在部署前都應進行自動化 regression testing。

<a id="cicd-integration"></a>
### CI/CD 整合

```
  PR (RAG change) --> CI: Load golden set --> Run pipeline --> Compute metrics
                          --> Compare vs. baseline --> FAIL if drop > 5%, WARN if > 2%
                          --> Post metrics table as PR comment
```

<a id="quality-gates"></a>
### 品質門檻

| 指標 | 絕對最低標準 | 退化門檻 |
|--------|-----------------|---------------------|
| Recall@10 | 0.85 | 相較 baseline 下滑 5% |
| MRR | 0.70 | 下滑 5% |
| Faithfulness | 0.80 | 下滑 3% |
| Answer Relevance | 0.75 | 下滑 5% |
| Answer Correctness | 0.70 | 下滑 5% |

任何指標低於絕對最低標準都應阻擋 PR；任何超過退化門檻的下降都應觸發警告，並標記出退步的具體查詢。

---

<a id="production-monitoring"></a>
## 正式環境監控

離線評估是必要條件，但不是充分條件。正式環境中的查詢和測試集不同，隨著語料改變，檢索品質也可能隨時間下降。

<a id="key-production-signals"></a>
### 關鍵正式環境訊號

| 訊號 | 偵測內容 | 衡量方法 |
|--------|----------------|----------------|
| **空檢索率** | 沒有相關結果的查詢 | top-1 similarity < threshold 的查詢比例 |
| **相似度分數漂移** | Embedding 或語料退化 | 追蹤平均 similarity；下跌即告警 |
| **Faithfulness 抽樣** | 正式環境幻覺率 | 對 5–10% 隨機樣本跑 LLM-as-judge |
| **使用者回饋關聯** | 指標是否反映真實品質 | 比較 thumbs-up/down 與自動化分數 |
| **Latency P99** | 效能退化 | 追蹤檢索 + 生成延遲 |
| **Token Usage** | 成本漂移 | 監控每次 query 的平均 context token |

<a id="retrieval-quality-drift"></a>
### 檢索品質漂移

當語料變了，但 embedding、chunk 或 prompt 沒有跟上，就會發生漂移。常見有四種情境：（1）新文件使用了不同詞彙，導致 embedding space 不匹配——應重新為受影響集合做 embedding；（2）使用者查詢模式轉向語料中不存在的主題——可透過空檢索率監控偵測；（3）過時內容回傳舊答案——需加入 freshness metadata，偏好較新文件；（4）embedding model 更新改變 similarity 分布——模型變更後需重新校準所有 threshold。

---

<a id="cost-of-evaluation-at-scale"></a>
## 大規模評估成本

LLM-as-judge 很強大，但也昂貴。理解其成本結構，是預算規劃的關鍵。

<a id="cost-per-query-full-rag-triad"></a>
### 每次查詢成本（完整 RAG Triad）

| 指標 | LLM 呼叫次數 | Tokens | GPT-4o 成本 | Claude Haiku 成本 |
|--------|-----------|--------|-------------|-------------------|
| Faithfulness | ~3（抽取 + 驗證） | ~3k | $0.0075 | $0.00075 |
| Context Relevance | ~5（每個 chunk） | ~2.5k | $0.00625 | $0.000625 |
| Answer Relevance | ~2（問題生成） | ~1.6k | $0.004 | $0.0004 |
| **完整 Triad** | **~10** | **~7k** | **~$0.018** | **~$0.002** |

<a id="scaling-strategy"></a>
### 擴展策略

| 評估類型 | 頻率 | 量級 | Judge Model | 月成本（每日 1 萬查詢） |
|----------------|-----------|--------|-------------|-------------------------------|
| **CI Regression** | 每次 PR | Golden set（500 queries） | GPT-4o | ~$9/次 |
| **Nightly Batch** | 每日 | 正式環境隨機 1k queries | Claude Haiku | ~$60/月 |
| **Production Sample** | 即時 | 5% 流量 | Claude Haiku | ~$300/月 |
| **Deep Audit** | 每週 | 完整 golden set + 分析 | GPT-4o | ~$36/月 |

**洞見**：在高流量正式環境抽樣中，使用 Claude Haiku 4.5 或 GPT-5.5-mini；在 CI regression 與 deep audit 這種準確度比成本更重要的情境，才使用 Claude Opus 4.7 或 GPT-5.5。

---

<a id="tools-comparison"></a>
## 工具比較

<a id="framework-overview"></a>
### Framework 總覽

| 工具 | 最適合 | Open Source | 核心優勢 | 核心弱點 |
|------|----------|------------|--------------|--------------|
| **RAGAS** | 快速 RAG 評估、合成資料 | 是 | Reference-free 指標、社群強 | 指標結果缺少解釋 |
| **DeepEval** | CI/CD 整合、LLM 的 TDD | 是 | pytest 相容、分數可自解釋 | 設定較重 |
| **TruLens** | RAG Triad 評估、可觀測性 | 是 | 提出 RAG Triad、追蹤好用 | 開發較不活躍 |
| **UpTrain** | 正式環境監控、漂移偵測 | 是 | 混合評估（LLM + heuristic）、漂移告警 | 排序準確性較低 |
| **Braintrust** | 團隊協作、實驗追蹤 | 商業產品 | 最佳 UI/UX、實驗比較強 | 高階功能需付費 |
| **LangSmith** | LangChain 生態、追蹤 | 商業產品 | 深度整合 LangChain、追蹤完整 | 綁定 LangChain 生態 |

<a id="when-to-use-what"></a>
### 何時該用哪個工具

```
  Starting a new RAG project?
    --> RAGAS for quick baseline metrics + synthetic test generation

  Adding RAG eval to CI/CD?
    --> DeepEval (pytest integration, quality gates as assertions)

  Need production monitoring?
    --> UpTrain or Braintrust (drift detection, alerting)

  Want end-to-end observability?
    --> LangSmith (if LangChain) or Braintrust (if framework-agnostic)

  Building custom eval pipeline?
    --> Roll your own with LLM-as-judge + the RAG Triad structure
```

<a id="custom-evaluator-pattern"></a>
### 自訂 Evaluator 模式

自訂 evaluator 的核心模式其實很簡單：對 triad 的每個維度使用二元 LLM-as-judge 呼叫，再將結果彙總。

```python
# Pseudocode: Core faithfulness evaluator (other dimensions follow the same pattern)

def evaluate_faithfulness(answer: str, context: str, judge) -> float:
    # Step 1: Extract atomic claims from the answer
    claims = judge.generate(f"List every factual claim as a JSON array:\n{answer}")

    # Step 2: Verify each claim against context (binary YES/NO)
    supported = sum(
        1 for claim in json.loads(claims)
        if "YES" in judge.generate(
            f"Is this claim supported by the context? YES or NO.\n"
            f"Claim: {claim}\nContext: {context}"
        ).upper()
    )
    return supported / max(len(json.loads(claims)), 1)
```

對 context relevance（逐 chunk 問「這和 query 有關嗎？」）與 answer relevance（產生假設問題，再衡量與原始 query 的相似度）也套用相同的 decompose-then-judge 模式。

---

<a id="system-design-interview-angle"></a>
## 系統設計面試角度

<a id="q-you-deployed-a-rag-system-and-users-report-that-answers-are-sometimes-wrong-how-do-you-systematically-diagnose-and-fix-the-problem"></a>
### Q：你已部署一個 RAG 系統，但使用者回報答案有時會出錯。你要如何系統化診斷並修復問題？

**強答範例：**

我會使用 RAG Triad 來隔離失敗模式：

1. **抽樣失敗查詢**：收集 50–100 個被使用者標記為錯誤答案的 query，依失敗類型分類。

2. **跑 triad**：
   - **Context Relevance 低？** → 檢索問題。系統抓錯文件。修法：檢查 embedding similarity score、確認查詢語言是否與文件語言一致、嘗試 hybrid search（BM25 + dense）、加入 reranker。
   - **Groundedness 低？** → 幻覺問題。LLM 在有不錯 context 的情況下仍自行編造。修法：強化 system prompt（「只能根據提供的 context 回答」）、降低 temperature、改用更聽指令的模型，或加入 citation requirement。
   - **Answer Relevance 低？** → 系統取回了相關內容，也忠實摘要，但沒回答真正的問題。修法：改善 query understanding（query rewriting、HyDE），加入 query classification，把查詢導到正確索引。

3. **建立 regression test**：把這 50 個失敗查詢標註出期望答案，加入 golden test set。之後所有 pipeline 變更都必須通過這些案例。

4. **建立持續監控**：對正式環境 5% 流量做自動評估抽樣。當 faithfulness 低於 0.80 或 context relevance 低於 0.60 時發出告警。

關鍵洞見是：「答案錯了」不是診斷，而是症狀。RAG Triad 能把模糊抱怨轉成具體且可行動的根因。

<a id="q-how-do-you-evaluate-a-rag-system-when-you-do-not-have-ground-truth-answers"></a>
### Q：當你沒有 ground-truth answer 時，要如何評估一個 RAG 系統？

**強答範例：**

這是最常見的真實世界情境。我會用三層方法：

**第 1 層：Reference-free 指標（第 1 天）**。RAGAS faithfulness 與 context relevance 不需要 ground truth。它們能告訴你系統是否在 hallucinate，以及檢索是否正常。你可以立刻對任何 query 執行。

**第 2 層：合成 golden set（第 1 週）**。使用 RAGAS TestsetGenerator，從語料產生合成的（query, answer）對。這會給你近似的 ground truth，以評估 answer correctness 與 context recall。再抽樣給人類審查，確認品質。

**第 3 層：來自正式環境的 golden set（第 1 個月）**。從 production log 中挖出使用者滿意度高的 query（thumbs-up、沒有後續追問），再由標註人員補上 reference answer。這會形成反映真實使用模式、而非合成分布的 golden set。

取捨在於準確度與速度。第 1 層能在數小時內給你訊號，但只是近似；第 3 層能提供真正的 ground truth，但需要數週。最佳做法是三層並行，先用第 1 層取得即時回饋。

<a id="q-your-rag-evaluation-pipeline-costs-500day-in-llm-judge-calls-how-do-you-reduce-it"></a>
### Q：你的 RAG 評估 pipeline 每天要花 $500 在 LLM judge 呼叫上。你會如何降低成本？

**強答範例：**

有四個策略，依影響力排序：

1. **分層 judge model**：正式環境抽樣（90% 流量）用 Claude Haiku（$0.002/query），CI regression test 與每週 deep audit 才用 GPT-4o（$0.018/query）。光這一步就能把成本砍掉 80%。

2. **聰明抽樣**：不要評估每一筆 query。對正式環境流量抽樣 5%，並依 query 類型與使用者分群做分層抽樣。CI 只跑 golden set（500 queries），不要跑完整 synthetic set。

3. **快取**：很多正式環境 query 很相似。可將（query, context, answer）tuple 做 hash，快取評估結果。相同或近似相同輸入可直接重用分數。

4. **Heuristic 預過濾**：呼叫 LLM judge 前先做便宜的 heuristic 檢查。若答案包含「I don't know」，或與 context 完全沒有重疊（ROUGE-L < 0.1），就跳過昂貴的 faithfulness 評估，直接給分。

目標是把評估預算花在最有訊號的地方：那些模糊、接近邊界、真正需要 LLM judge 細膩判斷的案例。

---

<a id="references"></a>
## 參考資料

- Es et al. "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023, arXiv:2309.15217)
- TruLens. "The RAG Triad" (2024)
- DeepEval. "Using the RAG Triad for RAG Evaluation" (2025)
- Confident AI. "RAG Evaluation Metrics" (2025)
- Microsoft. "The Path to a Golden Dataset" (2025)
- Prem AI. "RAG Evaluation: Metrics, Frameworks & Testing" (2026)

---

*上一篇：[Multi-Modal RAG](12-multimodal-rag.md) | 下一篇：Coming Soon*
