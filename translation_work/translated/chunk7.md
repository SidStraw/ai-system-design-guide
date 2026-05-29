<a name="appendix-a"></a>
## 附錄 A：PM 與 QA 術語詞彙表

本指南全文使用的技術術語簡明詞彙表，可與非技術利害關係人分享。

### 評估與指標術語

| 術語 | 定義 |
|------|-----------|
| **Eval（評估）** | 一種系統性測試，用於檢查 AI 系統是否針對特定標準正常運作 |
| **LLM-as-a-Judge** | 使用語言模型自動評估另一個 AI 系統的輸出 |
| **Ground Truth（基準真相）** | 由人工驗證的標籤，代表「正確」答案；用於衡量評估器的準確度 |
| **True Positive Rate (TPR)** | 評估器正確識別的實際正例（如良好回應）百分比。亦稱為*召回率（recall）*或*敏感度（sensitivity）*。公式：TP / (TP + FN) |
| **True Negative Rate (TNR)** | 評估器正確抓出的實際負例（如不良回應）百分比。亦稱為*特異度（specificity）*。公式：TN / (TN + FP) |
| **False Positive (FP)** | 評估器判定「通過」但真實答案為「失敗」的情況——即漏失的缺陷 |
| **False Negative (FN)** | 評估器判定「失敗」但真實答案為「通過」的情況——即誤報警報 |
| **Precision（精確率）** | 在所有評估器標記為正例的項目中，實際為正例的比例。公式：TP / (TP + FP) |
| **F1 Score** | 精確率與召回率的調和平均數——一個同時平衡兩者的數值。公式：2 * (Precision * Recall) / (Precision + Recall) |
| **Confusion Matrix（混淆矩陣）** | 顯示 TP、FP、FN、TN 計數的 2x2 表格——所有分類指標的基礎 |
| **Confidence Interval (CI)** | 考慮抽樣不確定性後，真實指標很可能落在其中的數值範圍（例如 72%–81%） |
| **Bias Correction（偏差校正）** | 調整評估器的原始分數，以修正對通過/失敗的系統性高估或低估 |
| **Cohen's Kappa** | 衡量兩個評分者（或評分者與基準真相）一致性的統計量，已扣除機率一致性。數值：<0.2 差、0.4–0.6 中等、0.6–0.8 顯著、>0.8 幾乎完美 |

### 資料與工作流程術語

| 術語 | 定義 |
|------|-----------|
| **Train/Dev/Test Split（訓練/開發/測試分割）** | 將標記資料分為三組：Train（用於建立評估器提示）、Dev（用於迭代改進）、Test（用於最終無偏測量） |
| **Stratified Split（分層分割）** | 分割資料時確保每個子集的通過/失敗標籤比例與原始資料相同 |
| **Few-Shot Examples（少樣本範例）** | 包含在提示中的範例輸入/輸出對，用以向模型示範良好評估的形式 |
| **Open Coding（開放編碼）** | 閱讀追蹤記錄並自由記錄問題所在——尚無分類 |
| **Axial Coding（軸心編碼）** | 將開放編碼的筆記整理成類別（錯誤類型）並統計頻率 |
| **Dimensional Sampling（維度抽樣）** | 系統性地建立覆蓋所有重要維度（主題、邊界案例、使用者類型）的測試輸入 |
| **Failure Mode（失敗模式）** | AI 系統可能發生失敗的具體、命名方式（例如「違反飲食限制」、「幻覺引用」） |
| **Error Taxonomy（錯誤分類法）** | 針對應用程式的所有失敗模式整理成的清單，依頻率與嚴重程度排序 |

### 可觀測性與平台術語

| 術語 | 定義 |
|------|-----------|
| **Trace（追蹤）** | 一次完整的 AI 互動記錄——從使用者輸入，經過所有處理步驟，到最終輸出 |
| **Span（跨度）** | 追蹤中的單一工作單元（例如一次 LLM 呼叫、一次資料庫查詢、一次工具調用） |
| **Instrumentation（植入監測）** | 在應用程式中加入程式碼，使追蹤與跨度能自動被捕獲 |
| **Dataset（資料集）** | 用於執行實驗的已儲存範例集合（輸入 + 預期輸出） |
| **Experiment（實驗）** | 針對資料集執行 AI 系統（或評估器）並記錄所有結果 |
| **Annotation（標注）** | 附加在追蹤或跨度上的標籤或分數——可由人工產生或來自自動化評估 |
| **Prompt Version（提示版本）** | 提示模板的已儲存快照，可用於追蹤變更並比較效能 |

### RAG 專用術語

