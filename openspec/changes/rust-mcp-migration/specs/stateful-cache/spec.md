## ADDED Requirements

### Requirement: Single Walk-up Discovery
系統必須且僅能在初始化時，自當前目錄向上 walk-up 尋找第一個包含 `.relay/relay.toon` 或 `relay.json` 的目錄，並將結果儲存於快取記憶體中。

#### Scenario: Sub-directory execution without I/O penalty
- **WHEN** 伺服器被拉起且首次定位完成
- **THEN** 後續所有的唯讀工具（如 `relayStatus`）均應在 `< 1ms` 內，直接使用記憶體中的快取路徑回傳，不允許在硬碟上重複進行遍歷搜尋。
