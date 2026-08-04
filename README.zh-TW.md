# CodeRelayMcp

基於 Rust 開發的 Model Context Protocol (MCP) 原生伺服器，專為 Opencode Code Relay 子系統設計。本專案負責管理跨 Session 與跨儲存庫（Repo）的狀態交接，作為極速、超輕量的高性能核心引擎，完整取代舊有基於 Node.js 的 `@jimmyyen/opencode-code-relay-plugin`，並保持 100% 向後相容。

本專案之設計概念啟發自原創 [code-relay](https://github.com/yan5xu/code-relay) 專案。我們高度致敬並尊重原作者的絕佳創意，並將此概念升級至常駐型的極速 Rust MCP 架構，以無縫對接更廣大的編輯器與 AI Agent 生態系。

## 💡 核心特色

- **單一執行檔與極致效能**：純 Rust 打造，編譯後體積預期小於 10MB，記憶體開銷微乎其微。
- **記憶體常駐 Workspace 快取**：徹底解決舊版 Node.js 每次都要往上層目錄搜尋（walk-up disk search）導致的磁碟 I/O 延遲。本 MCP 伺服器在初始化時僅定位一次工作區根目錄並常駐於記憶體中，後續操作延遲低於 1ms。
- **雙軌混合記憶機制**：
  - **短期記憶 (Short-Term)**：採用對 Token 極度友善的 TOON (Token-Oriented Object Notation) 格式，儲存於 `.relay/relay.toon`，提供高速且精確的狀態交接。
  - **長期記憶 (Long-Term)**：整合 Qdrant 向量資料庫進行語意 RAG 檢索，可跨 Session 模糊搜尋過往的重大決策與技術脈絡。
- **標準 MCP 協定**：基於 Stdio 標準輸入輸出的 JSON-RPC 傳輸協定，相容於任何支援 MCP 的客戶端（如 OpenCode, Cursor, Claude Desktop, Roo Code）。
- **安全原子寫入**：寫入狀態時，採用「寫入臨時檔 + OS rename」的原子寫入機制，並結合 `fs2` 進行跨行程排他鎖定，防止並行寫入造成檔案毀損。

## 🤝 與 OpencodeCodeRelayPlugin 的相容關係

本專案作為底層的效能核心。為了實現無痛對接與向後相容：
- 我們同步提供一個 0 依賴的 **Slim Wrapper 版本** 的 Node.js 插件：[OpencodeCodeRelayPlugin](https://github.com/cawa0505/opencode-code-relay-plugin)。
- 當開發者或 CI/CD 腳本在工作區執行 `npx opencode-code-relay-plugin <command>` 時，該 Slim 插件只會作為輕量轉發層，透明地（Transparently）將命令拉起並轉交給 Rust 執行檔。
- 這能讓現有的工作流、AI Agent 指引在**不更改任何程式碼與設定**的情況下，直接獲得 100 倍以上的冷啟動效能提升。

## 🛠️ 開發與驗證命令

```bash
# 建置專案
cargo build

# 程式碼品質與靜態檢查
cargo check
cargo clippy

# 執行單元測試
cargo test
```

## ⚙️ 快速設定

設定採完全動態與相對路徑設計。若要在編輯器中透過 MCP 載入此服務，只需在設定檔中指向您的編譯產物：

```json
{
  "mcpServers": {
    "code-relay": {
      "command": "code-relay-mcp",
      "args": []
    }
  }
}
```

## 📐 架構設計與細節

更詳細的需求書、系統設計以及 OpenSpec 規範文件，請參閱本專案的 `openspec/` 目錄。

## 📄 授權條款

MIT / Apache 2.0
