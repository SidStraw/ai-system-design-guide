<a id="semantic-kernel"></a>
# Semantic Kernel

**Semantic Kernel (SK)** 是 Microsoft 用於企業級 AI orchestration 的引擎。對於深度投入 **Azure/Microsoft 生態系**與 **C#/.NET** 架構的組織而言，它仍然是主要橋梁，雖然其許多後續動能現在已整合進 **Microsoft Agent Framework**（AutoGen + SK 的整合後繼者，2026 年 2 月 RC 1.0，2026 Q2 GA）。

<a id="table-of-contents"></a>
## 目錄

- [企業 DNA](#dna)
- [Plugins 與 Planners](#plugins)
- [Memory 與 Connectors](#memory)
- [多語言支援（C# vs. Python）](#multi-language)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="dna"></a>
<a id="enterprise-dna"></a>
## 企業 DNA

雖然 LangChain 更受新創公司青睞，但 Semantic Kernel 更受**銀行與 Fortune 500 企業**偏好。
- **Dependency Injection**：SK 遵循標準的企業設計模式。
- **Strong Typing**：對 C# 型別的一等支援，使它在大規模、任務關鍵型系統中更加可靠。
- **Security**：與 Azure Active Directory（Microsoft Entra ID）及 Managed Identities 深度整合。

---

<a id="plugins"></a>
<a id="plugins-and-planners"></a>
## Plugins 與 Planners

1. **Kernel Functions**：最基本的邏輯單位（原生程式碼或 LLM prompts）。
2. **Plugins**：一組 functions 的集合（例如「GitHub Plugin」或「SQL Plugin」）。
3. **Planners**：SK 的 planners 已從簡單的 ReAct 演進為**階層式 Planners**，可跨越多天協調長時間執行的商業流程。

---

<a id="memory"></a>
<a id="memory-and-connectors"></a>
## Memory 與 Connectors

Semantic Kernel 使用 **Connectors** 來抽象化底層基礎設施。
- **Universal Connectors**：以單一介面支援 OpenAI、Mistral 與本機 Onyx models。
- **Vector Store Abstraction**：可在不改動核心商業邏輯的情況下，無縫切換 Azure AI Search、Pinecone 與 Qdrant。

---

<a id="multi-language"></a>
<a id="multi-language-support-c-vs-python"></a>
## 多語言支援

SK 是少數將 C# 與 Python 視為對等公民的主流框架之一。
- **典型模式**：先用 Python 開發與原型驗證；再用 C# 部署核心 orchestration，以獲得效能與型別安全。
- **邏輯共享**：可跨兩種語言共用的 prompt templates（.yaml）。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-would-a-staff-engineer-choose-semantic-kernel-over-langchain"></a>
### 問：為什麼 Staff Engineer 會選擇 Semantic Kernel，而不是 LangChain？

**強力回答：**
**架構對齊**。如果一個組織已經建立在 .NET/Azure 技術棧之上，Semantic Kernel 就能自然融入現有的 CI/CD、監控（App Insights）與安全性（Entra ID）流程。LangChain 常讓人感覺像是一塊「外來」技術。此外，SK 的 **Strong Typing** 與 **Dependency Injection** 模式，能避免大型 LangChain 專案常見的「spaghetti code」。對於處理敏感金融資料的企業而言，安全與稽核所需的 **原生 Azure 整合**就是決定性因素。

<a id="q-what-is-the-function-calling-abstraction-in-semantic-kernel"></a>
### 問：Semantic Kernel 中的「Function Calling」抽象是什麼？

**強力回答：**
SK 使用**以 Plugin 為基礎的模型**。每個 function（原生 C# 或 LLM 型）都會註冊到 Kernel 中。當 LLM 判斷自己需要工具時，Kernel 會到 Plugin registry 查找對應 function、驗證參數，然後執行它。SK 現在也支援**Automatic Intent Detection**：Kernel 可以根據目前的 context window，在使用者尚未提出要求前，就主動建議他可能需要哪個 Plugin。

---

<a id="references"></a>
## 參考資料
- Microsoft Learn．《Semantic Kernel Documentation》（2025）
- Azure Architecture Center．《AI Design Patterns with Semantic Kernel》（2025）
- Build 2025．《The Future of Copilots with SK》（2025 Conference Recap）

---

*下一篇：[AutoGen and CrewAI](07-autogen-crewai.md)*