| 術語 | 定義 |
|------|-----------|
| **RAG (Retrieval-Augmented Generation)** | 一種在生成回應前先檢索相關文件的 AI 架構 |
| **BM25** | 一種傳統的關鍵字搜尋演算法，用作檢索品質的基準線 |
| **Recall@K** | 在所有相關文件中，有多少比例出現在前 K 個檢索結果中 |
| **MRR (Mean Reciprocal Rank)** | 第一個相關文件排名倒數的平均值——數值越高表示相關文件越早出現 |
| **Chunking（切塊）** | 將大型文件切分為較小片段以供檢索 |
| **Context Window（上下文視窗）** | LLM 在單次呼叫中能處理的最大文字量 |
| **Hallucination（幻覺）** | LLM 生成不受檢索上下文支持的資訊 |

### 統計術語

| 術語 | 定義 |
|------|-----------|
| **p_obs（觀測率）** | 評估器的原始通過率，校正前的數值 |
| **θ̂ (Theta-hat)** | 考量評估器誤差後的校正真實成功率 |
| **judgy** | 一個 Python 函式庫，依據 TPR 和 TNR 計算校正後的成功率與信賴區間 |
| **Sampling（抽樣）** | 評估追蹤記錄的隨機子集而非全部——用於控制成本 |
| **Statistical Significance（統計顯著性）** | 觀測到的差異是否可能為真實存在，或可能只是隨機波動所致 |

---

<a name="appendix-b"></a>
## 附錄 B：快速參考

### 何時使用哪種評估類型

| 情境 | 類型 | 範例 |
|-----------|------|---------|
| 格式檢查 | 程式碼型 | SMS 中無 Markdown |
| 必填欄位 | 程式碼型 | 行程確認包含日期/時間 |
| 工具選擇 | 程式碼型 | 呼叫了正確的函數 |
| 主觀品質 | LLM 評估器 | 回應是否有幫助 |
| 政策合規 | LLM 評估器 | 是否符合轉接要求 |
| 飲食遵守 | LLM 評估器 | 食譜是否符合飲食限制 |
| 事實準確性 | LLM 評估器 | 答案是否與來源相符 |
| 回應長度 | 程式碼型 | 少於 500 個字元 |

### 指標速查表

```
Confusion Matrix:
                 Actual Positive  |  Actual Negative
                 -----------------|-----------------
Predicted Pos    |      TP        |       FP        |
Predicted Neg    |      FN        |       TN        |

TPR (Recall) = TP / (TP + FN)      "Catches real positives"
TNR (Specificity) = TN / (TN + FP) "Avoids false alarms"
Precision = TP / (TP + FP)
F1 Score = 2 * (Precision * Recall) / (Precision + Recall)

Target for evals:
- TPR > 80% (catches real issues)
- TNR > 80% (doesn't false alarm)
```

### 資料分割比例

```
Train: ~15%  (few-shot examples for judge prompt)
Dev:   ~40%  (iterate and improve judge prompt)
Test:  ~45%  (final, unbiased evaluation - use ONCE)
```

### 時間估算

| 活動 | 時間 | 頻率 |
|----------|------|-----------|
| 初始設定（LangWatch） | 30 分鐘 | 一次 |
| 初始設定（Langfuse） | 1 小時 | 一次 |
| 錯誤分析（100 筆追蹤） | 1 小時 | 每月 |
| 建立程式碼型評估 | 1 小時 | 視需要 |
| 建立 LLM 評估器（完整流程） | 4-6 小時 | 視需要 |
| 在開發集上驗證評估 | 1 小時 | 每次迭代 |
| 每週維護 | 30 分鐘 | 每週 |

### 平台快速入門

**LangWatch（最快速）：**
```python
import langwatch
langwatch.init()
# Done! Auto-tracing enabled
```

**Langfuse（需要更多設定）：**
```python
from langfuse.openai import OpenAI
client = OpenAI()
# Set environment variables first
```

---

<a name="appendix-c"></a>
## 附錄 C：正式生產環境的完整評估器提示

以下是一個達到 TPR=95.7% 與 TNR=100% 的正式生產品質評估器提示：

