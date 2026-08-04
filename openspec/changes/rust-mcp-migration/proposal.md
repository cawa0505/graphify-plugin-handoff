# Change: Rust MCP Migration & Stateful Caching

## Why
1. 原版 Node.js 插件每次執行 CLI 命令或 Compacting 鉤子時，皆為無狀態（Stateless）進程，必須重複往上層目錄搜尋（walk-up）定位 `relay.json`，在多專案的工作區中會造成顯著的磁碟 I/O 延遲，影響 AI 代理回應效能。
2. node_modules 依賴與 Node.js 執行期體積龐大，不利於輕量跨平台部署。

## What Changes
1. 將核心功能重構為 Rust 原生 Stdio MCP 伺服器，擺脫 Node.js / node_modules。
2. 導入 **長駐型 Workspace Root 快取機制**。僅在啟動時執行一次磁碟定位，後續請求完全在記憶體中（Stateful Memory Cache）直接響應。
3. 採用雙軌記憶（TOON 短期記憶 + Qdrant 長期語意 RAG），提供極低 Token 開銷與跨會話決策記憶。

## Impact
- **效能提升**：啟動與查詢延遲從 >300ms 降至 <1ms（免磁碟搜尋）。
- **完全相容**：提供 Slim 版 Node.js 包裝器，維持舊版 `npx opencode-code-relay-plugin` 命令 100% 向後相容。
