---
repo: HKUDS/RAG-Anything
file: 99-questions
studied_at: 2026-05-21
commit_sha: e8c0081
---

# RAG-Anything · 未解問題

## 還沒搞懂的設計決策

- [ ] **跨模態 processor 之間沒有並行處理**
  目前每個跨模態區塊是序列處理的——即使檔案中有 10 張圖片，也是逐一呼叫 VLM。為什麼不平行處理？是為了控制 LLM rate limit，還是為了保持 context 提取的頁序相關性？
  - 相關程式碼: [`processor.py`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/processor.py)（跨模態處理區段）
  - 我目前的推測: [UNVERIFIED] 可能是為了避免 VLM rate limit 被大量並行呼叫打爆。檔案內的圖片通常也彼此相關（同一篇論文的圖），序列處理讓 shared context 的內容更豐富。

- [ ] **純文字和跨模態內容的索引路徑不一致**
  `insert_text_content` 和 `insert_text_content_with_multimodal_content` 是兩條不同的插入路徑，但最終都寫入同一個 LightRAG instance。跨模態的描述是當作一般 text chunk 插入，還是透過 LightRAG 的 graph entity/relation API？如果是當作 text chunk，那結構化資訊（type、entity_name、summary）在檢索時怎麼被利用？
  - 相關程式碼: [`utils.py`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/utils.py)（insert_text_content_with_multimodal_content）

- [ ] **PromptRegistry 的跨 coroutine 安全性**
  `PromptRegistry.swap()` 會替換整個 _data dict，Python 的 dict reference assignment 在 CPython 中是 atomic 的。但如果多個 coroutine 同時讀取 PROMPTS（透過 `__getitem__`）和寫入（透過 `swap`），讀取到的可能是新 dict 或舊 dict——這不是 data race，但可能導致短暫的行為不一致。這是設計意圖內的取捨，還是沒有考慮到的 edge case？
  - 相關程式碼: [`prompt.py:23-28`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/prompt.py#L23-L28)

## 想問維護者的問題

- 為什麼選擇 `@dataclass` + mixin 而不是 composition？有特別的歷史原因，還是純粹作者偏好？
- MinerU 2.0 不再包含 LibreOffice 模組，Office 文件需要預先轉 PDF。這個限制對使用者體驗的影響大嗎？是否有考慮整合 Docling 作為 Office 文件的替代解析方案？
- 跨模態分析使用 LLM/VLM 的成本（token 消耗）有做過統計嗎？對於一個包含 10+ 圖片的 PDF，跨模態處理的 token 成本大約是多少？

## 下次再看時的待辦

- [ ] 深入看 `LightRAG` 的對接程式碼（`_ensure_lightrag_initialized`），理解 `key_string_value_json_storage_cls` 的實作——這是一個有趣的 factory pattern
- [ ] 跑看看 RAG-Anything 的 pipeline（找一個有圖片的 PDF 實際測試），觀察 context 提取的品質對 VLM 輸出的影響
- [ ] 對照 LightRAG 的 standalone 用法和 RAG-Anything 的封裝，看看 RAG-Anything 對 LightRAG 做了哪些配置簡化

## 跨專案對照備忘

- **Context-aware multimodal indexing**（Pattern 3）跟 LlamaIndex 的 `ImageNode` 設計很像——兩者都意識到「獨立的圖片描述」在 RAG 場景中效果有限。LlamaIndex 的做法是把圖片作為 ImageNode，與前後的 TextNode 建立關聯。RAG-Anything 的做法是在解析階段就把 context 灌進 VLM prompt 中。兩個方法的最終目標一樣（讓圖片描述包含上下文），但路徑不同：一個在資料層（Node graph），一個在 prompt 層。→ **候選 pattern**（觀察到至少 2 個 repo）
