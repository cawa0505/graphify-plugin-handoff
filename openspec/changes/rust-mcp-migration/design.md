# Design: Embedded Plugin Architecture — graphify-plugin-handoff

## 1. 架構定位：單一 Crate 作為 Graphify 內嵌型 Plugin

- **`graphify-plugin-handoff`**：單一 Cargo crate（`lib.rs` 為主，`src/bin/cli.rs` 為選用 CLI）
- **無獨立 MCP Server**：不透過 stdio JSON-RPC 與 AI client 直接溝通。
- **Plugin 介面契約**：實作 `graphify-core` 定義的 `GraphifyPlugin` trait。
  - `get_id(&self) -> &str`：唯一標識插件
  - `bind(&mut self, ctx: &WorkspaceContext)`：由 Graphify 注入 workspace_uuid + repo_paths
  - `get_workspace_uuid(&self) -> &str`：傳回當前 workspace 識別碼
  - `sync_toon(&mut self, prev_toon) -> Vec<u8>`：TOON 序列化/反序列化
  - `perform_handoff(&self)`：觸發主流程（保存、關閉、切換）

## 2. Root 解析模型：Graphify 的 Workspace Context 注入

- **解析規則**：完全由 Graphify 的 `WorkspaceIdentityManager` 決定。
- plugin 只需從 `WorkspaceContext` 取得：
  - `workspace_uuid`：跨 session 換裝的唯一鑑別碼
  - `repo_paths: HashMap<String, PathBuf>`：repo_name → repo_root
  - `primary_repo: Option<String>`：當前目標 repo
- **Fail-fast**：若 `repo` 不存在於 `repo_paths` → 立即回傳 `RepoNotFound` 錯誤。

## 3. 寫入同步與安全 (Lazy-Write & File Lock)

- **Lazy-Write**：僅在 `save`、`close` 造成實質狀態變更時，將變更刷入磁碟。
- **Atomic & Lock**：透過 `fs2` 取得排他鎖，採用「寫入臨時檔 + OS rename」的 Atomic 寫入法，確保資料 100% 安全。
- **狀態檔案位置**：`.relay/relay.toon`（TOON 格式）或 `relay.json`（TOON 序列化）。

## 4. TOON 狀態模型（由 Graphify 插件共用）

`RelayState`（serde 可序列化）：
```toml
session_id           # 當前 relay session
phase                # init | active | saved | closed
volatile_state       # 進行中的工作內容
open_threads         # 待處理的任務清單
specs_hash           # spec 檔案的 hash，用於偵測變更
confidence           # 0.0~1.0，表示狀態完整度
```

## 5. Plugin SDK 擴展點

- **GraphifyMCP 自動註冊**：Graphify の `graphify-mcp` 服務啟動時，掃描依賴於 `graphify-plugin-handoff` 的 crate，根據其公開函式自動產生對應的 MCP tools：
  - `relayInit`、`relaySave`、`relayResume`、`relayStatus`、`relaySwitch`、`relayAdd`、`relayClose`
- **跨 Plugin 間協作**：透過 `workspace_uuid` 將此 plugin 的 relay state 與 `graphify-plugin-opendoc`、`graphify-plugin-review` 等串連。
- **檔案位置**：plugin 僅負責邏輯；檔案讀寫與鎖機制在此 crate 內完成，GraphifyCore 不依賴此 crate 的檔案 I/O。

## 6. 依賴清單

```toml
[dependencies]
graphify-core = { path = "../../../GraphifyRust/graphify-core" }  # GraphifyPlugin trait
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
fs2 = "0.4"
```

> 備註：`graphify-core` 為 Git submodule / path dependency，由 GraphifyRust 團隊維護。

## 7. 開發指令

```bash
cargo check                 # 編譯檢查
cargo test --doc            # 單元測試
cargo clippy --all-targets  # 程式碼檢查
```