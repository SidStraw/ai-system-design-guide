<a id="contextual-retrieval"></a>
# Contextual Retrieval

Contextual Retrieval 是一種在 ingestion 階段執行的技術，用來解決 RAG 失敗的頭號原因：**chunk 一旦脫離原始文件，就會失去語意**。這個方法由 Anthropic 在 2024 年底率先推廣，如今已成為高精度檢索的正式環境標準。Anthropic 自家的量測顯示：僅用 hybrid search 就能讓檢索失敗減少 49%；若再結合 reranking，則可減少 67%。

<a id="table-of-contents"></a>
## 目錄

- [問題：Context Dilution](#the-problem-context-dilution)
- [Contextual Retrieval 如何運作](#how-contextual-retrieval-works)
- [Contextual Embeddings](#contextual-embeddings)
- [Contextual BM25](#contextual-bm25)
- [完整 Pipeline：Hybrid + Reranking](#the-full-pipeline-hybrid-reranking)
- [實作模式](#implementation-patterns)
- [成本考量](#cost-considerations)
- [Contextual Retrieval 與其他方法的比較](#contextual-retrieval-vs-other-approaches)
- [正式環境架構](#production-architecture)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-problem-context-dilution"></a>
## 問題：Context Dilution

當我們為 RAG 對文件做 chunking 時，單一 chunk 會失去原本包圍它、賦予意義的上下文。

**Context Dilution 範例：**

```
Original Document: "Acme Corp Q3 2025 Financial Report"
  Section 4: Product Pricing

  "The Standard plan costs $200/month. The Enterprise
   plan includes SSO and audit logs for $800/month."

-------- After Chunking --------

Chunk 17: "It costs $200/month."
Chunk 18: "The Enterprise plan includes SSO and audit
           logs for $800/month."
```

**Chunk 17 的問題**：如果使用者搜尋「Acme Standard plan 多少錢？」，很可能會漏掉這個 chunk，因為它完全沒有提到「Acme」、「Standard」或「plan」。`It costs $200/month` 的 embedding 在語意上離這個 query 很遠。

**洞見**：Anthropic 的研究顯示，傳統 chunking 會讓 top-20 檢索結果出現 **5.7% 的檢索失敗率**。也就是說，大約每 18 個 query 就有 1 個，即使相關資訊明明存在於 knowledge base 中，也抓不出來。

---

<a id="how-contextual-retrieval-works"></a>
## Contextual Retrieval 如何運作

核心概念很簡單：**在為 chunk 建立 embedding 之前，先加上一段簡短的上下文字串，說明這個 chunk 在完整文件中的意義。**

```
┌──────────────────────────────────────────────────┐
│              TRADITIONAL CHUNKING                │
│                                                  │
│  Document ──► Split ──► Chunks ──► Embed ──► DB  │
│                                                  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              CONTEXTUAL RETRIEVAL                            │
│                                                              │
│  Document ──► Split ──► Chunks ──┐                           │
│                                  ├──► Contextualize ──►      │
│  Document (full) ───────────────┘    (LLM call per chunk)    │
│                                                              │
│  ──► Contextual Chunks ──► Embed ──► DB                      │
│                            + BM25 Index                      │
└──────────────────────────────────────────────────────────────┘
```

**Contextualization 步驟**會把完整文件 + 單一 chunk 一起送給 LLM，prompt 如下：

```
<document>
{{WHOLE_DOCUMENT}}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{{CHUNK_CONTENT}}
</chunk>

Please give a short succinct context to situate this chunk
within the overall document for the purposes of improving
search retrieval of the chunk. Answer only with the succinct
context and nothing else.
```

**Chunk 17 的結果**：

```
Before: "It costs $200/month."

After:  "This chunk is from the Acme Corp Q3 2025 Financial
         Report, Section 4 on Product Pricing. It describes
         the cost of the Standard plan.
         It costs $200/month."
```

現在，這個 chunk 的 embedding 已包含「Acme」、「Standard plan」與「Product Pricing」等使用者自然會搜尋的詞彙。

---

<a id="contextual-embeddings"></a>
## Contextual Embeddings

Contextual Embeddings 是第一個子技術：不是嵌入原始 chunk，而是嵌入 contextualized chunk。

<a id="how-it-improves-retrieval"></a>
### 它如何改善檢索

| 情境 | 原始 Chunk Embedding | Contextual Embedding |
|----------|--------------------|-----------------------|
| 使用者問「Acme pricing」 | 會漏掉「It costs $200」 | 可匹配「Acme...Standard plan...costs $200」 |
| 使用者問「SSO features」 | 只匹配「SSO and audit logs」 | 還會帶上「Enterprise plan」的補充上下文 |
| 使用者問「Q3 financials」 | 不會匹配（沒提到 Q3） | 可透過前綴的「Q3 2025 Financial Report」匹配 |

**效能**：單靠 Contextual Embeddings，就能把 top-20 檢索失敗率從 **5.7% 降到 3.7%**，也就是 **35% 的失敗減幅**。

<a id="the-vector-space-shift"></a>
### 向量空間位移

```
                    ▲ Dimension 2
                    │
                    │    ● "Acme pricing" (query)
                    │         \
                    │          \  close (contextual)
                    │           \
                    │            ● Contextualized chunk
                    │
                    │                          ● Raw chunk "It costs $200"
                    │                            (far from query)
                    │
                    └─────────────────────────────► Dimension 1
```

---

<a id="contextual-bm25"></a>
## Contextual BM25

第二個子技術，是把同樣的 contextualization 用在 enriched chunks 上建立 **BM25 keyword index**。

<a id="why-bm25-still-matters"></a>
### 為什麼 BM25 仍然重要

Dense embeddings 很擅長語意相似度，但對下列情況仍不理想：
- **精確詞彙**：產品 ID、版本號、縮寫
- **稀有 token**：embedding 模型較難充分表達的領域術語
- **專有名詞**：公司名、人名、地名

**範例**：使用者搜尋「Widget-X pricing」時，原始 chunk `It costs $200/month` 在 BM25 上完全不會命中，因為裡面根本沒有「Widget-X」。但若有 contextual BM25，前綴上下文就會包含「Widget-X」，從而啟用 BM25 命中。

<a id="performance-gains-cumulative"></a>
### 效能提升（累積）

| 配置 | 失敗率 | 相對基準的降幅 |
|---------------|-------------|----------------------|
| 傳統 embeddings（基準） | 5.7% | -- |
| 僅 Contextual Embeddings | 3.7% | 35% |
| Contextual Embeddings + Contextual BM25 | 2.9% | **49%** |
| Contextual Embeddings + Contextual BM25 + Reranking | 1.9% | **67%** |

**重點**：Contextual Embeddings + Contextual BM25 的組合，是你能對 RAG pipeline 做出的最高槓桿單一改動。再加上 reranker，就能把失敗率降低 67%。

---

<a id="the-full-pipeline-hybrid-reranking"></a>
## 完整 Pipeline：Hybrid + Reranking

正式環境級的 Contextual Retrieval pipeline 有四個階段：

```
┌─────────────────────────────────────────────────────────────────┐
│                     INGESTION PIPELINE                          │
│                                                                 │
│  1. Chunk documents (recursive, 300-500 tokens)                 │
│  2. For each chunk:                                             │
│     a. Send (full_doc + chunk) to LLM                           │
│     b. Get context string (50-100 tokens)                       │
│     c. Prepend context to chunk                                 │
│  3. Embed contextualized chunks ──► Vector DB                   │
│  4. Index contextualized chunks ──► BM25 Index                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     QUERY PIPELINE                              │
│                                                                 │
│  User Query                                                     │
│      │                                                          │
│      ├──► Vector Search (Top 50) ──┐                            │
│      │                             ├──► RRF Fusion (Top 25)     │
│      └──► BM25 Search (Top 50)  ──┘         │                   │
│                                             ▼                   │
│                                      Reranker (Top 5)           │
│                                             │                   │
│                                             ▼                   │
│                                     LLM Generation              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

<a id="reciprocal-rank-fusion-rrf-for-combining-results"></a>
### 使用 Reciprocal Rank Fusion（RRF）合併結果

這裡使用的 RRF 技術，與標準 hybrid search 中的做法相同：

```
RRF_Score(doc) = sum( 1 / (k + rank_in_list) )
                 for each list where doc appears

k = 60 (standard smoothing constant)
```

---

<a id="implementation-patterns"></a>
## 實作模式

<a id="pattern-1-basic-contextual-retrieval-python"></a>
### 模式 1：基礎 Contextual Retrieval（Python）

```python
import anthropic
from typing import List

client = anthropic.Anthropic()

CONTEXT_PROMPT = """<document>
{document}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{chunk}
</chunk>

Please give a short succinct context to situate this chunk
within the overall document for the purposes of improving
search retrieval of the chunk. Answer only with the succinct
context and nothing else."""


def contextualize_chunk(
    full_document: str,
    chunk: str,
    model: str = "claude-sonnet-4-20250514"
) -> str:
    """Generate context for a single chunk."""
    response = client.messages.create(
        model=model,
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": CONTEXT_PROMPT.format(
                document=full_document,
                chunk=chunk
            )
        }]
    )
    context = response.content[0].text
    return f"{context}\n\n{chunk}"


def process_document(document: str, chunks: List[str]) -> List[str]:
    """Contextualize all chunks in a document."""
    contextualized = []
    for chunk in chunks:
        ctx_chunk = contextualize_chunk(document, chunk)
        contextualized.append(ctx_chunk)
    return contextualized
```

<a id="pattern-2-cost-optimized-with-prompt-caching"></a>
### 模式 2：使用 Prompt Caching 進行成本最佳化

最大的成本來源，是每個 chunk 都要重送一次完整文件。**Prompt Caching** 就是用來解決這個問題：

```python
def contextualize_with_caching(
    full_document: str,
    chunks: List[str],
    model: str = "claude-sonnet-4-20250514"
) -> List[str]:
    """
    Use prompt caching so the full document is only
    processed once across all chunks.
    """
    results = []

    for chunk in chunks:
        response = client.messages.create(
            model=model,
            max_tokens=200,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"<document>\n{full_document}\n</document>",
                        "cache_control": {"type": "ephemeral"}
                    },
                    {
                        "type": "text",
                        "text": (
                            f"<chunk>\n{chunk}\n</chunk>\n\n"
                            "Please give a short succinct context to "
                            "situate this chunk within the overall "
                            "document for the purposes of improving "
                            "search retrieval of the chunk. Answer "
                            "only with the succinct context and "
                            "nothing else."
                        )
                    }
                ]
            }]
        )
        context = response.content[0].text
        results.append(f"{context}\n\n{chunk}")

    return results
```

**Prompt Caching 的成本影響**：對於一份 10,000 token、切成 30 個 chunks 的文件來說，prompt caching 可把 contextualization 成本降低高達 **90%**，因為文件前綴會在第一次呼叫後被快取。

<a id="pattern-3-contextual-chunk-headers-lightweight-alternative"></a>
### 模式 3：Contextual Chunk Headers（輕量替代方案）

如果基於 LLM 的 contextualization 太昂貴，可以改用 **Contextual Chunk Headers（CCH）** 作為決定性的替代方案：

```python
def add_chunk_headers(
    document_title: str,
    section_hierarchy: List[str],
    chunk: str
) -> str:
    """
    Prepend document and section metadata to the chunk.
    No LLM call required -- purely structural.
    """
    header_parts = [f"Document: {document_title}"]

    for i, section in enumerate(section_hierarchy):
        prefix = "  " * i
        header_parts.append(f"{prefix}Section: {section}")

    header = "\n".join(header_parts)
    return f"{header}\n\n{chunk}"

# Example usage:
contextualized = add_chunk_headers(
    document_title="Acme Corp Q3 2025 Financial Report",
    section_hierarchy=["Finance", "Product Pricing", "Standard Plan"],
    chunk="It costs $200/month."
)

# Result:
# Document: Acme Corp Q3 2025 Financial Report
#   Section: Finance
#     Section: Product Pricing
#       Section: Standard Plan
#
# It costs $200/month.
```

**何時使用 CCH、何時使用 LLM Contextualization：**

| 因素 | Chunk Headers（CCH） | LLM Contextualization |
|--------|--------------------|-----------------------|
| **成本** | 免費（無 LLM 呼叫） | 每 1M tokens 約 $1-5 |
| **品質** | 對結構化文件效果佳 | 對所有文件都很強 |
| **速度** | 即時 | 每個 chunk 50-200ms |
| **最適合** | Markdown、HTML、具清楚標題的 PDF | 非結構化文字、法律、醫療 |

---

<a id="cost-considerations"></a>
## 成本考量

<a id="contextualization-costs"></a>
### Contextualization 成本

對一個含有 10,000 個 chunks（平均每個 400 tokens）的 knowledge base：

| 模型 | 每個 Chunk 成本 | 總成本 | 品質 |
|-------|---------------|------------|---------|
| Claude Haiku（快、便宜） | ~$0.0003 | ~$3 | Good |
| Claude Sonnet（平衡） | ~$0.002 | ~$20 | Very Good |
| Claude Opus（最高品質） | ~$0.01 | ~$100 | Excellent |

**最佳實務**：對 contextualization 使用 Haiku（或其他快速便宜的模型）即可。因為這些 context strings 很短、偏事實性，不需要 frontier model。再搭配 prompt caching，就能把反覆傳入的文件本文成本再降約 90%。

<a id="when-to-use-contextual-retrieval"></a>
### 何時使用 Contextual Retrieval

**適合使用的情況：**
- 你的語料包含許多碎片化文件，chunk 單獨存在時會失去意義
- 你有 embedding 模型難以掌握的領域術語
- 你的檢索失敗率超過 3-5%
- 你能負擔一次性的 ingestion 成本

**不適合使用的情況：**
- 你的 chunks 已經自成一體（例如 FAQ 配對、產品描述）
- 你的語料非常小（<100 個 chunks）——直接用 long-context 即可
- 你需要即時 ingestion（每份文件 <1 秒），而且無法做 batch

---

<a id="contextual-retrieval-vs-other-approaches"></a>
## Contextual Retrieval 與其他方法的比較

| 方法 | 運作方式 | 檢索改善 | 成本 | 複雜度 |
|----------|-------------|----------------------|------|------------|
| **Naive Chunking** | 固定大小切分，直接嵌入原文 | 基準 | 無 | 低 |
| **Chunk Headers（CCH）** | 在前面加上文件／章節標題 | 10-20% | 無 | 低 |
| **Contextual Retrieval** | 每個 chunk 由 LLM 產生 context | 35-49% | 每 10k chunks 約 $3-20 | 中 |
| **Contextual + Reranking** | 上述方法 + cross-encoder rerank | 67% | 每 10k chunks 約 $5-30 | 中高 |
| **HyDE** | 在 query time 產生假想文件 | 20-40% | 每 query 的 LLM 成本 | 中 |
| **Parent-Child Chunking** | 嵌入 children、檢索 parents | 15-30% | 無 | 中 |

**關鍵差異**：Contextual Retrieval 是 **ingestion-time** 技術（付一次），HyDE 則是 **query-time** 技術（每次查詢都付）。對高流量系統而言，Contextual Retrieval 的攤提效果好得多。

<a id="contextual-retrieval-vs-late-chunking"></a>
### Contextual Retrieval 與 Late Chunking 的比較

**Late Chunking**（Jina，2024）是相關但不同的方法：

```
Contextual Retrieval:
  Chunk ──► LLM adds context ──► Embed enriched chunk

Late Chunking:
  Full doc ──► Long-context embed model ──► Token embeddings
  ──► THEN chunk the token embeddings (preserving context)
```

Late Chunking 需要 long-context embedding model（例如 Jina v3），完全不需要 LLM calls。它是透過 embedding model 的 attention mechanism 保留上下文，而不是明確地在文字前加前綴。取捨在於：Late Chunking 只幫 dense retrieval，對 BM25 搜尋沒有幫助。

---

<a id="production-architecture"></a>
## 正式環境架構

<a id="reference-architecture-contextual-rag-at-scale"></a>
### 參考架構：大規模 Contextual RAG

```
┌─────────────────────────────────────────────────────────────────────┐
│                     INGESTION SERVICE                               │
│                                                                     │
│  Document Store ──► Chunker ──► Contextualization Queue             │
│                       │              │                              │
│                       │         ┌────┴────┐                         │
│                       │         │ Workers  │ (N parallel LLM calls) │
│                       │         │ + Cache  │                        │
│                       │         └────┬────┘                         │
│                       │              │                              │
│                       ▼              ▼                              │
│                  Raw Chunks    Contextualized Chunks                 │
│                       │              │                              │
│                       │         ┌────┴────┐                         │
│                       │         │ Embed + │                         │
│                       │         │ BM25    │                         │
│                       │         └────┬────┘                         │
│                       │              │                              │
│                       ▼              ▼                              │
│                  Metadata DB    Vector DB + BM25 Index               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     QUERY SERVICE                                   │
│                                                                     │
│  Query ──► [Vector Search] + [BM25 Search]                          │
│                    │               │                                │
│                    └───── RRF ─────┘                                │
│                           │                                         │
│                      Top 25 chunks                                  │
│                           │                                         │
│                      Reranker (Cohere, Cross-Encoder)               │
│                           │                                         │
│                      Top 5 chunks                                   │
│                           │                                         │
│                      LLM Generation                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

<a id="scaling-considerations"></a>
### 擴充性考量

| 關注點 | 解法 |
|---------|----------|
| **Ingestion 吞吐量** | 以 async workers 平行 LLM calls（50-100 個並行） |
| **文件更新** | 只重新 contextualize 有變更的 chunks；分開存 raw 與 context |
| **大規模成本** | 使用 Haiku + prompt caching；依文件大小批次處理 |
| **品質監控** | 抽樣 1% chunks，由人工評估 context 品質 |
| **索引一致性** | 以文件為單位，原子性更新 vector DB + BM25 index |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-explain-anthropics-contextual-retrieval-when-would-you-use-it-and-when-would-you-skip-it"></a>
### Q：請解釋 Anthropic 的 Contextual Retrieval。什麼情況會使用，什麼情況會跳過？

**強答：**
Contextual Retrieval 解決的是 RAG 中的「context dilution」問題。當文件被切成 chunks 後，單一 chunk 會失去原本賦予它意義的上下文——例如一個 chunk 只寫著「It costs $200」，若不知道是什麼東西要 200 美元，這段資訊幾乎沒有價值。這個技術會在 ingestion 階段，讓 LLM 為每個 chunk 生成一段 50-100 token 的短上下文，說明它在整份文件中的位置與意義。接著，這段 context 會先被 prepend 到 chunk 前面，再做 embedding 與 BM25 indexing。

關鍵結果如下：單靠 Contextual Embeddings 就能減少 35% 的檢索失敗；加上 Contextual BM25 可達 49%；再加上 reranker 則可達 67%。

如果 chunk 常常單獨失去意義——例如法律合約、財報、技術手冊——我會使用它。若 chunks 本來就自成一體（FAQ、商品卡片），或語料小到可以直接做 long-context RAG，我就會跳過。

<a id="q-a-knowledge-base-of-50000-documents-needs-contextual-retrieval-how-do-you-manage-the-ingestion-cost"></a>
### Q：一個有 50,000 份文件的 knowledge base 需要 Contextual Retrieval。你如何管理 ingestion 成本？

**強答：**
有三個策略：
1. **模型選型**：用小而快的模型（如 Claude Haiku 級別）做 contextualization。輸出只是短小、偏事實的文字，不是創意寫作——frontier model 只會增加成本，不會帶來相應品質收益。
2. **Prompt caching**：在所有 chunk contextualization calls 間快取完整文件文字。對 10,000 token、30 個 chunks 的文件來說，這可讓輸入 token 成本下降約 90%。
3. **分層策略**：不是每份文件都需要 LLM contextualization。對於結構良好的文件（Markdown、HTML、有標題的文件），可用決定性的 Contextual Chunk Headers（prepend 文件標題 + 章節階層），成本為零。只把 LLM contextualization 留給非結構化或語意模糊的文件。

<a id="q-how-does-contextual-retrieval-compare-to-hyde-for-improving-retrieval-quality"></a>
### Q：Contextual Retrieval 與 HyDE 在提升檢索品質上有何不同？

**強答：**
它們解的是同一個問題的不同面向。Contextual Retrieval 在 ingestion 時增豐的是 **documents**（付一次），HyDE 在搜尋時增豐的是 **queries**（每次查詢都付）。對一個每天 10,000 次查詢、語料有 50,000 個 chunks 的系統來說，Contextual Retrieval 會便宜得多，因為 ingestion 成本可以被攤提。HyDE 也帶有幻覺風險——假想文件可能把錯誤資料拉進來。實務上，最強的系統通常兩者並用：用 Contextual Retrieval 增豐文件端，再用 HyDE（或 multi-query expansion）處理需要 query 端補強的複雜查詢。

---

<a id="references"></a>
## 參考資料
- Anthropic. "Contextual Retrieval" (September 2024)
- Jina AI. "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models" (2024)
- Voyage AI. "voyage-context-3: Contextualized Chunk Embeddings" (2025)
- NirDiamant. "RAG Techniques: Contextual Chunk Headers" (GitHub, 2024)

---

*上一節：[Advanced Retrieval Patterns](09-advanced-retrieval-patterns.md) | 下一節：[Late Interaction & ColBERT](11-late-interaction-colbert.md)*

