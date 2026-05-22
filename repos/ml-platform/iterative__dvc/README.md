---
repo: iterative/dvc
file: README
studied_at: 2026-05-22
commit_sha: 06ff81c
type: ml-platform
language: Python
framework: N/A (CLI tool + Python SDK)
stars: 15.6k
status: active
---

# DVC · 概覽

## 解決什麼問題

**資料版本控制（Data Version Control）** — 讓 ML 專案可以像用 Git 管理程式碼一樣管理資料集、模型檔案、實驗參數。

DVC 的核心命題很簡單：Git 不適合管大檔案（binary diff 膨脹、LFS 慢又貴），但 ML 專案偏偏充滿大檔案。DVC 的做法是把 data/file hash 存進 Git（取代實際檔案），實際檔案存在本機快取或 remote storage（S3/GCS/SSH 等）。每次 `git commit` 時 DVC 會把對應的 `.dvc` 或 `dvc.lock` 檔案一起 commit，達到「資料跟著程式碼版本走」的效果。

## 為什麼值得研究

1. **MLOps 領域的典範專案** — DVC 是少數從 CLI 工具長成完整 MLOps 生態（資料版控、pipeline 編排、實驗追蹤、Studio 整合）的開源專案，其架構演進歷程本身就是很好的 case study。
2. **純 CLI + Python SDK 的 API 設計** — 沒有獨立 server／DB，所有狀態存在 Git 與本機檔案系統。這種「零 infra」設計對小團隊極友善，也暴露了當規模增長時的瓶頸。
3. **hash-based 內容定址實作** — 跟 Git 的內容定址概念相同，但針對大檔案（GB+）做了各種優化：symlink/hardlink cache、平行傳輸、worktree 模式。
4. **Pipeline as DAG 的純 Python 實作** — 用 `networkx` 管理 pipeline 的依賴 DAG，搭配 `celery` 做非同步佇列，這種組合在 2021-2023 年的 ML 工具中很常見。

## 任務與資料

| 面向 | 值 |
|---|---|
| 核心功能 | 資料版本控制、pipeline 編排、實驗管理 |
| 輸入 | CLI 指令 (`dvc add/push/pull/repro`) 或 Python API |
| 輸出 | `.dvc` / `dvc.yaml` / `dvc.lock` 檔案 & Remote storage 中的 content-addressed blob |
| 主要儲存層 | S3、GCS、Azure Blob、GDrive、SSH、WebDAV、HDFS 等（via fsspec）|
| 支援語言 | Python（CLI + SDK）|

## 與競品的比較

| 面向 | DVC | Git LFS | DVC (vs) MLflow | LakeFS |
|---|---|---|---|---|
| 資料儲存方式 | Hash-addressed cache + remote | Pointer file + transfer | 關注 artifact logging | Git-like branch on object storage |
| Pipeline 支援 | 內建 DAG pipeline + reproduce | 無 | 有（但較陽春） | 無 |
| 實驗追蹤 | 內建（dvc exp） | 無 | 核心功能 | 無 |
| 對 Git 的依賴 | 強依賴（狀態存在 Git ref） | 強依賴 | 可獨立使用 | 獨立（不須 Git） |
| 零 infra 部署 | ✅ 只要檔案系統 | ✅ 需 LFS server | ✅ 只要檔案系統 | ❌ 需 LakeFS server |
| 儲存效率 | de-duplication via hash | server-side de-dup | 不 focus 版本 | full copy on copy-on-write |

> 以上比較基於 DVC 的實作觀察：[`dvc/config_schema.py:46`](https://github.com/iterative/dvc/blob/06ff81c/dvc/config_schema.py#L46) 定義了 cache link type；[`pyproject.toml:138`](https://github.com/iterative/dvc/blob/06ff81c/pyproject.toml#L138) 顯示 fsspec entry point 如何支援多種 remote。

## 健康度信號

- ⭐ Stars: ~15,600（活躍度高）
- 📅 最後 commit: 2026-05-18（持續活躍開發中）
- 👥 主要維護者: Treeverse（母公司，也是 `dvc-studio` 的開發者）
- 🔬 許可證: Apache-2.0
- 🔄 相關專案: `dvc-data`、`dvc-objects`、`dvc-task`、`dvc-render`、`dvc-studio-client`（已拆成獨立套件）

## 我會在後續筆記中回答的問題

- DVC 的內容定址（hash）機制跟 Git 的差異在哪？為什麼可以對大檔案 scale？
- Pipeline 的 DAG 是怎麼構建、怎麼執行的？reproduce 的策略是什麼？
- 實驗追蹤（`dvc experiments`）是怎麼用 Git stash 實作的？
- 多層級的 config 系統（system → global → repo → local）怎麼設計？
- Remote storage 的 pluggable 架構是怎麼透過 fsspec 實作的？