```
You are an expert nutritionist and dietary specialist evaluating whether
recipe responses properly adhere to specified dietary restrictions.

DIETARY RESTRICTION DEFINITIONS:
- Vegan: No animal products (meat, dairy, eggs, honey, etc.)
- Vegetarian: No meat or fish, but dairy and eggs are allowed
- Gluten-free: No wheat, barley, rye, or other gluten-containing grains
- Dairy-free: No milk, cheese, butter, yogurt, or other dairy products
- Keto: Very low carb (typically <20g net carbs), high fat, moderate protein
- Paleo: No grains, legumes, dairy, refined sugar, or processed foods
- Pescatarian: No meat except fish and seafood
- Kosher: Follows Jewish dietary laws (no pork, shellfish, mixing meat/dairy)
- Halal: Follows Islamic dietary laws (no pork, alcohol, proper slaughter)
- Nut-free: No tree nuts or peanuts
- Low-carb: Significantly reduced carbohydrates (typically <50g per day)
- Sugar-free: No added sugars or high-sugar ingredients
- Raw vegan: Vegan foods not heated above 118 degrees F (48 degrees C)
- Whole30: No grains, dairy, legumes, sugar, alcohol, or processed foods
- Diabetic-friendly: Low glycemic index, controlled carbohydrates
- Low-sodium: Reduced sodium content for heart health

EVALUATION CRITERIA:
- PASS: The recipe clearly adheres to the dietary preferences with
  appropriate ingredients and preparation methods
- FAIL: The recipe contains ingredients or methods that violate the
  dietary preferences
- Consider both explicit ingredients and cooking methods

Example 1:
Query and Response: [Gluten-free pizza dough using gluten-free flour blend,
baking powder, olive oil, honey, apple cider vinegar...]
Explanation: The recipe uses gluten-free flour blend. All other ingredients
do not contain gluten. The preparation method does not introduce any
gluten-containing elements.
Label: PASS

Example 2:
Query and Response: [Raw vegan quinoa salad with cooked quinoa,
fresh vegetables, olive oil, lemon juice...]
Explanation: The recipe FAILS because it includes cooked quinoa.
Raw vegan diets do not allow foods heated above 118 degrees F (48 degrees C),
and cooking quinoa involves boiling, which exceeds this limit.
Label: FAIL

Now evaluate the following recipe response:

Query: {query}
Dietary Restriction: {dietary_restriction}
Recipe Response: {response}

RETURN YOUR EVALUATION IN JSON FORMAT:
"label": "PASS" or "FAIL",
"explanation": "Detailed explanation citing specific ingredients or methods"
```

---

<a name="appendix-d"></a>
## 附錄 D：流水線狀態評估器提示

各流水線狀態的完整評估器提示，每個提示遵循相同結構：

### 標準評估器結構

```
1. Role definition ("You are an expert evaluator for the X state")
2. What the state should do (3-4 bullet points)
3. Evaluation criteria (3-4 numbered criteria)
4. What counts as a failure (4-5 specific failure types)
5. What does NOT count as a failure (2-3 acceptable variations)
6. Input/Output template variables
7. Output format (JSON with label and explanation)
```

### 可用評估器

| 狀態 | 關鍵標準 | 常見失敗 |
|-------|-------------|----------------|
| ParseRequest | 準確性、完整性、格式 | 誤解、遺漏限制條件 |
| PlanToolCalls | 工具選擇、順序、理由 | 缺少工具、選錯工具 |
| GenRecipeArgs | 查詢相關性、篩選器準確性 | 缺少飲食篩選器、份量錯誤 |
| GetRecipes | 相關性、飲食合規性 | 不相關食譜、違反飲食限制 |
| GenWebArgs | 相關性、上下文對齊 | 偏離主題的查詢、過於籠統 |
| GetWebInfo | 相關性、品質 | 不相關結果、偏離主題的內容 |
| ComposeResponse | 食譜準確性、步驟清晰度、限制合規性 | 矛盾、資訊缺漏、違規 |

各狀態的完整評估器提示遵循上述結構，並針對各流水線階段的具體職責與失敗模式加以調整。

---

<a name="appendix-e"></a>
## 附錄 E：評估器提示工程技巧

一系列能持續提升 LLM 評估器準確度的技巧，在建立或除錯評估器時可作為檢查清單使用。

### 1. 先寫解釋，再給判定

永遠要求評估器在給出最終標籤*之前*先說明其推理過程。這是最具影響力的單一技巧。

```
❌ Bad:  "Label: PASS or FAIL. Explanation: ..."
✅ Good: "Explanation: [your reasoning]. Label: PASS or FAIL"
```

**為何有效：** 當模型先寫標籤時，解釋會變成事後的合理化說詞。當推理過程在前，模型才會真正深思熟慮，標籤也才能合乎邏輯地得出。

### 2. 對標準毫不妥協地具體

模糊的標準會導致不一致的判斷。精確定義何者算通過、何者算失敗。

```
❌ Vague:  "Does the response follow dietary restrictions?"
✅ Specific: "PASS: Every ingredient in the recipe is compatible with the stated
   dietary restriction. FAIL: At least one ingredient violates the restriction,
   OR the cooking method introduces a violation (e.g., frying in butter for
   dairy-free)."
```

### 3. 列出「什麼情況不算失敗」

評估器傾向過度嚴苛。明確列出可接受的變體，以校準寬鬆度。

