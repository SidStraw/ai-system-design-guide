<a id="computer-use-agents"></a>
# 電腦操作代理

Computer-use agents 讓 LLM 能看見螢幕、對其進行推理，並透過滑鼠點擊與鍵盤輸入採取動作——就像人類操作電腦一樣。模型不是呼叫結構化 API，而是直接處理原始像素。本章會說明它們的運作方式、何時能勝過傳統自動化，以及如何圍繞它們設計可投入生產的系統。

<a id="table-of-contents"></a>
## 目錄

- [什麼是 Computer-Use Agents？](#what-are-computer-use-agents)
- [Screenshot-Reason-Act 迴圈](#the-screenshot-reason-act-loop)
- [Claude Computer Use：工具與 API](#claude-computer-use-tools-and-api)
- [架構：沙箱化環境](#architecture-sandboxed-environments)
- [瀏覽器自動化 vs 桌面自動化](#browser-vs-desktop-automation)
- [與傳統自動化的比較](#comparison-with-traditional-automation)
- [何時 Computer-Use 會勝過 API 呼叫](#when-computer-use-beats-api-calls)
- [錯誤處理與復原](#error-handling-and-recovery)
- [效能：延遲、成本、吞吐量](#performance-latency-cost-throughput)
- [真實世界應用](#real-world-applications)
- [安全考量](#security-considerations)
- [程式碼範例](#code-examples)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="what-are-computer-use-agents"></a>
## 什麼是 Computer-Use Agents？

Computer-use agent 是一種透過解讀螢幕截圖並發出低階輸入指令（滑鼠移動、點擊、鍵盤輸入）來控制圖形介面的 LLM。它會在人機互動迴圈中取代人類的位置。

```
Traditional Tool Use:           Computer Use:

User Request                    User Request
     |                               |
     v                               v
 LLM reasons                    LLM reasons
     |                               |
     v                               v
 Structured API call             Screenshot captured
 {"tool": "search",                  |
  "query": "..."}                    v
     |                          LLM sees pixels, finds button
     v                               |
 API returns JSON                    v
     |                          Mouse click at (x=340, y=220)
     v                               |
 LLM formats answer                  v
                                New screenshot captured
                                     |
                                     v
                                LLM verifies result, continues...
```

關鍵差異在於：傳統工具使用仰賴預先定義、schema 已知的 API。Computer use 則可作用於任何具備視覺介面的應用程式——不需要 API。

<a id="the-landscape-2026"></a>
### 目前版圖（2026）

現在已有多家供應商提供 computer-use 能力：

| 供應商 | 代理 | 方法 | 主要優勢 |
|----------|-------|----------|--------------|
| Anthropic | Claude Computer Use | Vision + 座標推理 | 桌面 + 瀏覽器、成熟 API |
| OpenAI | ChatGPT Agent Mode | 以 Operator 為基礎的瀏覽器代理 | 深度網頁導覽 |
| Google | Project Mariner | Gemini vision-language | Chrome 整合 |
| Microsoft | UFO/UFO2 | Windows UI Automation + vision | 原生 Windows 支援 |
| Amazon | Nova Act | 為瀏覽器打造的專用模型 | 電商工作流程 |

---

<a id="the-screenshot-reason-act-loop"></a>
## Screenshot-Reason-Act 迴圈

每個 computer-use agent 都遵循相同的核心迴圈，通常稱為「agent loop」或「action loop」：

```
+------------------+
|  Capture Screen  |<-----------+
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  Send to LLM     |            |
|  (screenshot +   |            |
|   task context)  |            |
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  LLM Reasons     |            |
|  about next      |            |
|  action           |           |
+--------+---------+            |
         |                      |
    +----+----+                 |
    |         |                 |
    v         v                 |
 [Action]  [Done]               |
    |                           |
    v                           |
+------------------+            |
| Execute Action   |            |
| (click, type,    |            |
|  scroll, key)    |            |
+--------+---------+            |
         |                      |
         +----------------------+
```

每次迭代都包含：
1. **Capture**：擷取目前顯示狀態的螢幕截圖。
2. **Send**：把截圖（base64 圖像）加上對話歷史傳給 LLM。
3. **Reason**：模型分析螢幕上的內容，判斷朝目標前進的下一步。
4. **Act**：模型輸出工具呼叫（例如 `click at (450, 320)`），由執行環境實際執行。
5. **Repeat**：再次擷取新截圖，直到模型發出完成訊號前持續重複。

模型透過持續累積截圖與動作的對話歷史，在多次迭代間維持上下文，形成一種視覺上的「記憶」，記住已經發生過的事。

---

<a id="claude-computer-use-tools-and-api"></a>
## Claude Computer Use：工具與 API

Claude 提供三個內建的 computer use 工具。這些工具由 Anthropic 定義——你不需要自行撰寫實作；Claude 知道如何產生對它們的呼叫，而你的 runtime 會在環境中執行這些呼叫。

<a id="the-three-tools"></a>
### 三個工具

**1. `computer` -- 完整 GUI 控制**

控制虛擬顯示器上的滑鼠與鍵盤。能力包括：
- `screenshot` -- 擷取目前螢幕狀態
- `left_click`, `right_click`, `double_click`, `triple_click` -- 在座標位置進行滑鼠點擊
- `left_click_drag` -- 從一個點拖曳到另一個點
- `type` -- 輸入一段文字字串
- `key` -- 按下鍵盤按鍵（例如 `ctrl+c`、`Return`、`Escape`）
- `scroll` -- 在指定座標向上／下／左／右捲動
- `move` -- 將游標移動到座標位置
- `hold_key` -- 執行其他動作時按住修飾鍵
- `wait` -- 暫停指定時間

**2. `bash` -- Shell 指令執行**

在持續存在的工作階段中執行 shell 指令：
- 指令會共享狀態（環境變數、工作目錄）
- 支援多行 script
- 輸出會被擷取並以文字形式回傳

**3. `text_editor` -- 檔案操作**

以結構化方式編輯檔案，支援以下指令：
- `view` -- 讀取檔案內容（可選擇行號範圍）
- `create` -- 建立帶有內容的新檔案
- `str_replace` -- 取代檔案中的特定字串（必須唯一匹配）
- `insert` -- 在指定行號插入文字

<a id="api-request-structure"></a>
### API 請求結構

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    tools=[
        {
            "type": "computer_20250124",
            "name": "computer",
            "display_width_px": 1280,
            "display_height_px": 800,
            "display_number": 1
        },
        {
            "type": "bash_20250124",
            "name": "bash"
        },
        {
            "type": "text_editor_20250124",
            "name": "str_replace_based_edit_tool"
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "Open Firefox, navigate to github.com, and find repos trending today."
        }
    ],
    betas=["computer-use-2025-01-24"]
)
```

回應會包含 `tool_use` 區塊，你的 runtime 必須執行它們，並以 `tool_result` 訊息回傳結果。

---

<a id="architecture-sandboxed-environments"></a>
## 架構：沙箱化環境

Computer-use agents 必須在隔離環境中執行。模型可完全控制滑鼠與鍵盤——你絕對不會希望它直接操作你的正式工作站。

<a id="standard-architecture-docker--vnc"></a>
### 標準架構：Docker + VNC

```
+-----------------------------------------------------+
|  Docker Container                                   |
|                                                     |
|  Xvfb (Virtual X11) + Mutter (WM) + Tint2 (Panel)  |
|         |                                           |
|         v                                           |
|  +------------------+     +-------------------+     |
|  | Virtual Desktop  |---->| Screenshot Capture|     |
|  | 1280x800         |     | (scrot/maim)      |     |
|  | Firefox, apps    |     +--------+----------+     |
|  +------------------+              |                |
|                                    v                |
|                           +--------+----------+     |
|                           | Agent Runtime     |     |
|                           | - Calls Claude API|     |
|                           | - Executes actions|     |
|                           | - Manages loop    |     |
|                           +-------------------+     |
+-----------------------------------------------------+
```

<a id="cloud-hosted-alternatives"></a>
### 雲端託管替代方案

像 E2B (e2b.dev) 這類服務提供預先設定好的沙箱環境：
- 內建瀏覽器與工具的短暫性 VM
- 用於截圖擷取與輸入注入的 API
- 工作階段結束後自動清理
- 無需承擔 Docker 管理成本

<a id="key-environment-components"></a>
### 關鍵環境元件

| 元件 | 用途 | 範例 |
|-----------|---------|---------|
| Xvfb | 虛擬 X11 顯示伺服器 | 在沒有實體顯示器的情況下建立 framebuffer |
| Mutter/Xfwm | 視窗管理器 | 處理視窗定位與大小調整 |
| Tint2 | 工作列面板 | 顯示正在執行的應用程式 |
| xdotool | 輸入注入 | 執行滑鼠／鍵盤指令 |
| scrot/maim | 截圖擷取 | 將顯示內容快照為 PNG |

---

<a id="browser-vs-desktop-automation"></a>
## 瀏覽器自動化 vs 桌面自動化

| 面向 | 僅瀏覽器 | 完整桌面 |
|-----------|-------------|--------------|
| 範圍 | 僅限 Web 應用 | 任何 GUI 應用程式 |
| 設定複雜度 | 較低（headless browser） | 較高（完整桌面環境） |
| 效能 | 較快（較小的截圖） | 較慢（全螢幕擷取） |
| 可靠性 | 較高（版面較可預測） | 較低（OS 差異） |
| 使用情境 | 網頁爬取、表單填寫 | 舊式軟體、跨應用工作流程 |

瀏覽器自動化是控制 web browser（導覽、填表、點按按鈕、處理 SPA）。桌面自動化則控制整個 OS 環境（啟動應用程式、使用原生對話框、與 thick-client 軟體互動、串接多個應用程式的操作）。

---

<a id="comparison-with-traditional-automation"></a>
## 與傳統自動化的比較

Selenium、Playwright 與 Puppeteer 透過直接存取 DOM 來自動化瀏覽器。Computer-use agents 則是處理像素。兩者在生產環境中都有其定位。

| 特性 | Selenium/Playwright | Computer Use Agent |
|---------|--------------------|--------------------|
| 速度 | 快（直接操作 DOM） | 慢（截圖 + LLM） |
| 可靠性 | 脆弱（selector 一改就壞） | 韌性較高（視覺辨識） |
| 維護成本 | 需要持續更新 selector | 極少（可適應 UI 變化） |
| 反機器人偵測 | 經常被擋 | 較難偵測 |
| 每次動作成本 | ~$0.001 | ~$0.01-0.05 |
| 非 Web 支援 | 否 | 是（任何 GUI） |

**混合式方法** 在正式環境中效果最好：Playwright 處理高流量、明確定義的流程（登入、導覽），而 computer-use agents 處理動態且不可預測的步驟（視覺驗證、新穎版面、反機器人網站）。

---

<a id="when-computer-use-beats-api-calls"></a>
## 何時 Computer-Use 會勝過 API 呼叫

**適合使用 computer use 的情況：** 沒有 API 可用（舊系統）、反機器人防護會擋下 Selenium、需要視覺判斷（圖表驗證、PDF 版面）、UI 變化速度快到 selector 難以維護，或工作流程橫跨多個桌面應用程式。

**應繼續使用 API 的情況：** 有結構化 API 可用（永遠優先）、延遲很重要（次秒級）、量很大（每小時數千個動作），或必須要有決定性行為（相同輸入得到相同輸出）。

---

<a id="error-handling-and-recovery"></a>
## 錯誤處理與復原

Computer-use agents 的失敗方式與 API 型工具不同。主要失敗模式如下：

<a id="1-misclicks-wrong-coordinates"></a>
### 1. 點錯（錯誤座標）

模型會根據截圖計算座標，但可能偏差幾個像素：
- **Mitigation**：每次點擊後都使用 `screenshot` 驗證是否真的發生預期的狀態變化。
- **Recovery**：如果點到了錯誤元素，模型可以推理新的狀態並修正路徑。

<a id="2-stale-screenshots"></a>
### 2. 過期截圖

畫面可能在擷取與執行动作之間已經改變（動畫、彈窗、載入中）：
- **Mitigation**：截圖前先短暫等待。頁面載入時使用 `wait` 動作。
- **Recovery**：繼續前重新擷取並重新判斷。

<a id="3-infinite-loops"></a>
### 3. 無限迴圈

模型重複相同動作，卻沒有任何進展：
- **Mitigation**：設定最大迭代次數（例如每個任務最多 50 個動作）。
- **Recovery**：若連續 N 次出現相同動作，就強制改用不同方法，或升級交由人工處理。

<a id="4-unexpected-dialogs"></a>
### 4. 非預期對話框

Cookie 橫幅、彈窗、權限對話框可能會突然出現：
- **Mitigation**：在 system prompt 中加入處理常見對話框的指示。
- **Recovery**：模型的視覺推理通常能自然處理這些情況——它會看到對話框並將其關閉。

<a id="5-resolution-and-scaling-mismatches"></a>
### 5. 解析度與縮放不一致

模型是在特定解析度下訓練的。不一致會導致座標錯誤：
- **Mitigation**：使用建議解析度（1280x800），並將顯示縮放設為 100%。
- **Recovery**：調整 `display_width_px` 與 `display_height_px`，使其與實際顯示相符。

<a id="error-handling-pattern"></a>
### 錯誤處理模式

agent loop 應追蹤動作歷史並偵測重複。如果相同行動連續出現 3 次以上，就注入一則訊息要求模型改用其他方法。務必設定硬性的最大迭代次數（例如 50），並在每次動作後擷取驗證截圖，以偵測狀態變化。完整 agent loop 請見下方的程式碼範例章節。

---

<a id="performance-latency-cost-throughput"></a>
## 效能：延遲、成本、吞吐量

<a id="latency-breakdown"></a>
### 延遲拆解

agent loop 的每次迭代都包含：

```
Screenshot capture:     ~100ms
Image encoding (base64): ~50ms
API call (with image):   ~2-5s  (model inference)
Action execution:        ~100ms
                        --------
Total per action:        ~2.5-5.5s
```

一個典型的 10 步任務需要 25-55 秒。相比之下，Playwright 完成相同 10 個步驟通常不到 2 秒。

<a id="cost-per-action"></a>
### 每次動作成本

每個動作都會傳送一張截圖（約 800KB 的 base64）加上對話歷史：

| 模型 | 每次動作成本（約略） | 備註 |
|-------|-------------------------|-------|
| Claude Sonnet 4 | $0.01-0.03 | 建議用於大多數任務 |
| Claude Opus 4 | $0.05-0.15 | 適合複雜視覺推理 |

使用 Sonnet 時，20 步工作流程大約花費 $0.20-0.60；使用 Opus 則約為 $1.00-3.00。

<a id="throughput-optimization"></a>
### 吞吐量最佳化

- **Parallel sessions**：為並行任務執行多個 Docker container。
- **Selective screenshots**：只在不確定的動作後截圖；輸入文字後可略過。
- **Resolution reduction**：使用 1024x768 取代 1920x1080，以降低 token 成本。
- **Early termination**：教模型在確認目標達成後盡早發出完成訊號。

---

<a id="real-world-applications"></a>
## 真實世界應用

| 應用 | 運作方式 | 為何適合 Computer Use |
|------------|--------------|------------------|
| 舊系統整合 | 代理操作 mainframe/thick-client UI，將資料擷取為結構化格式 | 舊式軟體沒有 API |
| 表單填寫／資料輸入 | 讀取來源文件，逐欄填入 web form，處理多頁 wizard | 政府入口網站、具有複雜條件邏輯的保險理賠 |
| QA 與視覺測試 | 像使用者一樣操作 app，驗證視覺呈現，並用自然語言回報問題 | 不只做 pixel-diff——還能理解版面與 UX |
| 競品情報 | 瀏覽產品頁面，從 JS-rendered widget 擷取價格資料 | 能在封鎖傳統爬蟲的網站上運作 |

---

<a id="security-considerations"></a>
## 安全考量

| 風險 | 會發生什麼事 | 緩解方式 |
|------|-------------|------------|
| **可見的秘密資訊** | 模型會在截圖中看到密碼、工作階段、通知 | 使用短暫性 container，使用後清除憑證 |
| **不受限制的動作** | 代理可執行 shell 指令、任意導覽、下載檔案 | 防火牆規則、唯讀檔案系統、工作階段時間限制、對破壞性操作採用 HITL |
| **資料外洩** | 傳送給 LLM 供應商的截圖可能包含敏感資料 | 受監管產業可採 on-premise 部署，遮罩敏感 UI 欄位 |
| **經由 UI 的 prompt injection** | 惡意網站顯示文字以操控代理 | 在 system prompt 中警告不要遵從與任務衝突的螢幕指示 |

最重要的原則：**永遠不要在你的正式工作站上執行 computer-use agents，也不要讓它們接觸真實憑證，除非是在完全沙箱化的 container 中**。

---

<a id="code-examples"></a>
## 程式碼範例

<a id="minimal-agent-loop"></a>
### 最小化 Agent Loop

```python
import anthropic, base64, subprocess

client = anthropic.Anthropic()

def capture_screenshot():
    subprocess.run(["scrot", "/tmp/screen.png", "-o"], check=True)
    with open("/tmp/screen.png", "rb") as f:
        return base64.standard_b64encode(f.read()).decode()

def execute_action(action):
    name = action["action"]
    if name == "left_click":
        x, y = action["coordinate"]
        subprocess.run(["xdotool", "mousemove", str(x), str(y), "click", "1"])
    elif name == "type":
        subprocess.run(["xdotool", "type", "--", action["text"]])
    elif name == "key":
        subprocess.run(["xdotool", "key", action["text"]])

def run_agent(task: str, max_steps: int = 30):
    messages = [{"role": "user", "content": task}]
    tools = [
        {"type": "computer_20250124", "name": "computer",
         "display_width_px": 1280, "display_height_px": 800},
        {"type": "bash_20250124", "name": "bash"},
        {"type": "text_editor_20250124", "name": "str_replace_based_edit_tool"},
    ]
    for step in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-4-20250514", max_tokens=4096,
            tools=tools, messages=messages, betas=["computer-use-2025-01-24"],
        )
        if response.stop_reason == "end_turn":
            return [b.text for b in response.content if b.type == "text"]

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            if block.name == "computer":
                execute_action(block.input)
                tool_results.append({
                    "type": "tool_result", "tool_use_id": block.id,
                    "content": [{"type": "image", "source": {
                        "type": "base64", "media_type": "image/png",
                        "data": capture_screenshot()}}],
                })
            elif block.name == "bash":
                r = subprocess.run(block.input["command"],
                    shell=True, capture_output=True, text=True)
                tool_results.append({
                    "type": "tool_result", "tool_use_id": block.id,
                    "content": r.stdout + r.stderr,
                })
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
    return ["Max steps reached"]
```

<a id="dockerfile-for-sandboxed-environment"></a>
### 沙箱化環境的 Dockerfile

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    xvfb mutter tint2 xdotool scrot firefox-esr python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*
RUN pip3 install anthropic
ENV DISPLAY=:1
COPY agent.py /agent.py
CMD Xvfb :1 -screen 0 1280x800x24 & sleep 1 && mutter & tint2 & \
    sleep 1 && python3 /agent.py
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-a-client-has-500-insurance-claim-pdfs-per-day-that-must-be-entered-into-a-legacy-web-portal-with-no-api-design-a-system-using-computer-use-agents"></a>
### Q：某客戶每天有 500 份保險理賠 PDF，必須輸入到一個沒有 API 的舊式 Web 入口網站。請設計一個使用 computer-use agents 的系統。

**強答：**
我會建立一個包含三個階段的 pipeline。第一階段是文件處理，使用 LLM 從 PDF 中擷取結構化資料（理賠編號、申請人姓名、金額、日期）。第二階段是 computer-use agent，讓每份理賠都由在隔離 Docker container 與虛擬顯示器中執行的 Claude Computer Use agent 處理。代理會導覽該 web portal、使用擷取出的資料填寫表單欄位，並在送出後擷取確認截圖。第三階段是驗證，使用獨立的 LLM 呼叫比對確認截圖與預期資料，以捕捉任何輸入錯誤。

若要擴展，我會平行執行 10-20 個 container，每個 container 依序處理理賠案件。若代理平均每件約需 2 分鐘，20 個 container 在 8 小時工作天內可處理 600 件。我也會加入 dead-letter queue，讓重試 3 次後仍失敗的案件交由人工複查。

若每件理賠成本是 $0.50（約 20 個動作、每次 $0.025），那麼 500 件每天約為 $250——很可能比被取代的人工資料輸入團隊更便宜。

<a id="q-compare-computer-use-agents-with-selenium-for-web-automation-when-would-you-choose-each"></a>
### Q：請比較 computer-use agents 與 Selenium 在 Web 自動化上的差異。你會在什麼情況下選擇各自方案？

**強答：**
Selenium 直接與 DOM 互動——速度快、具決定性、成本低。但只要 selector 改變就容易失效，也會被反機器人系統阻擋，而且無法處理需要視覺判斷的任務。

Computer-use agents 每個動作大約慢 100 倍、貴 10 倍，但因為它們處理的是像素而不是 selector，所以能適應 UI 變化。它們也更擅長避開反機器人偵測，因為產生的是更像人類的互動模式。此外，它們能推理視覺版面——例如驗證圖表是否正確渲染，或從 Selenium 無法檢查的 canvas 元素讀取內容。

對於高流量、穩定且目標網站由我掌控的工作流程，我會選擇 Selenium。對於一次性任務、經常變動的第三方網站、跨應用桌面工作流程，以及維護 selector 的人工成本已高於 LLM 推理成本的情境，我會選擇 computer-use agents。

最佳的生產系統通常兩者並用：Playwright 處理可預測步驟（驗證、導覽），computer-use agent 則處理動態步驟（解讀結果、進行判斷）。

---

<a id="references"></a>
## 參考資料

- Anthropic. "Computer Use Tool" API Documentation (2025)
- Anthropic. "Bash Tool" and "Text Editor Tool" API Documentation (2025)
- E2B. "Sandboxed Cloud Environments for AI Agents" (2025)
- OSWorld Benchmark: Desktop Agent Evaluation Suite (2025)
- WebArena Benchmark: Web Agent Evaluation Suite (2024)

---

*下一節： [Building Tool-Use Agents](05-building-tool-agents.md)*
