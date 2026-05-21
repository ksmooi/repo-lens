# multimodal/ — 多模態模型與系統

**上層目錄**: [repos/](../README.md)

視覺語言模型、語音模型、影像生成、多模態 pipeline。
學習重點在於「不同模態的資訊如何對齊、融合、生成」的架構設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **模態融合** | 視覺 / 語音 token 怎麼對齊語言 token?projection layer 設計?cross-attention vs concat? |
| **Encoder 設計** | 視覺 encoder(ViT / CLIP)?語音 encoder(Whisper encoder)?各模態的 tokenizer? |
| **訓練策略** | 哪個 stage 凍結哪個 module?多模態對齊訓練的 loss 設計?資料 curriculum? |
| **Inference Pipeline** | processor / tokenizer / model / decoder 怎麼串?batch 處理多模態輸入? |
| **生成側** | 影像生成的 decoder 設計?latent diffusion 怎麼接 LLM?條件控制機制? |
| **資料 Pipeline** | 多模態 dataset 格式?圖文對 / 影片字幕 的 dataloader 設計? |
| **評估** | VQA / captioning / OCR 的 eval 怎麼整合?多模態 benchmark? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 模態 | 特色 |
|------|------|------|
| LLaVA | 視覺語言 | MLP projection、instruction tuning、簡潔架構 |
| Whisper | 語音辨識 | encoder-decoder、多語言、OpenAI 出品 |
| Stable Diffusion | 文字生成影像 | latent diffusion、CLIP conditioning、廣泛生態 |
| CogVLM | 視覺語言 | visual expert、高解析度、深度融合 |
| Qwen-VL | 視覺語言 | 多語言、document understanding、Alibaba |
| Florence-2 | 視覺基礎模型 | unified prompt-based、多任務、Microsoft |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 視覺 + 語言 / 語音多模態系統 | ✅ `multimodal` | — |
| 純語言模型架構 | — | ✅ [`model-arch`](../model-arch/) |
| 多模態訓練框架(以訓練工程為主) | — | ✅ [`llm-training`](../llm-training/) |
| 多模態 serving 引擎 | — | ✅ [`llm-serving`](../llm-serving/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
