# LLM Provider Abstraction

> 把「呼叫 LLM」這件事抽象成一個內部介面,讓 application code 不直接綁定 OpenAI / Anthropic / 其他 provider 的 SDK。

## 它解決什麼問題

LLM 應用發展到一定規模後,幾乎都會遇到這些需求:

- **多 provider 支援** — OpenAI 出新模型、Claude 在某類任務更便宜、本地 vLLM 用於敏感資料
- **A/B 測試** — 同樣的 prompt 在不同模型表現差多少
- **Fallback** — 主 provider 限流或故障時自動切到備援
- **Cost / latency 路由** — 簡單任務用便宜模型,複雜任務用強模型
- **可測試性** — 單元測試時用 mock,不要真的打 API
- **可觀測性** — 統一的 token 計算、cost 追蹤、tracing 接入點

如果 application code 直接寫 `openai.chat.completions.create(...)`,上述每一項都要散落各處重寫。

## 核心想法

定義一個內部介面,通常包含三個層次:

```
┌──────────────────────────┐
│  Application Code        │   ← 只認識 internal interface
└──────────────────────────┘
            ↓
┌──────────────────────────┐
│  Provider Interface      │   ← 定義抽象方法:complete(), stream(), embed()
└──────────────────────────┘
            ↓
┌──────────────────────────┐
│  Provider Adapters       │   ← 各 provider 的具體實作
└──────────────────────────┘
            ↓
┌──────────────────────────┐
│  Vendor SDKs             │   ← openai, anthropic, etc.
└──────────────────────────┘
```

關鍵設計決策:

1. **介面該抽象到什麼程度** — 太薄(直接轉發 OpenAI 格式)會被 OpenAI-shaped 綁架;太厚(自製訊息格式)會增加學習成本
2. **怎麼處理 provider-specific 功能** — 例如 Anthropic 的 cache_control、OpenAI 的 logprobs。常見做法是 extras dict 或 conditional kwargs
3. **錯誤怎麼正規化** — 每個 provider 的 error 類型不同,要不要全部映射到統一的 exception hierarchy
4. **是否抽象 streaming** — streaming 介面差異更大,有人選擇只抽象 non-streaming,streaming 各 provider 各自處理

## 觀察到的變體

| 變體 | 特徵 | 取捨 |
|---|---|---|
| **OpenAI-shaped 通用介面** | 用 OpenAI 的 message schema 當共同語言,其他 provider 在 adapter 內轉譯 | 上手快、生態完整;但 OpenAI 變了大家都要跟著變 |
| **自製訊息格式** | 完全自己定義 Message、ToolCall 等抽象 | 中立性高、可控;但學習成本與維護成本都高 |
| **薄包裝 + 直通** | 介面非常薄,大量功能用 extras dict 傳遞 | 彈性大;但失去了類型保護 |
| **第三方統一層** | 直接用 LiteLLM、OpenRouter 之類的中間層 | 開發快;多了一層依賴與不確定性 |

## 跟替代方案的對照

| 方案 | 優點 | 缺點 |
|---|---|---|
| **直接用 vendor SDK** | 簡單、功能完整、跟 vendor 文件對得起來 | 換 provider 要大改、難測試 |
| **自製抽象層** | 完全可控、可塑造團隊概念模型 | 維護成本、學習成本 |
| **LiteLLM 等通用層** | 開箱即用、社群維護 | 多一層依賴、特殊功能要繞 |
| **OpenAI-compatible endpoint** | 統一在 OpenAI schema | 仍需要選 client 套件、有些 provider 兼容性差 |

選擇通常跟組織規模有關:小團隊 / prototype 用 LiteLLM 或直接綁 vendor;中型團隊自製薄抽象;大型團隊或核心產品才值得做厚抽象。

## 何時用 / 何時不用

**用抽象層當**:

- 確定會用到 2 個以上 provider(包含本地模型)
- 有合規 / 資料邊界考量(某些資料只能走特定 provider)
- 預期會做 A/B 或 routing
- LLM 呼叫散佈在多處(超過 5-10 個 call site)
- 需要統一的觀測 / cost 控制

