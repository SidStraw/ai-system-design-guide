<a id="batching-strategies"></a>
# Batching 策略

Batching 是提升 LLM throughput 與降低成本的主要槓桿。服務框架已不再停留在單純的 request-level batching，而是進一步走向 sub-token、iteration-level 的協調排程。

<a id="table-of-contents"></a>
## 目錄

- [Static vs. Dynamic Batching](#static-vs-dynamic)
- [Continuous Batching](#continuous-batching)
- [In-Flight Batching（Prefill-Decode Fusion）](#in-flight-batching)
- [Chunked Prefill 與 RAD-O](#chunked-prefill)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="static-vs-dynamic-batching"></a>
## Static vs. Dynamic Batching

在傳統 ML（Classification）中，我們使用 **Static Batching**，也就是所有請求都必須同樣大小，並一起開始／結束。由於 LLM 回應長度可變，這種方式非常沒效率。

---

<a id="continuous-batching-iteration-level"></a>
## Continuous Batching（Iteration-level）

Continuous batching（由 Orca 與 vLLM 推廣）允許新請求在每個 token 生成步驟結束時加入 batch，也允許已完成的請求離開。

| 面向 | Static Batching | Continuous Batching |
|--------|-----------------|---------------------|
| **Join/Leave** | 只在開始／結束 | 任一 iteration 都可 |
| **GPU Utilization**| 低（等待最長請求） | 高（始終接近飽和） |
| **Throughput** | 1x | **4x - 10x** |
| **Latency** | 對短請求最差 | 較平衡 |

---

<a id="in-flight-batching-prefill-decode-fusion"></a>
## In-Flight Batching（Prefill-Decode Fusion）

過去的 serving engines 只能處理一批「Prefill」（重算力）**或**一批「Decode」（重記憶體）。
**In-Flight Batching**（TensorRT-LLM）允許兩者混合：
- 1 個請求處於 Prefill 階段。
- 15 個請求處於 Decode 階段。
- **Benefit**：Prefill 請求吃掉 GPU 閒置的算力核心，而 Decode 請求則吃掉記憶體頻寬。

---

<a id="chunked-prefill-rad-o"></a>
## Chunked Prefill 與 RAD-O

超大上下文 prompts（1M+ tokens）在 Prefill 階段可能讓整個 batch 卡住數秒，形成「stalls」。

**解法：Chunked Prefill**
與其一次 prefill 128k tokens，engine 會把 prefill 切成更小區塊（例如每塊 4k tokens），再與其他使用者持續進行的 Decode 步驟交錯執行。這樣即使重型請求進來，也能維持穩定的 **TPOT**。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-continuous-batching-superior-to-static-batching-for-llms"></a>
### Q: 為什麼 Continuous Batching 比 Static Batching 更適合 LLM？

**強答：**
Static batching 會讓 batch 中所有請求都得等最長生成完成（也就是「longest tail」問題）。如果一位使用者要 500 tokens，另一位只要 5 tokens，對 5-token 使用者而言，GPU 接下來 495 個 cycle 都是空等。Continuous batching 則允許 5-token 請求在最後一個 token 產生後立刻退出 GPU，釋放 VRAM 與計算槽位給佇列中的新請求。這能最大化整個硬體叢集的「Tokens per Second」。

<a id="q-what-is-a-stall-in-llm-serving-and-how-does-chunked-prefill-mitigate-it"></a>
### Q: 什麼是 LLM serving 裡的「stall」？Chunked Prefill 如何緩解？

**強答：**
當一個超大新請求進來，而它的 Prefill 階段（非常吃算力）需要 2-3 秒才能完成時，就會發生「stall」。在這段期間，GPU 忙著處理 prefill，導致無法替既有使用者的 Decode 階段產生 tokens，使他們的 TPOT 暴增。Chunked Prefill 會把那段 3 秒 prefill 拆成許多 200ms 的小區塊：做一塊、替其他人解碼一輪、再回來做下一塊。這樣就能讓所有使用者都維持一致且平滑的體驗。

---

<a id="references"></a>
## 參考資料
- Yu et al. "Orca: A Distributed Serving System for [Transformer] Models" (2022)
- NVIDIA. "TensorRT-LLM: In-Flight Batching" (2023)
- vLLM Project. "Iteration-Level Scheduling" (2023)

---

*下一篇：[PagedAttention](05-paged-attention.md)*
