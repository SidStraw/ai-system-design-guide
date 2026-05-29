<a id="dspy-programming-language-models"></a>
# DSPy：Programming Language Models

**DSPy** 已成為高可靠性 AI 系統的業界參考標準。它代表了從「Prompt Engineering」（反覆試錯）轉向 **Prompt Compilation**（自動化最佳化）的典範轉移，而且基準測試一再顯示，相較於人工微調的 prompts，品質可提升 10–40%。

<a id="table-of-contents"></a>
## 目錄

- [程式設計典範](#paradigm)
- [Signatures：描述任務](#signatures)
- [Optimizers 與 MIPROv2](#optimizers)
- [Assertions 與 Constraints](#assertions)
- [管理模型漂移](#model-drift)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="paradigm"></a>
<a id="the-programming-paradigm"></a>
## 程式設計典範

DSPy 將一個 LLM 應用視為**神經網路**。
- **Module**：可重複使用的邏輯區塊（例如 `ChainOfThought`）。
- **Signature**：對 module 行為的宣告式規格（Input -> Output）。
- **Optimizer**：根據某個 metric 為 module 找出最佳「Weights」（Prompts）的過程。

---

<a id="signatures"></a>
<a id="signatures-describing-the-task"></a>
## Signatures：描述任務

你不需要撰寫 100 行 prompt，而是撰寫一個 **Signature**：
```python
class ResearchAssistant(dspy.Signature):
    """Answer the question by synthesizing the provided web context."""
    context = dspy.InputField(desc="Scraped web content")
    question = dspy.InputField()
    answer = dspy.OutputField(desc="A technical summary with citations")
```
**關鍵細節**：Signatures 是**模型無關**的。你可以將它們編譯給 Claude Opus 4.7、Claude Sonnet 4.6、GPT-5.5、Gemini 3.1 Pro 或 Llama 4 8B，而不必改動任何一行程式碼。

---

<a id="optimizers"></a>
<a id="optimizers-and-miprov2"></a>
## Optimizers 與 MIPROv2

**MIPROv2（Multi-stage Instruction PRoposal Optimizer）** 是 DSPy 的旗艦 optimizer。
1. **Instruction Proposal**：一個「Assistant Model」會提出 10–20 種不同的 system prompt 撰寫方式。
2. **Bayesian Optimization**：DSPy 會在一小組訓練資料上執行這些 prompts，並用某個 metric 為其評分。
3. **Selection**：它會選出能讓你的 metric（例如 Factuality score）最大化的 prompt。

---

<a id="assertions"></a>
<a id="assertions-and-constraints"></a>
## Assertions 與 Constraints

DSPy 允許使用**硬性與軟性 Assertions**。
- `dspy.Suggest(...)`：如果模型未通過某項檢查（例如「答案必須少於 50 個字」），DSPy 會根據失敗原因**自動重新提示**模型，讓它自行修正。
- `dspy.Assert(...)`：如果違反硬性限制（例如「不得包含 PII」），執行流程就會停止並進入恢復狀態。

---

<a id="model-drift"></a>
<a id="managing-model-drift"></a>
## 管理模型漂移

當 OpenAI 或 Anthropic 發布新的 weights 更新時，手工打造的 prompts 往往會失效。
- **2025 年的解法**：在 DSPy 中，你只需要**重新編譯**。optimizer 會為更新後的模型架構找出新的「最佳」tokens，無需人工勞動也能維持一致性。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-dspy-considered-anti-prompt-engineering"></a>
### 問：為什麼 DSPy 被視為「反 Prompt Engineering」？

**強力回答：**
因為它用**最佳化迴圈**取代了**人工反覆試錯迴圈**。在 prompt engineering 中，人類是 optimizer；在 DSPy 中，人類是**Teacher**。你定義的是*目標*（Signature）與*評估方式*（Metric），並提供少量*Examples*。接著，框架會使用數學最佳化（例如 Bayesian search）去找出在統計上表現最好的 tokens。這使系統比一堆寫死字串的做法更具**可移植性**與**可擴充性**。

<a id="q-what-is-the-biggest-drawback-of-using-dspy-in-a-production-environment"></a>
### 問：在正式生產環境中使用 DSPy，最大的缺點是什麼？

**強力回答：**
**編譯延遲與成本**。要編譯一條複雜的 DSPy pipeline，你可能需要執行 100–500 次 LLM 呼叫來測試不同的 prompt 變體。這是一筆可觀的前期成本。不過，對 Staff 級工程師而言，這是一種**權衡**：你在開發／編譯階段投入更多時間，換來**有保證的可靠性**以及更低的**執行期失敗率**。另一個挑戰是學習曲線；它要求你像 ML 研究人員一樣思考，而不是傳統開發者。

---

<a id="references"></a>
## 參考資料
- Khattab et al.《DSPy: Compiling Declarative Language Model Calls》（2024/2025）
- Stanford NLP．《The MIPROv2 Technical Report》（2025）
- Databricks．《Productionizing Programmed Prompts》（2025）

---

*下一篇：[Semantic Kernel: Enterprise AI](06-semantic-kernel.md)*