**不要用當**:

- 還在驗證產品想法,只用一個 provider
- LLM call 只在一兩個地方,直接寫 vendor SDK 更清楚
- 團隊很小、沒有要做 multi-provider 計畫
- 你抽象的速度趕不上 vendor 加新功能的速度——這時 LiteLLM 之類的社群方案會更划算

一個常見錯誤:**過早抽象**。在還沒有第二個 provider 出現之前就建抽象層,通常會抽象錯方向,等真的要加第二個 provider 時還要重寫。

## 在不同 repo 的實作

### Hermes Agent

支援極多 provider(Nous Portal、OpenRouter、NovitaAI、NVIDIA NIM、Xiaomi MiMo、z.ai/GLM、Kimi、MiniMax、HuggingFace、OpenAI、自製 endpoint)。`hermes model` 指令切換不需改 code。

- Provider 抽象目錄: [`providers/`](https://github.com/NousResearch/hermes-agent/tree/main/providers) `[UNVERIFIED 內部結構]`
- 設計亮點: 對使用者暴露的切換介面極簡(一個 CLI 指令),內部用 adapter pattern 處理差異
- 觀察: provider 數量這麼多代表抽象層做得不錯,否則維護成本會爆炸

### LangChain

最早期、影響最廣的 LLM provider 抽象。`BaseChatModel` 介面被廣泛模仿。

- 核心介面: `langchain_core.language_models.BaseChatModel` `[UNVERIFIED]`
- 設計亮點: 用 Runnable 介面把 LLM 跟其他元件(retriever、tool)統一在同一個 protocol 下
- 批評: 抽象層次太厚,有時讓人搞不清楚到底發生了什麼;`[UNVERIFIED]` 觀察可能因版本而異

### LiteLLM

走「第三方統一層」路線。把所有 provider 都翻譯成 OpenAI 格式。

- 核心翻譯邏輯: `litellm/llms/` 各 provider 子目錄 `[UNVERIFIED]`
- 設計亮點: 用既有的 OpenAI schema 當 lingua franca,使用者只需要學一套 API
- Trade-off: 跟著 OpenAI 走,某些 provider 的特色功能無法完整表達

### OpenAI Python SDK + OpenAI-compatible endpoints

不是抽象層本身,但形成了事實標準。許多 provider(包括許多開源推論引擎)都提供 OpenAI-compatible 端點,讓你直接用 OpenAI SDK 連到非 OpenAI 服務。

- 觀察價值: 這展示了「不抽象也能多 provider」的另一條路——只要大家都實作同一個 API 規範

> `[TODO]` 補上對 instructor、Marvin、Mirascope 等 typed-LLM library 的觀察,它們對「結構化輸出」這部分的抽象方式很有啟發

## 我的建議

如果你正在決定要不要做 provider abstraction:

1. **先問:你會用幾個 provider?** — 答案是 1,別抽象;是 2-3,用 LiteLLM 或類似方案;是「核心產品的長期基礎建設」,才考慮自製
2. **抽象層要薄、不要厚** — 抽象 90% 的常用功能,剩下 10% 用 extras / kwargs 漏出去。試圖完美抽象 100% 通常會失敗
3. **訊息格式選 OpenAI shape** — 不是因為它設計最好,而是因為生態最大,大部分 provider 已經會主動兼容
4. **儘早決定 streaming 怎麼辦** — streaming 介面是最容易被 vendor 牽著走的部分。決定要不要抽象,然後堅持住
5. **錯誤正規化值得投資** — 不一定要全部映射,但至少把 retry-able 跟 not-retry-able 分清楚,因為這直接影響上層的 error handling
6. **加 telemetry hook 在抽象層** — 這是抽象層相對於直接呼叫 SDK 的最大隱性收益。把 token usage、latency、cost 都從這層發出去,上層完全不用管

最後一個觀察:**provider abstraction 是少數「做了就回不去」的基礎建設**。一旦 application code 大量依賴它,後續要改抽象就很痛。所以前期多花時間設計介面是值得的——但前提是你已經確定需要它,不是出於「以後可能會用到」的恐懼性設計。
