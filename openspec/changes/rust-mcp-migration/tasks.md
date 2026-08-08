# Tasks: Embedded Plugin Architecture & Verification

## Overview

此計畫以 **文件先行** 原則執行 — 文件與規格永遠先於實作，每一階段皆須通過對應驗證門檻（validation gate）才進入下一階段。

## Roadmap

### Task 1: 同步 Graphify Plugin Trait 與 crate 結構
- [ ] 等待 GraphifyRust 完成 `graphify-core/src/plugin.rs` 中的 `GraphifyPlugin` trait 定義
- [ ] 更新 `openspec/design.md`：揭露 Plugin 介設計，強調「Embedded Crate」模式
- [ ] 更新 `openspec/proposal.md`：描述 Plugin SDK，與 GraphifyMCP 工具註冊機制
- [ ] 於 `graphify-plugin-handoff` 建立 `src/lib.rs`，實作 `GraphifyPlugin` trait
- [ ] `Cargo.toml`：`[lib]` 為主，optional `[[bin]]` 僅供本地測試
- [ ] 依賴：`serde`、`thiserror`、`fs2`（檔案鎖）、`graphify-core`（trait 來源，path/git dep）

### Task 2: TOON 狀態模型與 Workspace Root 解析
- [ ] `src/state.rs`：`RelayState` (session_id, phase, volatile, open_threads, specs_hash, confidence)
- [ ] `src/workspace.rs`：單次 walk-up 找 `.graphify-workspace`，快取到記憶體 `HashMap<String, PathBuf>`

### Task 3: relay* 核心邏輯（供 graphify-mcp 呼叫）
- [ ] `src/relay.rs`：`save`, `resume`, `status`, `switch`, `add`, `close`, `init`
- [ ] 使用 `fs2::FileExt::lock_exclusive` + temp-write + rename

### Task 4: 測試與驗證
- [ ] `cargo test`：狀態讀寫、workspace 解析、檔案鎖併發
- [ ] 整合測試：在 GraphifyRust workspace 加入此 crate，驗證 `graphify-mcp` 能自動註冊 relay tools

### Task 5: 效能驗證 (Performance Benchmark)
- [ ] 使用 `criterion` 或 `std::time::Instant` 量測：
  - `relayResume` / `relaySave` 單次呼叫時間
  - workspace 常駐快取 vs 每次 walk-up 的延遲對比
- [ ] 效能目標：記憶體快取路徑 < 1ms

### Task 6: 整合與驗證 (INTEGRATION.md)
- [ ] 與 `opendoc-mcp`：確認 `workspace_uuid` 作為 RAG 過濾鍵
- [ ] 與 `graphify`：確認 `.toon` 在 graph 節點間的傳遞
- [ ] 整合測試：在 GraphifyRust workspace 加入此 crate，驗證 `graphify-mcp` 自動註冊 relay tools

### Task 7: 雙端 README 互聯與 Parity 驗證
- [ ] 更新 `README.md` 及 `README.zh-TW.md`，預告內嵌 plugin 定位
- [ ] 建立 `openspec/` 與實作的雙向連結（docs ↔ code）
- [ ] 執行 parity 驗證：doc 中描述的功能必須對應至實作

### Task 8: 編譯與發布 (可選)
- [ ] 編譯各平台二進位產物（僅供本地測試；正式版由 GraphifyRust 打包）
- [ ] 更新 `CHANGELOG.md`（若適用）

### Task 9: 工具整合研究 (INTEGRATION.md)
- [ ] 整合 use cases：code-relay 與 opendoc-mcp / graphify 的搭配場景
- [ ] 整合測試：輸出 `INTEGRATION.md`（若尚未存在）

## [待討論] 未決項目
- ~~Thin Node.js wrapper（`npx opencode-code-relay-plugin` 向後相容）是否保留~~ → **決議：不保留**。slim wrapper 的唯一用途是將獨立 MCP server 註冊進 opencode.json，該路徑在嵌入式架構下已不存在；`PLUGIN_SLIM.md` 已刪除。
- 是否保留原 plugin repo（OpencodeCodeRelayPlugin）作為向後相容橋接 → 視實際使用需求而定，但無 MCP 註冊需求即無橋接必要。
