---
repo: HKUDS/CLI-Anything
file: 3-key-patterns
studied_at: 2026-05-21
commit_sha: 436a4f5
---

# CLI-Anything · 值得偷學的設計

## Pattern 1: Method-Driven Code Generation（方法論驅動的程式碼產生）

**是什麼**: 不是寫一個程式來產生程式碼，而是寫一份 SOP 文件讓 AI agent 去照著做。CLI-Anything 的 HARNESS.md 就是這份 SOP。

**為什麼有效**: 傳統程式碼產生器（cookiecutter、yeoman、rails generators）的輸出品質被 template 的彈性限制——超越 template 就需要改寫產生器本身。Method-driven 則把 AI agent 當作通用執行器，SOP 只定義「做什麼」的步驟和品質標準，「怎麼做」由 agent 根據每個軟體的特性判斷。

**程式碼位置**:
- SOP 文件：[`cli-anything-plugin/HARNESS.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/HARNESS.md#L1-L8)
- 命令入口（強制讀取 SOP）：[`cli-anything-plugin/commands/cli-anything.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/commands/cli-anything.md#L5-L7)

**何時可以借用**:
- 你有一系列重複性但需要針對每次情境微調的開發工作
- 輸出品質可以用「檢查清單」來定義，而不需要嚴格的結構約束
- 你信任 AI agent 勝過信任 template engine

**替代方案**: Cookie-cutter template（保證結構一致，但缺乏彈性）；Script-based generator（彈性由程式邏輯決定，維護成本高）。

**注意事項**: 
- SOP 必須寫得極度明確——Agent 不像人會「讀空氣」
- 需要出口品質檢查機制（CLI-Anything 用強制測試覆蓋 + SKILL.md generator 來做這件事）
- 不是所有工作都適合——對輸出格式有嚴格要求（如 JSON schema）的場景，傳統 template 更可靠

---

## Pattern 2: PEP 420 Namespace Packages 做多套件生態

**是什麼**: 每個產出的 CLI 是一個獨立的 pip 套件，透過 PEP 420 implicit namespace packages 共享 `cli_anything.*` 根命名空間。`cli_anything/` 目錄沒有 `__init__.py`。

**為什麼有效**: 
- 50+ 套件可以獨立安裝、獨立版本化
- `from cli_anything.blender.core.session import Session` 這種跨套件 import 可以直接運作
- 沒有中央 registry 或 plugin 發現機制的開銷
- 新增一個 CLI 套件不需要修改任何既有套件

**程式碼位置**: 
- `setup.py` 配置：[`cli-anything-plugin/commands/cli-anything.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/commands/cli-anything.md#L80-L85)
  ```python
  packages=find_namespace_packages(include=["cli_anything.*"])
  ```

**何時可以借用**:
- 你有一個 namespace 下需要多個獨立發布的套件
- 套件間需要互相 import 但不是互相依賴
- 你想避免中央 plugin registry 的維護成本

**替代方案**: 單一 monolith 套件（管理簡單但版本耦合）；entry_points-based plugin system（如 `pre-commit` 或 `pytest` 的 plugin 機制，需要中央套件載入 plugin）

**注意事項**:
- `PEP 420` 需要所有參與套件都使用 `find_namespace_packages` 而不是 `find_packages`
- 匯入時若多個套件貢獻相同子包名，行為是 undefined
- 使用者如果只裝一個 CLI，這個設計沒有明顯好處——它的價值在生態規模

---

## Pattern 3: Unified REPL Skin（統一終端皮膚）

**是什麼**: 所有產出的 CLI 共用同一份 `repl_skin.py`，提供完全一致的 banner、prompt、訊息、表格、進度條風格。

**為什麼有效**: 
- 零學習成本——使用者（或 agent）從一個 CLI 換到另一個 CLI，互動方式完全相同
- 每個 CLI 有軟體專屬色系（Blender=orange, GIMP=warm orange, Inkscape=blue），品牌感強但不破壞統一性
- 核心樣式用 ANSI escape codes 硬編碼，零外部依賴；進階功能（prompt_toolkit）是 optional

**程式碼位置**: [`cli-anything-plugin/repl_skin.py`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/repl_skin.py#L42-L51)

```python
_ACCENT_COLORS = {
    "gimp":        "\033[38;5;214m",   # warm orange
    "blender":     "\033[38;5;208m",   # deep orange
    "inkscape":    "\033[38;5;39m",    # bright blue
    "audacity":    "\033[38;5;33m",    # navy blue
    "libreoffice": "\033[38;5;40m",    # green
    ...
}
```

**何時可以借用**:
- 一個工具鏈產出多個 CLI，你想確保一致的 UX
- 你希望 terminal 套件有「品牌感」但不是每個都從頭設計 UI
- 你需要在 terminal 中展示專業感

**替代方案**: 每個 CLI 各自設計 UI（彈性大但 UX 不一致）；使用 rich / textual 等 library（功能多但增加依賴）

**注意事項**:
- 複製貼上模式（每個 harness 手動 copy repl_skin.py）不是 DRY——CLI-Anything 選擇這種方式是因為每個 CLI 都是獨立 pip 套件，不能共享同一個 repl_skin 模組
- 改 repl_skin 需要更新所有已產出的 CLI（沒有自動更新機制）

---

## Pattern 4: 雙 Registry 設計（Internal + Public）

**是什麼**: CLI-Anything 維護兩個獨立的 registry JSON 檔案——`registry.json`（約 50 個 harness CLI，社群貢獻）和 `public_registry.json`（第三方官方 CLI，如 feishu、minimax、wecom）。

**為什麼有效**: 
- harness CLI 和第三方 CLI 的審查標準不同。Harness 需要經過 PR review、測試驗證才合併；public CLI 只是註冊一個「我知道這個 CLI 存在」的條目
- `_source` tag 區分來源，installer 根據來源決定用 pip 還是 npm
- 兩個 registry 可以獨立更新、獨立快取

**程式碼位置**: 
- Registry 讀取：[`cli-hub/cli_hub/registry.py`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-hub/cli_hub/registry.py#L1-L15)
- Registry JSON：[`registry.json`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/registry.json#L1-L10)
- Public registry：[`public_registry.json`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/public_registry.json#L1-L10)

**何時可以借用**:
- 你的生態系統同時有社群貢獻和第三方整合兩種來源
- 不同來源的品質門檻不同
- 安裝策略因來源而異（pip vs npm vs brew）

**替代方案**: 單一 registry + 標籤過濾（簡單但混雜）；全部只有一種貢獻方式（門檻統一但可能限制生態）

**注意事項**:
- 兩個 registry 的 schema 需要保持一致，否則 registry.py 的合集函式會變複雜
- CLI-Hub 的 install dispatcher（`installer.py`）需要根據 `_source` 正確選擇安裝策略——這是關鍵風險點

---

## Pattern 5: 自我修復的 Output Verification（輸出不被信任）

**是什麼**: Phase 5 的測試寫入規範要求 E2E 測試**不能只檢查 exit code**——必須以程式方式驗證實際輸出檔案的內容（magic bytes、ZIP 結構、pixel 數值、duration）。

**為什麼有效**: AI agent 產生的程式碼常見的錯誤是「介面對了但內容錯了」。一個 CLI 可能 exit(0) 但產生一個空的 PDF、不完整的 MP4、或格式錯誤的 XML。強制 magic byte 驗證讓這種錯誤在 Phase 5 就被發現。

**程式碼位置**: [`cli-anything-plugin/HARNESS.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/HARNESS.md#L151-L163)

```python
# Not just exit code — verify output:
# - Magic bytes: %PDF-, PK\x03\x04
# - ZIP structure for OOXML
# - Pixel-level analysis for video/images
# - Duration/format checks
```

**何時可以借用**:
- Agent 產出的程式碼有「外部可見輸出」（檔案、API response、渲染結果）
- 你經歷過「測試通過但產出不能用」的 bug
- 你想要 agent-generated code 的 CI 迴圈

**替代方案**: 只測 exit code（便宜但不可靠）；人工 review（可靠但慢）

**注意事項**:
- 每個 harness 都要實作自己的 magic byte 檢查——HARNESS.md 給了 pattern，但沒有統一的驗證 helper
- 對於某些輸出格式（如場景 XML），schema validation 比 magic byte 更可靠但更難實作

---

## API 設計品味的觀察

CLI-Anything 的 API 設計有幾個一貫的品味:

1. **偏好 `--json` flag 雙模輸出**: 不是用 `--format json` 參數，而是一個獨立的 `--json` flag，切換整個 CLI 到 machine-readable 模式。設計哲學是「人看 table、agent 看 JSON」，而不是讓人指定格式
2. **REPL 作為預設行為**: `click.Group(invoke_without_command=True)` + 無子命令時自動進入 REPL。這讓 CLI 同時滿足「單次操作」和「互動式工作」兩個場景
3. **Session 作為隱式上下文**: 不像傳統 CLI 每次獨立執行，CLI-Anything 的 CLI 會記住當前 project、開啟的檔案、undo/redo stack。這更接近 GUI 的操作模式

## 對相容性的態度

- 由於 CLI-Anything 的產出是獨立的套件，每個 harness 可以獨立演進
- `registry.json` 裡每個條目有獨立的版本號碼，更新不會破壞其他 CLI
- 主要的 breaking change 風險在 HARNESS.md 本身——如果方法論改版，已產出的 harness 不會自動更新產生方法
