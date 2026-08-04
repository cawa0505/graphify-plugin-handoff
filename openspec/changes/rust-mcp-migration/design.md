# Design: Stateful Workspace Caching & Thin Wrapper

## 1. 記憶體長駐 Workspace 定位 (Stateful Root Discovery)
- **架構設計**：
  - MCP Server 於啟動（Initialization）時接收並鎖定工作目錄。
  - 於記憶體中維護 `current_workspace_root` 快取狀態。
  - 任何對 `get_handoff` 或 `relayStatus` 的唯讀呼叫，一律直接從記憶體記憶（In-Memory Cache）進行讀取，達成物理級的 0 磁碟搜尋 I/O。

## 2. 寫入同步與安全 (Lazy-Write & File Lock)
- **Lazy-Write**：僅在 `save`、`close` 造成實質狀態變更時，非同步（Async）將變更刷入磁碟。
- **Atomic & Lock**：透過 `fs2` 取得排他鎖，並採用「寫入臨時檔 + OS rename」之 Atomic 寫入法，確保在任何硬體崩潰、多行程併發下資料 100% 安全。
