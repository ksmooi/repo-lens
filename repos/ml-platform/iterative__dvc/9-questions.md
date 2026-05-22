---
repo: iterative/dvc
file: 9-questions
studied_at: 2026-05-22
commit_sha: 06ff81c
---

# DVC · 未解問題

## 還沒搞懂的設計決策

- [ ] **DVC 為什麼選擇 `voluptuous` 而不是 `pydantic` 或 `jsonschema` 做 schema validation？**
  - 我目前的推測：DVC 開始開發時（2017）pydantic 還不成熟；voluptuous 提供更 Pythonic 的 declarative schema。但現在回頭看，pydantic 的社群跟 ecosystem 明顯更強。[UNVERIFIED]
  - 相關程式碼:[`dvc/config_schema.py:1`](https://github.com/iterative/dvc/blob/06ff81c/dvc/config_schema.py#L1)

- [ ] **為什麼 experiments queue 預設用 `LocalCeleryQueue` 而不是更輕量的 threading/multiprocessing？**
  - 我目前的推測：Celery 的 task orchestration 模型（routing、retry、result backend）可以複用；但 DVC 的 queue 完全沒有分散式 worker 的支援，引入 Celery 的重量似乎不划算。[UNVERIFIED]
  - 相關程式碼:[`dvc/repo/experiments/queue/celery.py`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/experiments/queue/celery.py)

- [ ] **DVC 的 `@locked` 裝飾器使用 re-entrant lock counter，但如果多個 `@locked` 方法互相呼叫，會不會有死結（deadlock）？**
  - 我目前的推測：`lock_repo()` 的 `_lock_depth` 機制理論上可以處理，但實務上 `@locked` 方法的呼叫圖（call graph）需要仔細審計才能確認沒有循環。[UNVERIFIED]
  - 相關程式碼:[`dvc/repo/__init__.py:37-61`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/__init__.py#L37-L61)

- [ ] **為什麼 `Repo` class 同時使用 `from X import Y` 方法注入和 `staticmethod(_du)` 兩種 pattern？**
  - 我目前的推測：`staticmethod()` 的模式是為了那些不依賴 `self` 的靜態方法（如 `du`、`ls`），而 import assignment 保留 instance method 的特性是因為它們需要 `self`。但混用兩種模式增加了讀者的認知負擔。[UNVERIFIED]
  - 相關程式碼:[`dvc/repo/__init__.py:96-100`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/__init__.py#L96-L100)

- [ ] **DVC 的 hash 為什麼支援兩種演算法（SHA-256 和 legacy `md5-dos2unix`）？**
  - 我目前的推測：早期 DVC 使用 MD5，後來為了安全與一致性遷移到 SHA-256。`md5-dos2unix` 是舊版相容用的特殊變體（標準化換行符號後計算 MD5）。但保留了 legacy 支援意味著永遠有兩套 hash 計算邏輯要維護。[UNVERIFIED]
  - 相關程式碼:[`dvc/data_cloud.py:46`](https://github.com/iterative/dvc/blob/06ff81c/dvc/data_cloud.py#L46)

## 想問維護者的問題

- [ ] DVC 跟 Git 的搭配非常緊密，但 Git 本身也有各種實作（libgit2、dulwich、git CLI）。你們在 `scmrepo` wrapper 的設計上遇到過哪些 git 實作之間的相容性問題？
- [ ] 為什麼決定把 `dvc-data`、`dvc-objects`、`dvc-task` 拆成獨立套件？是為了版本管理，還是為了讓社群能獨立使用這些底層庫？
- [ ] Pipeline 的執行目前是單執行緒的（一個 stage 跑完才跑下一個）。有考慮過平行執行獨立 stage？（像是 `snakemake` 或 `nextflow` 的策略）

## 下次再看時的待辦

- [ ] 試著用 `dvc-studio-client` 接上 DVC Studio，理解遠端 experiment tracking 的資料流
- [ ] 深入閱讀 `dvc-data` 套件的 `DataIndex` 實作，理解 content-addressed 的 index 結構
- [ ] 比較 DVC 的 `dvc.yaml` pipeline 定義跟 `ZenML` / `Kedro` 的 pipeline 抽象差異
- [ ] 實際用 `dvc-gdrive` 測試 Google Drive backend 的穩定性
- [ ] 閱讀 `dvc/render/` 的實作，理解 DVC plots 的 rendering pipeline

## 跨專案對照備忘

- **Pattern: 方法注入（Method Injection）** — DVC 的 `Repo` class 模式跟 SQLAlchemy 的 `Session` class 注入 query 方法有相似之處。兩者都透過 import-time assignment 來分解大 class。→ 候選 pattern，已在 2 個 repo 觀察到（DVC + SQLAlchemy）。
- **Pattern: Git stash 作為 queue** — 用 Git ref 存佇列狀態的做法。跟 `paper_lens` 的 branch-based 排程不同。值得追蹤是否有其他 repo 也用類似模式。→ 尚未滿足收錄門檻。
