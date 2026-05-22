---
repo: iterative/dvc
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 06ff81c
---

# DVC · 值得偷學的設計

## Pattern 1: 方法注入（Method Injection via Module Import）

**是什麼**: `Repo` class 透過在 class body 中 `from .module import method` 的方式「注入」方法。

```python
class Repo:
    from dvc.repo.add import add
    from dvc.repo.checkout import checkout
    from dvc.repo.reproduce import reproduce
    # ... 數十個方法
```

詳見 [`dvc/repo/__init__.py:67-91`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/__init__.py#L67-L91)。

**為什麼有效**: 這解決了「大 class 分解」的經典問題——每個方法可以獨立放在一個檔案中，有完整的 module docstring、type hints、專屬 imports，但仍然作為 `Repo` 的 instance method 暴露給外部呼叫者。

**替代方案**: 
- Mixin class（例如 `RepoAddMixin`、`RepoReproduceMixin`）：依賴層次更明確，但語法較重。
- 純函數接收 `repo` 參數：更 functional，但失去了 `self.repo` 的自然句法。
- `Repo` class 裡的 delegate method：多一層 wrapper。

**何時可以借用**: 當你有一個 class 有太多方法（>20）但又不適合拆成多個 class 時。尤其適合 CLI 工具的核心類別。

**注意事項**: 
- import assignment 在 class body 中會變成 class attribute（非 instance method），但 Python 的 descriptor protocol 讓 function 自動變成 method，所以這個 pattern 能運作。
- 靜態檢查工具（mypy / Pyright）對這個 pattern 支援不一。DVC 的做法是在 import 上加 `# type: ignore[misc]`。

---

## Pattern 2: 可組合的鎖裝飾器（Lock + Lock-free Run）

**是什麼**: `@locked` 裝飾器確保 repo 操作是 thread-safe，但實際執行外部 command 時會先 unlock repo。

```python
@contextmanager
def lock_repo(repo: "Repo"):
    depth = repo._lock_depth
    repo._lock_depth += 1
    try:
        if depth > 0:
            yield  # 已鎖定，不重複鎖
        else:
            with repo.lock:
                repo._reset()
                yield
                repo._reset()
    finally:
        repo._lock_depth = depth


@locked
@scm_context
def reproduce(self: "Repo", ...):
    ...

# 在 stage/run.py: 執行 command 時先 unlock
def run_stage(stage, ...):
    run = cmd_run if dry else unlocked_repo(cmd_run)
    run(stage, ...)
```

詳見 [`dvc/repo/__init__.py:37-61`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/__init__.py#L37-L61) 與 [`dvc/stage/run.py:181`](https://github.com/iterative/dvc/blob/06ff81c/dvc/stage/run.py#L181)。

**為什麼有效**: 長時間持有檔案鎖會讓其他 DVC process 無法操作。`repo.lock` 保護的是 cache/index 的讀寫一致性，但實際執行使用者 command 時不需要鎖。先 unlock 再執行 command，執行完再 re-lock，是典型的 **fine-grained locking** 策略。

**替代方案**: 用讀寫鎖（reader-writer lock）或完全不用鎖。前者太複雜，後者不安全。

**何時可以借用**: 任何需要在長時間操作中釋放資源鎖的場景——例如 server 在處理大量請求時持有 DB connection lock。

**注意事項**: 
- `unlocked_repo` 裝飾器在 stage 執行期間 unlock，此時其他 DVC process 可以操作 cache。這在理論上可能導致 race condition，但因為 DVC 的 cache 是 content-addressed（immutable write），破壞性操作只有 `gc` 而已。
- `_lock_depth` 計數器防止可重入鎖問題。

**不適用的情境**: 如果你保護的資源是可變的（mutable），不能在長時間操作中解鎖。

---

## Pattern 3: Git stash 作為實驗佇列（Git-native Queue）

**是什麼**: 實驗佇列（`dvc exp queue`）將排隊的實驗狀態存在 Git stash ref 中，而不是獨立的 message queue 或資料庫。

```python
class BaseStashQueue(ABC):
    """Naive Git-stash based experiment queue."""
    def __init__(self, repo, ref, failed_ref=None):
        self.ref = ref          # e.g., refs/exps/queue/default
        self.failed_ref = failed_ref
```

詳見 [`dvc/repo/experiments/queue/base.py:83`](https://github.com/iterative/dvc/blob/06ff81c/dvc/repo/experiments/queue/base.py#L83)。

**為什麼有效**: 
- 零 infra：不需要 Redis / RabbitMQ / DB
- 實驗狀態跟著 repo 走：clone repo 就 clone 了佇列狀態
- Git ref 本身就是 append-only log，天然支援佇列語意

**替代方案**: 
- Celery / Redis Queue：功能更完整，但多了維運成本
- 檔案系統 queue（寫 temp dir）：簡單但缺少 atomicity
- 純 SQLite：需要 schema，且狀態不跟 repo 綁定

**何時可以借用**: 當你需要一個輕量 async task queue，且 task 的狀態應該跟 git repo 綁定的時候。對 ML 專案的實驗排程特別合適。

**注意事項**:
- Stash queue 依賴 Git 操作，而 Git 不是為 queue 設計的——大量佇列操作可能導致 reflog 膨脹。
- 如果 git stash 操作出錯（如 merge conflict），可能影響整個佇列。
- Celery queue 實作是本機限定的（`LocalCeleryQueue`），不支援分散式 worker。

---

## Pattern 4: 內容定址 cache + 多種 link type 最佳化

**是什麼**: DVC 的 file cache 支援四種檔案連結方式來避免重複複製：

```python
supported_cache_type(types):
    unsupported = set(types) - {"reflink", "hardlink", "symlink", "copy"}
```

詳見 [`dvc/config_schema.py:46`](https://github.com/iterative/dvc/blob/06ff81c/dvc/config_schema.py#L46) 與 [`dvc/cachemgr.py`](https://github.com/iterative/dvc/blob/06ff81c/dvc/cachemgr.py)。

**為什麼有效**: ML 專案中經常幾十 GB 的檔案在不同版本間只是微調——每次 full copy 會大量浪費磁碟空間。link 機制的優化順序是：
1. `reflink`（CoW，最快的無複製方案，但需要 filesystem 支援，如 Btrfs / XFS / APFS）
2. `hardlink`（無複製，但不能跨裝置）
3. `symlink`（無複製，但 Git 不追蹤 symlink 目標）
4. `copy`（fallback，最慢但最相容）

**替代方案**: Git LFS 不管這個——它永遠是 pointer file + 實際 blob 分開存。DVC 的 link 機制是對已 checkout 的 working tree 做的最佳化。

**何時可以借用**: 任何需要在 working directory 中「呈現」大檔案的系統，但不想浪費 disk space 做重複複製。

**注意事項**:
- `symlink` 模式下，如果先 `dvc checkout` 再刪 cache（`dvc gc --workspace`），working tree 的檔案會斷鏈。
- Windows 上 symlink 需要 developer mode 或 admin 權限。
- reflink 是 filesystem-level 的特性，不是所有 OS/filesystem 都支援。

---

## Pattern 5: fsspec 作為統一的 storage 抽象層

**是什麼**: 所有 remote storage backend 都透過 `fsspec` 抽象，DVC 不需要知道 S3 跟 SSH 的差異。

```python
# dvc/fs/__init__.py
from fsspec.spec import AbstractFileSystem

# pyproject.toml: entry point
# [project.entry-points."fsspec.specs"]
# dvc = "dvc.api:DVCFileSystem"
```

詳見 [`pyproject.toml:138`](https://github.com/iterative/dvc/blob/06ff81c/pyproject.toml#L138) 與 [`dvc/fs/`](https://github.com/iterative/dvc/blob/06ff81c/dvc/fs/)。

**為什麼有效**: fsspec 統一了 POSIX-like file operations（`open`、`ls`、`glob`、`exists` 等），讓 DVC 在不修改核心程式碼的情況下支援 10+ 種 remote backend。DVC 自己的 `DVCFileSystem` 甚至可以當作 fsspec plugin 給其他工具使用（如 pandas 讀取 DVC 管理的檔案）。

**替代方案**: 
- 自定義 abstract class（前期簡單但後期難維護）
- Apache Hadoop Filesystem（太 heavy）
- 每個 backend 各寫一套（不易維護）

**何時可以借用**: 任何需要支援多種 storage backend 的系統——特別是 data pipeline／ML 工具。

**注意事項**:
- fsspec 的 exception 層次跟 DVC 的 `AuthError` / `ConfigError` 不直接相容，DVC 需要自己包一層轉換。
- 某些 fsspec backend 的效能差異極大（例如 `webdav` 明顯慢於 `s3`）。
- fsspec 的 async 支援在不同 backend 之間不一致，DVC 選擇全部用 sync API。
