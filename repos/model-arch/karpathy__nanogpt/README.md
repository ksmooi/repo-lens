---
repo: karpathy/nanoGPT
file: README
studied_at: 2026-05-25
commit_sha: 3adf61e
type: model-arch
language: Python
framework: PyTorch
task: language-modeling (autoregressive, decoder-only)
paper: GPT-2 (Language Models are Unsupervised Multitask Learners)
stars: ~58.7k
status: maintenance (deprecated in favor of nanochat)
---

# nanoGPT · 概覽

## 解決什麼問題

nanoGPT 是 [minGPT](https://github.com/karpathy/minGPT) 的繼任者，提供一個**極簡但 production-viable** 的 GPT-2 訓練／微調實作。核心主張是：以最少程式碼行數（~600 行 Python）完成從 random init 到 reproduce GPT-2 (124M) val loss ~2.85 的完整流程。

不在於創新架構，而在於**用最乾淨的方式展示一個 medium-sized GPT 的完整實作**。

## 為什麼值得研究

nanoGPT 是程式碼閱讀的「最小可行範本」：

| 面向 | 為什麼值得看 |
|------|------------|
| **單一檔案模型** | `model.py` 約 330 行，完整實作 GPT-2 — 不到 HuggingFace transformers 同功能程式碼的 1/10 |
| **訓練迴圈透明度** | `train.py` 約 336 行，無框架抽象，可以直接看到每個 gradient accumulation step 在做什麼 |
| **Config 極簡主義** | `configurator.py` 約 47 行，用 `exec()` 做 config override — 故意不走 Hydra / YAML / argparse |
| **GPT-2 權重載入** | `from_pretrained()` 方法展示如何從 HuggingFace checkpoint 做 weight mapping，包括 Conv1D → Linear 的 transpose |
| **MFU 估算** | `estimate_mfu()` 實作了 PaLM paper 的 Model FLOPs Utilization 估算公式 |

## 任務與資料

| 面向 | 值 |
|------|----|
| 任務類型 | self-supervised (autoregressive next-token prediction) |
| 輸入 | tokenized text (uint16 sequence) |
| 輸出 | logits over vocab |
| 主要資料集 | OpenWebText (open reproduction of OpenAI's WebText) |
| 評估指標 | cross-entropy loss (perplexity) |

## 模型一句話

GPT-2 架構 decoder-only transformer，參數規模 124M / 350M / 774M / 1558M（對應四種預訓練大小），使用 learned positional embedding + LayerNorm pre-norm + GELU activation + standard multi-head causal self-attention。

## 健康度信號

| 指標 | 值 |
|------|----|
| ⭐ Stars | ~58,700 |
| 📅 最後 commit | 2025-11-12 |
| 👥 主要維護者 | Andrej Karpathy（個人專案） |
| 🔬 狀態 | 已 deprecated，官方建議改用 [nanochat](https://github.com/karpathy/nanochat) |

## 跟競品的比較

| 面向 | nanoGPT | HuggingFace transformers | minGPT |
|------|---------|------------------------|--------|
| 模型實作行數 | ~330 行 | ~3000+ 行 (GPT2Model + GPT2LMHeadModel) | ~500 行 |
| 訓練迴圈 | 內建 single file | 需自己另寫或套 Trainer | 教學向，不完整 |
| Config 系統 | `exec()` over globals | YAML / dataclass | argparse |
| DDP 支援 | 完整（手寫，~20 行） | 需套 accelerate / Trainer | 無 |
| GPT-2 權重載入 | 內建 from_pretrained | 內建 from_pretrained | 部分支援 |
| 論文 reproduce | ✅ GPT-2 124M val loss ~2.85 | ✅ 但抽象層太多 | ❌ 未驗證 |
| 可讀性取向 | 教育 + 實用 | production + 通用性 | 純教育 |

核心 trade-off：nanoGPT 為了可讀性犧牲了泛用性（只能跑 decoder-only LM），transformers 為了泛用性犧牲了可讀性。這不是缺點，而是設計取捨。

## 我會在後續筆記中回答的問題

- 為什麼 GPT-2 的訓練可以只用 600 行 Python 就完成 reproduce？
- `exec()` config system 是 genius 還是 terrible idea？什麼情境適用？
- nanoGPT 的 gradient accumulation + DDP 的 handled sync pattern 是怎麼運作的？
- weight tying、residual projection scaling、causal mask 的實作細節是什麼？
