<a id="whiteboard-exercises-for-ai-system-design"></a>
# AI 系統設計白板練習

本章提供 AI 面向面試中常見系統設計題的詳細解析。每個練習都包含完整題目敘述、結構化解題方式，以及能區分優秀候選人的討論重點。

<a id="table-of-contents"></a>
## 目錄

- [練習 1：企業級 RAG 系統](#exercise-1-enterprise-rag-system)
- [練習 2：客戶支援聊天機器人](#exercise-2-customer-support-chatbot)
- [練習 3：程式碼審查助理](#exercise-3-code-review-assistant)
- [練習 4：文件處理管線](#exercise-4-document-processing-pipeline)
- [練習 5：即時內容審核](#exercise-5-real-time-content-moderation)
- [練習 6：多租戶 AI 平台](#exercise-6-multi-tenant-ai-platform)
- [練習 7：大規模語意搜尋](#exercise-7-semantic-search-at-scale)
- [白板練習技巧](#tips-for-whiteboard-exercises)

---

<a id="exercise-1-enterprise-rag-system"></a>
## 練習 1：企業級 RAG 系統

<a id="problem-statement"></a>
### 題目敘述

為一家大型企業設計一個以 RAG 為基礎的知識助理，需求如下：

- 來自多個來源的 1,000 萬份文件（SharePoint、Confluence、Google Drive、內部 wiki）
- 5 萬名員工，具備角色式存取控制
- 文件會持續更新
- 查詢時必須遵守文件權限
- 95% 查詢的回應時間需低於 3 秒
- 支援多語言（English、Spanish、Mandarin）

<a id="time-allocation-35-minutes"></a>
### 時間分配（35 分鐘）

| 階段 | 時間 | 重點 |
|-------|------|-------|
| 釐清問題 | 3 分鐘 | 範圍、優先順序、限制 |
| 高階架構 | 7 分鐘 | 元件與資料流 |
| 資料管線 | 8 分鐘 | 擷取、切塊、索引 |
| 查詢管線 | 8 分鐘 | 檢索、生成、權限 |
| 可靠性與擴展性 | 5 分鐘 | 故障處理、擴展 |
| 評估 | 4 分鐘 | 指標與監控 |

<a id="solution-walkthrough"></a>
### 解題走讀

<a id="clarification-questions"></a>
#### 釐清問題

```
1. 文件大小分布為何？（PDF、wiki、程式碼？）
2. 權限多久變更一次？（會影響快取策略）
3. 需要對話歷史，還是單輪問答即可？
4. 準確度門檻是什麼？（可以回答「我不知道」嗎？）
5. 是否有合規要求？（稽核、資料駐留）
```

<a id="high-level-architecture"></a>
#### 高階架構

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DATA PLANE                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────────┐│
│  │   Connectors │───▶│   Processor  │───▶│       Vector Database        ││
│  │ (SP,GD,Conf) │    │ (chunk,embed)│    │  (Pinecone/Qdrant/Weaviate)  ││
│  └──────────────┘    └──────────────┘    └──────────────────────────────┘│
│                                                      ▲                   │
│                                                      │ sync              │
│  ┌──────────────────────────────────────────────────┼───────────────────┐│
│  │                    Permission Service            │                   ││
│  └──────────────────────────────────────────────────┼───────────────────┘│
└─────────────────────────────────────────────────────┼───────────────────┘
                                                      │
┌─────────────────────────────────────────────────────┼───────────────────┐
│                          QUERY PLANE                │                   │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────┴──────┐             │
│  │    User      │───▶│  Query API   │───▶│   Retriever    │             │
│  │  Interface   │    │              │    │ (+ perm filter)│             │
│  └──────────────┘    └──────────────┘    └────────┬───────┘             │
│                             │                      │                     │
│                             ▼                      ▼                     │
│                      ┌──────────────┐    ┌──────────────┐               │
│                      │   Reranker   │◀───│   Chunks     │               │
│                      └──────┬───────┘    └──────────────┘               │
│                             │                                            │
│                             ▼                                            │
│                      ┌──────────────┐    ┌──────────────┐               │
│                      │  Generator   │───▶│   Response   │               │
│                      │    (LLM)     │    │  + Citations │               │
│                      └──────────────┘    └──────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

<a id="data-pipeline-deep-dive"></a>
#### 資料管線深入解析

**1. 連接器：**
```
每個來源都有專屬連接器：
- SharePoint：使用 Graph API 搭配 delta sync
- Confluence：使用 REST API 搭配 webhook
- Google Drive：使用 Drive API 搭配 push notification

連接器職責：
- 擷取文件內容與中繼資料
- 追蹤變更事件（create、update、delete）
- 從來源系統擷取權限資訊
- 正規化為通用文件 schema
```

**2. 文件 Schema：**
```json
{
  "doc_id": "uuid",
  "source": "sharepoint|confluence|gdrive",
  "source_id": "original_id_in_source",
  "title": "string",
  "content": "string",
  "content_type": "pdf|html|docx|md",
  "language": "en|es|zh",
  "permissions": {
    "users": ["user_id_1", "user_id_2"],
    "groups": ["group_id_1"],
    "visibility": "private|internal|public"
  },
  "metadata": {
    "author": "string",
    "created_at": "timestamp",
    "updated_at": "timestamp",
    "path": "folder/path"
  }
}
```

**3. 切塊策略：**
```
面對混合型文件，可採用自適應切塊：

- Markdown/HTML：依標題做語意切塊
- PDF：使用 document AI 做版面感知切塊
- Wiki 頁面：依章節切塊

切塊參數：
- 目標大小：512 tokens
- 重疊：50 tokens
- 保留：標題、表格、程式碼區塊

每個 chunk 會繼承母文件的權限。
```

**4. Embedding：**
```
多語需求代表可考慮：
- 模型：Cohere embed-v3（多語、品質佳）
- 替代方案：OpenAI text-embedding-3-large

批次 embedding：
- 每批處理 100 個 chunks
- 使用 exponential backoff 處理 rate limit
- 將 embedding 與 chunk 一起存入 vector DB
```

**5. 向量資料庫選型：**
```
以這個規模來看，Pinecone 或 Qdrant 都合適。

選擇標準：
- Metadata filtering：對權限控管至關重要
- 規模：10M 文件 × 5 chunks = 50M vectors
- Hybrid search：關鍵字查詢會用到

Schema：
- Vector：embedding
- Metadata：doc_id、chunk_id、language、permissions、source
```

<a id="query-pipeline-deep-dive"></a>
#### 查詢管線深入解析

**1. 權限解析：**
```python
def get_user_permissions(user_id: str) -> PermissionSet:
    """
    解析使用者可存取的所有文件。
    回傳集合包含：
    - 直接授權給使用者的文件
    - 展開後的群組成員資格
    - 公開文件存取權

    使用 5 分鐘 TTL 快取，因為權限變更通常不頻繁。
    """
    cache_key = f"permissions:{user_id}"
    if cached := cache.get(cache_key):
        return cached

    perms = permission_service.resolve(user_id)
    cache.set(cache_key, perms, ttl=300)
    return perms
```

**2. 帶過濾條件的檢索：**
```python
def retrieve(query: str, user_id: str, top_k: int = 20) -> List[Chunk]:
    perms = get_user_permissions(user_id)

    # 偵測查詢語言
    lang = detect_language(query)

    # 建立權限過濾條件
    # 使用者可看到：公開文件、自己的文件，或所屬群組文件
    filter = {
        "$or": [
            {"visibility": "public"},
            {"users": {"$in": [user_id]}},
            {"groups": {"$in": perms.groups}}
        ]
    }

    # 可選：提高同語言內容權重
    if lang != "en":
        filter["language"] = lang

    results = vector_db.search(
        query_embedding=embed(query),
        top_k=top_k,
        filter=filter
    )
    return results
```

**3. 重排序：**
```
將 top-20 重新排序，選出 top-5，使用 cross-encoder。
模型：bge-reranker-v2-m3（多語）
延遲預算：約 100ms
```

**4. 生成：**
```python
def generate(query: str, chunks: List[Chunk], user_id: str) -> Response:
    context = format_chunks_with_citations(chunks)

    prompt = f"""You are a knowledge assistant for [Company].
Answer the question using ONLY the provided context.
If the context does not contain the answer, say "I could not find information about that in our knowledge base."
Always cite sources using [1], [2] format.

CONTEXT:
{context}

QUESTION: {query}
"""

    response = llm.generate(
        prompt=prompt,
        model="gpt-4o",
        temperature=0.1
    )

    return format_with_source_links(response, chunks)
```

<a id="scaling-and-reliability"></a>
#### 擴展性與可靠性

**延遲預算（p95 < 3s）：**
```
權限解析：          50ms  （已快取）
Embedding：        100ms
向量搜尋：          100ms
重排序：            150ms
LLM 生成：         1500ms
網路／額外開銷：    100ms
─────────────────────────────
總計：             2000ms（保留 P95 緩衝）
```

**擴展考量：**
```
- Vector DB：依來源或 hash 分片
- Embedding service：水平擴展、無狀態
- LLM calls：多供應商備援
- Cache：以 Redis cluster 快取權限與回應
```

**故障處理：**
```
- Vector DB 當機：回傳快取結果 + 降級警告
- LLM 當機：切換至次要供應商
- Rate limiting：以佇列搭配 backpressure
- Embedding service：批次重試搭配 circuit breaker
```

<a id="evaluation-approach"></a>
#### 評估方式

**離線指標：**
```
- Retrieval：Precision@5、Recall@5、MRR
- Generation：RAGAS（faithfulness、relevance）
- End-to-end：測試集上的答案正確率
```

**線上指標：**
```
- 使用者回饋：Thumbs up/down
- Query reformulation rate：使用者反覆改寫代表失敗
- Citation click-through：來源是否真的有幫助？
```

**監控：**
```
- 依 percentile 劃分的延遲儀表板
- Permission filter hit rate
- 各來源的空結果比例
- 每次查詢成本
```

---

<a id="exercise-2-customer-support-chatbot"></a>
## 練習 2：客戶支援聊天機器人

<a id="problem-statement-1"></a>
### 題目敘述

為一家電商公司設計 AI 驅動的客戶支援系統：

- 每天處理 10,000 段對話
- 可存取商品目錄（100 萬項商品）、訂單歷史、FAQ
- 目標：70% 工單無需人工交接即可解決
- 支援訂單查詢、退貨、商品問題
- 多語支援（3 種語言）
- 與既有 Zendesk 工單系統整合

<a id="solution-highlights"></a>
### 解法亮點

**關鍵架構決策：**

1. **具流程控制的 Agent 架構：**
```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌─────────┐     ┌─────────────┐     ┌─────────────┐   │
│   │ Intake  │────▶│  Classify   │────▶│   Router    │   │
│   └─────────┘     └─────────────┘     └──────┬──────┘   │
│                                              │           │
│         ┌────────────────┬──────────────────┼───────┐   │
│         ▼                ▼                  ▼       ▼   │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐ ┌─────┐ │
│   │Order Flow │   │Product Q&A│   │ Returns   │ │Human│ │
│   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘ └─────┘ │
│         │               │               │               │
│         └───────────────┴───────────────┘               │
│                         │                               │
│                   ┌─────▼─────┐                         │
│                   │  Response │                         │
│                   │ Generator │                         │
│                   └───────────┘                         │
└─────────────────────────────────────────────────────────┘
```

2. **工具設計：**
```python
tools = [
    {
        "name": "lookup_order",
        "description": "Look up order details by order ID or customer email",
        "parameters": {
            "order_id": "optional string",
            "email": "optional string"
        }
    },
    {
        "name": "search_products",
        "description": "Search product catalog",
        "parameters": {
            "query": "string",
            "category": "optional string",
            "price_range": "optional tuple"
        }
    },
    {
        "name": "create_return",
        "description": "Initiate a return for an order",
        "parameters": {
            "order_id": "string",
            "reason": "string",
            "items": "list of item IDs"
        }
    },
    {
        "name": "escalate_to_human",
        "description": "Transfer to human agent",
        "parameters": {
            "reason": "string",
            "priority": "low|medium|high"
        }
    }
]
```

3. **升級給人工的條件：**
```
以下情況升級給人工：
- 客戶明確要求人工服務
- 情緒極度負面（由分類器偵測）
- 問題涉及付款爭議
- Agent 兩次嘗試後信心仍低
- 複雜的多訂單問題
- 退款金額超過門檻
```

4. **整合模式：**
```
Zendesk 整合：
- Webhook 接收新工單
- AI 透過 API 處理
- 解決後 → 關閉工單
- 升級處理 → 附帶摘要指派到佇列
- 所有互動都記錄到工單時間軸
```

---

<a id="exercise-3-code-review-assistant"></a>
## 練習 3：程式碼審查助理

<a id="problem-statement-2"></a>
### 題目敘述

為一個開發平台設計程式碼審查助理：

- 自動審查 pull request
- 提供具體、可執行的回饋
- 遵守 repository 的 style guide 與慣例
- 可以建議程式碼修正
- 與 GitHub/GitLab 整合
- 每天處理 50,000 個 PR

<a id="solution-highlights-1"></a>
### 解法亮點

**關鍵技術選擇：**

1. **內容脈絡組裝：**
```
對每個變更檔案，組裝以下脈絡：
- Diff（變更行）
- 完整檔案內容（幫助理解）
- 相關檔案（imports、tests、types）
- Repository 慣例（.eslintrc、.editorconfig）
- 過往審查評論（從回饋中學習）
```

2. **審查類別：**
```python
review_types = [
    "bug_risk",           # 潛在 bug
    "security",           # 安全問題
    "performance",        # 效能疑慮
    "maintainability",    # 可維護性
    "style",              # 風格規範違反
    "test_coverage"       # 測試不足
]
```

3. **模型選擇：**
```
主要：Claude 3.5 Sonnet（最擅長理解程式碼）
備援：GPT-4o

專用模型：
- Security scanning：CodeQL + LLM review
- Style：Linters + LLM explanation
```

4. **輸出格式：**
```markdown
## 審查摘要

### 嚴重問題（必修）
- **第 45 行**：user query 存在 SQL injection 漏洞
  ```python
  # 不要這樣：
  query = f"SELECT * FROM users WHERE id = {user_id}"
  # 請改成：
  query = "SELECT * FROM users WHERE id = ?"
  cursor.execute(query, (user_id,))
  ```

### 建議（可考慮修正）
- **第 78-82 行**：這段迴圈可改用 list comprehension 簡化
...
```

5. **延遲策略：**
```
目標：在 PR 建立後 2 分鐘內完成可閱讀的審查結果

策略：
- 將 PR 放入佇列處理
- 檔案平行處理
- 結果可用時即串流輸出
- 快取 repository 慣例
```

---

<a id="exercise-4-document-processing-pipeline"></a>
## 練習 4：文件處理管線

<a id="problem-statement-3"></a>
### 題目敘述

為金融服務設計文件處理管線：

- 每天處理 100,000 份文件（發票、合約、表單）
- 以 99% 準確率擷取結構化資料
- 處理 PDF、掃描文件、手寫筆記
- 符合 HIPAA/SOC2
- 低信心擷取結果需人工覆核

<a id="solution-highlights-2"></a>
### 解法亮點

**管線架構：**

```
┌────────┐   ┌───────────┐   ┌────────────┐   ┌────────────┐
│ Ingest │──▶│ Classify  │──▶│  Extract   │──▶│  Validate  │
└────────┘   └───────────┘   └────────────┘   └────────────┘
                                                     │
                                     ┌───────────────┼───────────────┐
                                     ▼               ▼               ▼
                              ┌──────────┐   ┌──────────┐   ┌──────────┐
                              │ Auto-pass│   │  Review  │   │  Reject  │
                              └──────────┘   └──────────┘   └──────────┘
```

**關鍵元件：**

1. **文件分類：**
```
針對文件類型微調分類器：
- Invoice、Contract、Receipt、Form、ID、Other

模型：LayoutLMv3 或微調後的 ViT
自動路由的信心門檻：0.95
```

2. **擷取策略：**
```
依文件類型採用分層擷取：

第 1 層：Document AI（Textract/Azure）
- 適合結構化表單
- 快且便宜
- 會回傳 confidence scores

第 2 層：Vision LLM（GPT-4V/Claude）
- 作為複雜版面的備援
- 更適合非結構化文字
- 成本較高

整合兩層輸出並交叉驗證。
```

3. **驗證規則：**
```python
validation_rules = {
    "invoice": [
        ("total", lambda x: x > 0, "Total must be positive"),
        ("date", lambda x: parse_date(x), "Invalid date format"),
        ("vendor_id", lambda x: regex_match(x, TAX_ID_PATTERN), "Invalid tax ID"),
        ("line_items", lambda x: sum(i.amount for i in x) == total, "Line items must sum to total")
    ],
    "contract": [
        ("parties", lambda x: len(x) >= 2, "Contract must have at least 2 parties"),
        ("effective_date", lambda x: parse_date(x), "Invalid date"),
        ("signature_present", lambda x: x == True, "Signature required")
    ]
}
```

4. **人工審查介面：**
```
審查者可看到：
- 原始文件影像
- 含 confidence score 的擷取欄位
- 已標示的驗證錯誤
- LLM 提供的修正建議
- 一鍵核准或逐欄修正
```

5. **合規措施：**
```
HIPAA/SOC2 要求：
- 所有文件靜態加密（AES-256）
- 傳輸中使用 TLS 1.3
- 所有存取與變更皆保留 audit log
- PHI 偵測與遮罩
- 強制執行保留政策
- 使用 MFA 的存取控制
```

---

<a id="exercise-5-real-time-content-moderation"></a>
## 練習 5：即時內容審核

<a id="problem-statement-4"></a>
### 題目敘述

為一個社群平台設計內容審核系統：

- 每天 100 萬則貼文（文字、圖片、影片）
- 延遲要求：貼文在 500ms 內可見
- 偵測：仇恨言論、暴力、成人內容、垃圾訊息
- 需有誤判申訴流程
- 支援 10 種語言

<a id="solution-highlights-3"></a>
### 解法亮點

**架構模式：多階段級聯（Multi-Stage Cascade）**

```
         ┌───────────────────────────────────────────┐
         │              Fast Filters                 │
         │   (regex, blocklist, hash matching)       │
         └─────────────────┬─────────────────────────┘
                           │ Pass 95%
                           ▼
         ┌───────────────────────────────────────────┐
         │            ML Classifiers                 │
         │   (text: BERT, image: CLIP, video: X3D)   │
         └─────────────────┬─────────────────────────┘
                           │ Uncertain 5%
                           ▼
         ┌───────────────────────────────────────────┐
         │            LLM Analysis                   │
         │   (context-aware, nuanced decisions)      │
         └─────────────────┬─────────────────────────┘
                           │ Still uncertain 0.5%
                           ▼
         ┌───────────────────────────────────────────┐
         │            Human Review                   │
         └───────────────────────────────────────────┘
```

**關鍵設計決策：**

1. **延遲最佳化：**
```
目標：總延遲 500ms

第 1 階段（Fast）：20ms
- Regex patterns
- 已知 hash 比對（PhotoDNA）
- Blocklist lookup

第 2 階段（ML）：80ms
- GPU 批次推論
- 小型專用模型
- 文字／圖片平行處理

第 3 階段（LLM）：400ms（邊界案例採 async）
- 只有 5% 內容會進到這裡
- 用於細緻判斷
```

2. **門檻策略：**
```python
class ModerationDecision:
    BLOCK = "block"          # 高信心違規
    ALLOW = "allow"          # 高信心安全
    LIMIT = "limit"          # 降低分發
    REVIEW = "human_review"  # 送人工審查

thresholds = {
    "hate_speech": {
        "block": 0.95,
        "limit": 0.80,
        "review": 0.60
    },
    "adult_content": {
        "block": 0.98,  # 門檻更高，涉及法律風險
        "limit": 0.90,
        "review": 0.70
    }
}
```

3. **申訴流程：**
```
1. 使用者提出申訴
2. 內容進入人工審查佇列
3. 由不同於原判定者的人員複審（blind review）
4. 決策與理由記錄留存
5. 若被推翻：
   - 恢復內容
   - 原決策作為負樣本加入訓練資料
   - 定期重新訓練模型
```

---

<a id="exercise-6-multi-tenant-ai-platform"></a>
## 練習 6：多租戶 AI 平台

<a id="problem-statement-5"></a>
### 題目敘述

設計一個多租戶 AI 平台（AI-as-a-Service）：

- 服務 500+ 家企業客戶
- 每個客戶都有自己的文件與模型
- 租戶之間必須完全資料隔離
- 支援按租戶追蹤用量與計費
- 不同價格方案具備不同能力
- 必須符合 SOC2

<a id="solution-highlights-4"></a>
### 解法亮點

**租戶隔離架構：**

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   Auth → Tenant Context → Rate Limit → Route             │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Tenant-Aware Service Layer                    │
│                                                                  │
│  All operations scoped to tenant_id from context                │
│  - Retrieval filters by tenant                                  │
│  - Cache keys prefixed by tenant                                │
│  - Audit logs include tenant                                    │
└─────────────────────────────────────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Shared Vector  │ │  Shared LLM     │ │  Shared Object  │
│  DB (filtered)  │ │  (no tenant     │ │  Storage        │
│                 │ │   data in prompt│ │  (tenant paths) │
│  tenant_id in   │ │   history)      │ │                 │
│  all metadata   │ │                 │ │  s3://bucket/   │
└─────────────────┘ └─────────────────┘ │  {tenant_id}/   │
                                        └─────────────────┘
```

**關鍵隔離點：**

```python
class TenantContext:
    tenant_id: str
    user_id: str
    tier: str  # "starter" | "pro" | "enterprise"

    def __enter__(self):
        # 為所有下游呼叫設定 tenant context
        _tenant_context.set(self)

    def __exit__(self, *args):
        _tenant_context.set(None)

# Middleware 確保每個 request 都有 tenant context
@middleware
def enforce_tenant_context(request, call_next):
    tenant_id = extract_tenant_from_token(request.headers["Authorization"])
    with TenantContext(tenant_id=tenant_id, ...):
        verify_tenant_access(tenant_id, request.path)
        response = call_next(request)
        add_tenant_to_audit_log(tenant_id, request, response)
    return response
```

**計費與用量追蹤：**

```python
usage_schema = {
    "tenant_id": "string",
    "timestamp": "datetime",
    "operation": "embed|retrieve|generate",
    "model": "string",
    "tokens_in": "int",
    "tokens_out": "int",
    "latency_ms": "int",
    "cost_cents": "decimal"
}

# 即時用量彙總
async def track_usage(tenant_id: str, operation: Usage):
    # 追加寫入 time-series DB
    await timeseries.write("usage", {
        "tenant_id": tenant_id,
        **operation.dict()
    })

    # 更新 rate limiting 用的即時計數器
    await redis.incr(f"usage:{tenant_id}:{today()}", operation.tokens)
```

---

<a id="exercise-7-semantic-search-at-scale"></a>
## 練習 7：大規模語意搜尋

<a id="problem-statement-6"></a>
### 題目敘述

為電商網站設計語意搜尋系統：

- 5,000 萬項商品
- 每天 1 億筆查詢
- P99 延遲低於 100ms
- 支援篩選條件（價格、類別、品牌、評分）
- 根據使用者歷史做個人化
- 即時庫存更新

<a id="solution-highlights-5"></a>
### 解法亮點

**關鍵挑戰：在 100M queries/day 下維持 100ms**

```
100M queries/day = 平均 1,157 QPS
尖峰：5,000-10,000 QPS

在 100ms 延遲下，需要：
- Edge caching
- 預先計算 embeddings
- 最佳化檢索
- 最小化 LLM 參與
```

**架構：**

```
┌────────────────────────────────────────────────────────────┐
│                         CDN/Edge                            │
│              (Cache popular queries: ~30% hit)              │
└─────────────────────────────┬──────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│                      Query Service                          │
│  1. Embed query (cached embeddings for common queries)      │
│  2. Retrieve candidates (ANN search)                        │
│  3. Apply filters (post-filter or hybrid)                   │
│  4. Personalize ranking                                     │
│  5. Return results                                          │
└─────────────────────────────┬──────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│                    Vector Database Cluster                  │
│  - Sharded by category (reduce search space)                │
│  - HNSW index with ef_search tuned for speed                │
│  - Metadata filtering with roaring bitmaps                  │
└────────────────────────────────────────────────────────────┘
```

**延遲預算：**

```
Edge cache check:    5ms
Embedding lookup:   10ms（已快取）或 30ms（即時計算）
Vector search:      30ms
Filtering:          10ms
Personalization:    10ms
Serialization:      10ms
Network overhead:   25ms
─────────────────────
Total:              100ms 目標（快取命中時）
```

**混合搜尋策略：**

```python
def search(query: str, filters: dict, user_id: str) -> List[Product]:
    # 根據查詢內容決定搜尋策略
    if is_keyword_heavy(query):
        # "nike air max 90 size 10"
        sparse_weight = 0.7
        dense_weight = 0.3
    else:
        # "comfortable running shoes for flat feet"
        sparse_weight = 0.3
        dense_weight = 0.7

    # 平行檢索
    dense_results = vector_db.search(embed(query), top_k=100, filter=filters)
    sparse_results = elastic.search(query, top_k=100, filter=filters)

    # Reciprocal rank fusion
    combined = rrf_merge(
        [dense_results, sparse_results],
        weights=[dense_weight, sparse_weight]
    )

    # 個人化加權
    personalized = apply_user_preferences(combined, user_id)

    return personalized[:20]
```

**即時更新：**

```
商品更新（價格、庫存）流程：
1. 變更事件發布到 Kafka
2. Consumer 更新 vector DB metadata
3. 搜尋結果在數秒內反映變更

重新索引（描述變更）：
1. 需要完整重新 embedding
2. 以 async job 執行
3. 完成後交換索引
```

---

<a id="tips-for-whiteboard-exercises"></a>
## 白板練習技巧

<a id="drawing-tips"></a>
### 繪圖技巧

1. **先畫方塊與標籤**，再用箭頭連接
2. **使用一致記號**：矩形代表服務、圓柱代表資料庫、箭頭代表資料流
3. **在箭頭上標示資料**：說明元件之間流動的是什麼
4. **預留空間**，方便討論時補充

<a id="common-patterns-to-know"></a>
### 常見架構模式

| 模式 | 何時使用 | 畫法 |
|---------|-------------|---------|
| Load balancer + service fleet | 任何需要擴展的服務 | LB → 多個方塊 |
| Queue + workers | 非同步處理 | Queue → worker pool |
| Cache layer | 讀多寫少、延遲敏感 | 服務前一個菱形 |
| CDC/streaming | 即時更新 | Kafka/stream 圖示 |
| Sidecar | 橫切式關注點 | 附在服務旁的小方塊 |

<a id="phrases-that-signal-strong-candidates"></a>
### 能展現強候選人特質的說法

- 「在我開始設計之前，先讓我理解一下規模……」
- 「這裡的權衡是……」
- 「在正式上線時，我們還需要……」
- 「一個值得考慮的 failure mode 是……」
- 「讓我帶你看一下延遲預算……」
- 「在評估方面，我會衡量……」

<a id="time-management"></a>
### 時間管理

- 釐清問題不要超過 5 分鐘
- 先畫完整的高階圖，再深入細節
- 為可靠性與評估預留時間
- 針對重點領域與面試官同步

---

*另請參閱：[題庫](01-question-bank.md) | [回答框架](02-answer-frameworks.md) | [常見陷阱](03-common-pitfalls.md)*