```
What does NOT count as a failure:
- Suggesting optional toppings that can be omitted
- Using brand names instead of generic ingredient names
- Minor formatting issues in the recipe
- Providing substitution suggestions alongside the main recipe
```

### 4. 使用領域特定的少樣本範例

通用範例的效果遠不如來自實際資料的範例。務必從訓練集中挑選少樣本範例。

**範例選取策略：**
- 1 個明確通過的案例（容易判斷的案例）
- 1 個明確失敗的案例（容易判斷的案例）
- 1 個邊界案例（評估器最容易出錯的類型）

**在每個範例中包含推理過程**，而非只有標籤。評估器學習的是推理模式，而非只是答案。

### 5. 溫度設定

| 使用情境 | 溫度 | 理由 |
|----------|-------------|-----------|
| 二元分類（通過/失敗） | 0.0 | 確定性、可重現 |
| Likert 量表評分（1-5） | 0.0–0.3 | 低變異、一致性高 |
| 產生多樣化評論 | 0.5–0.7 | 適度創意以涵蓋不同角度 |
| 腦力激盪失敗模式 | 0.7–1.0 | 高創意以利探索 |

進行評估器評估時，請一律使用溫度 0.0。您希望相同的輸入每次產生相同的輸出。

### 6. 結構化輸出格式

明確告訴評估器如何格式化其回應。JSON 因解析可靠性而為首選。

```
Return your evaluation as JSON:
{
  "explanation": "Step-by-step reasoning about the response...",
  "label": "PASS or FAIL",
  "confidence": "HIGH, MEDIUM, or LOW",
  "flagged_items": ["list of specific problematic items, if any"]
}
```

**提示：** `confidence` 欄位在錯誤分析期間有助於識別邊界案例，但並非可靠的校正機率。

### 7. 防範常見的評估器偏差

| 偏差 | 說明 | 緩解方式 |
|------|-------------|------------|
| **寬鬆偏差** | 評估器過於頻繁地預設「通過」 | 新增明確的失敗範例；強調「有疑慮時判為失敗」 |
| **冗長偏差** | 評估器偏好更長、更詳細的回應 | 新增短回應通過而長回應失敗的範例 |
| **位置偏差** | 評估器偏好清單中第一個或最後一個選項 | 若比較多個輸出，隨機打亂順序 |
| **奉承偏差** | 評估器認同措辭充滿信心的文字 | 新增充滿信心但內容錯誤的文字範例 |
| **錨定偏差** | 評估器受到第一條證據影響 | 指示評估器在得出結論前考慮所有證據 |

### 8. 迭代改進工作流程

```
1. Write initial prompt with 2-3 few-shot examples
2. Run on Dev set → calculate TPR and TNR
3. Find the worst errors (cases where judge was wrong)
4. For each error:
   a. Understand WHY the judge was wrong
   b. Add a clarification, edge case, or new example to the prompt
5. Re-run on Dev set → check if metrics improved
6. Repeat steps 3-5 until TPR > 80% and TNR > 80%
7. Run ONCE on Test set for final, unbiased metrics
```

**常見迭代模式：**
- TPR 過低 → 評估器遺漏了真實失敗。新增更多失敗範例，使失敗標準更明確。
- TNR 過低 → 評估器有太多誤報。新增「什麼情況不算失敗」段落，為邊界案例新增通過範例。
- 兩者均低 → 標準模糊不清。以更清晰的定義從頭重寫。

### 9. 評估器的模型選擇

| 模型層級 | 使用時機 | 典型準確度 |
|------------|------------|-----------------|
| GPT-4o / Claude Sonnet 4.6 | 高風險評估、複雜推理 | 85–95% |
| GPT-4o-mini / Claude Haiku | 對成本敏感、高量評估 | 75–90% |
| 開源模型（Llama, Mistral） | 自架、隱私敏感 | 70–85% |

**提示：** 先使用能力最強的模型以建立效能上限，再測試較便宜的模型是否能在您的特定使用情境下達到相同效果。通常是可以的——尤其是搭配良好的少樣本範例時。

### 10. 提示版本控制

永遠為您的評估器提示進行版本控制。追蹤：
- 提示文字
- 使用的少樣本範例
- 模型與溫度
- 該版本的開發集指標（TPR、TNR）
- 變更日期與原因

LangWatch 和 Langfuse 均有內建的提示版本控制功能，請善加利用。

**使用 LangWatch：**
```python
import langwatch

langwatch.prompts.create(
    name="dietary-judge-v3",
    description="Added edge cases for keto",
    template=judge_prompt_text,
    model="gpt-4o",
    temperature=0
)
```

**使用 Langfuse：**
```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_prompt(
    name="dietary-judge",
    prompt=judge_prompt_text,
    labels=["staging"],  # promote to "production" after validation
)
```

---
