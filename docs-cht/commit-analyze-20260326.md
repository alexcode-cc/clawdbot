# OpenClaw 提交分析報告 — 2026-03-26

> **分析範圍**：從 `f698774324`（2026.3.23 release）至 `cff6dc94e3`（2026.3.24 release），共 **463 個非合併提交**，涉及 **1,603 個檔案**、新增 79,320 行、刪除 38,867 行。

---

## 目錄

1. [重大變更 (Breaking / Major Changes)](#1-重大變更)
2. [新功能 (New Features)](#2-新功能)
3. [安全強化 (Security)](#3-安全強化)
4. [頻道改進 (Channels)](#4-頻道改進)
5. [Gateway 與 CLI](#5-gateway-與-cli)
6. [模型與提供者](#6-模型與提供者)
7. [Plugin 與 Skills](#7-plugin-與-skills)
8. [記憶與效能](#8-記憶與效能)
9. [Control UI](#9-control-ui)
10. [Docker / Container](#10-docker--container)
11. [Cron 排程](#11-cron-排程)
12. [平台相容性](#12-平台相容性)
13. [重構](#13-重構)
14. [測試與 CI](#14-測試與-ci)
15. [文件與發布](#15-文件與發布)
16. [摘要統計](#16-摘要統計)

---

## 1. 重大變更

本次更新無 breaking changes（`feat!` / `refactor!`），屬於穩定性和功能增補的發布週期。

版本跨度：`2026.3.23` → `2026.3.24`（含 `2026.3.23-2`、`2026.3.24-beta.1`、`2026.3.24-beta.2`）。

---

## 2. 新功能

### 2.1 MiniMax Image Generation Provider

```
feat(minimax): add image generation provider and trim model catalog to M2.7 (#54487)
```

MiniMax 新增 Image Generation provider，並精簡模型目錄至 M2.7 系列。透過 Plugin Capability 系統註冊為 image generation provider。

---

### 2.2 OpenAI 相容端點（Gateway）

```
feat(gateway): add missing OpenAI-compatible endpoints (models and embeddings) (#53992)
feat(gateway): make openai compatibility agent-first
```

Gateway 新增 OpenAI 相容的 `/v1/models` 和 `/v1/embeddings` 端點：
- **Agent-first**：相容層以 Agent 為中心（而非全域），回傳的模型列表和行為對齊當前 Agent 的設定
- 使第三方工具可透過 OpenAI 相容 API 與 OpenClaw Gateway 互動

---

### 2.3 `/tools` 執行時可用性視圖

```
feat: add /tools runtime availability view (#54088)
```

新增 `/tools` 指令，可在對話中查看所有工具的執行時可用性狀態。

---

### 2.4 Discord autoThreadName 'generated' 策略

```
feat(discord): add autoThreadName 'generated' strategy (#43366)
```

Discord 新增 `autoThreadName: "generated"` 策略，由 LLM 根據對話內容自動生成 thread 名稱。

---

### 2.5 Container 內 CLI 定向

```
feat(cli): support targeting running containerized openclaw instances (#52651)
```

CLI 新增支援直接定向至正在運行的容器化 OpenClaw 實例，簡化 Docker/Podman 部署的管理。

---

### 2.6 Control UI 大幅打磨

```
feat(ui): Control UI polish — skills revamp, markdown preview, agent workspace, macOS config tree (#53411)
```

Control UI 全面打磨：
- **Skills 面板翻新**：全新的 Skills 管理介面
- **Markdown 預覽**：設定值的 Markdown 預覽
- **Agent workspace**：工作區檔案瀏覽
- **macOS config tree**：macOS 風格的設定樹狀結構

---

### 2.7 CSP Inline Script Hashes

```
feat(csp): support inline script hashes in Control UI CSP (#53307)
```

Control UI 的 Content Security Policy 支援 inline script hashes，提升安全性的同時允許必要的 inline scripts。

---

### 2.8 MS Teams 訊息編輯與刪除

```
msteams: add message edit and delete support (#49925)
msteams: extract structured quote/reply context (#51647)
```

MS Teams 新增：
- 訊息編輯和刪除支援
- 結構化引用/回覆上下文提取

---

## 3. 安全強化

### 3.1 Sandbox 修復

```
fix(sandbox): honor sandbox alsoAllow and explicit re-allows (#54492)
fix: close sandbox media root bypass for mediaUrl/fileUrl aliases (#54034)
fix: enforce sandbox visibility for session_status ids
```

- **alsoAllow 遵從**：沙箱的 `alsoAllow` 和明確 re-allow 規則現在正確執行
- **媒體根繞過修復**：封閉 mediaUrl/fileUrl 別名的沙箱媒體根繞過
- **session_status 可見性**：強制沙箱化的 session_status ID 可見性

---

### 3.2 Channel Auth 安全

```
fix(cli): guard channel-auth against prototype-chain pollution and control-char injection
fix(cli): auto-select login-capable auth channels (#53254)
fix: blcok non-owner authorized senders from chaning /send policy (#53994)
```

- **Prototype 污染防護**：防止 channel-auth 的原型鏈污染
- **控制字元注入防護**：防止 channel-auth 中的控制字元注入
- **/send 政策保護**：阻止非擁有者的授權發送者更改 `/send` 政策

---

### 3.3 Aisle 安全發現修復

```
fix(security): resolve Aisle findings — skill installer validation, terminal sanitization, URL scheme allowlisting (#53471)
```

修復 Aisle 安全審計發現：
- Skill 安裝器驗證強化
- Terminal 輸出消毒
- URL scheme 白名單

---

### 3.4 Config 值遮蔽

```
fix(ui): redact sensitive config values in diff panel
```

Control UI 的 diff 面板中遮蔽敏感設定值。

---

### 3.5 Plugin SDK 子路徑消毒

```
Plugins: sanitize sdk export subpaths
Plugins: trust only startup cli sdk roots
```

- 消毒 Plugin SDK 匯出子路徑
- 僅信任啟動時的 CLI SDK roots

---

### 3.6 Exec Policy 分離

```
fix: split exec and policy resolution for wrapper trust (#53134)
fix(infra): preserve blocked dispatch policy target
Infra: preserve wrapper executable for multiplexer trust
```

將 exec 執行與 policy resolution 分離，改善 wrapper trust 的安全邊界。

---

### 3.7 其他安全修復

| 修復 | 說明 | PR |
|------|------|-----|
| Command auth SecretRef | SecretRef 解析修復 | #52791 |
| Live model auth gating | 統一 live model auth gating | — |
| Hash inline scripts | data-src 屬性的 inline scripts 雜湊 | — |
| Provider inference fail-closed | 錯誤 allowlist 時 fail closed | — |

---

## 4. 頻道改進

### 4.1 WhatsApp

| 改進 | 說明 | PR |
|------|------|-----|
| FutureProofMessage 解包 | 解包 botInvokeMessage 恢復 reply-to-bot 偵測 | — |
| selfLid 比對 | 使用 selfLid 進行群組 reply-to-bot 隱式提及比對 | — |
| selfLid 讀取 | 從 creds.json 讀取 selfLid | — |
| 非同步讀取 | 使用 async fs.promises.readFile | — |
| Login tool | 避免 eager login tool runtime 存取 | — |
| Active listener | 恢復 WhatsApp active listener singleton | #54232 |
| Identity handling | 統一 whatsapp identity handling | — |

---

### 4.2 Telegram

| 改進 | 說明 | PR |
|------|------|-----|
| Thread create topic payload | thread create 使用 topic payload | #54336 |
| Topic-create 路由 | CLI 路由 thread create 至 topic-create | — |
| 無效照片預檢 | 預檢無效的 telegram photos | #52545 |
| Forum topic 路由 | 保留 forum topic last-route delivery | #53052 |
| Native commands | 保持 native commands 在 runtime snapshot | #53179 |
| Pairing code blocks | 範圍限定 pairing code blocks | #52784 |
| 403 membership errors | 改善 403 membership delivery 錯誤訊息 | #53635 |

---

### 4.3 Discord

| 改進 | 說明 | PR |
|------|------|-----|
| autoThreadName 'generated' | LLM 自動生成 thread 名稱 | #43366 |
| /think autocomplete | 從 session model 解析 /think autocomplete | #49176 |
| Safety error listener | teardown 後移除 safety error listener 防止洩漏 | — |
| Gateway crash 防護 | 防止未捕獲的 gateway errors 導致 process crash | — |
| Native command deploy | 記錄被拒絕的 native command deploy 失敗 | #54118 |
| autoArchiveDuration 型別 | 新增 autoArchiveDuration 至 config 型別 | #43427 |
| Gateway supervision | 集中化 Discord gateway supervision | — |

---

### 4.4 Feishu / Lark

| 改進 | 說明 | PR |
|------|------|-----|
| Inbound timestamps | 使用 message create_time 作為 inbound timestamps | #52809 |
| WebSocket 關閉 | monitor stop 時關閉 WebSocket 連線 | #52844 |
| Open-group docs | 完成 feishu open-group 文件和 baselines | #54058 |
| requireMention 預設 | groupPolicy open 時 requireMention 預設為 false | — |
| Retry helper | 打磨 feishu retry helper | #43788 |
| Group message drops | 防止 bot-info probe 逾時時的靜默群組訊息丟棄 | — |
| Mention policy | 對齊 pairing replies、daemon hints 和 feishu mention policy | — |

---

### 4.5 MS Teams

| 改進 | 說明 | PR |
|------|------|-----|
| Message edit/delete | 新增訊息編輯和刪除支援 | #49925 |
| Quote/reply context | 提取結構化引用/回覆上下文 | #51647 |

---

## 5. Gateway 與 CLI

### 5.1 Gateway 改善

| 項目 | 說明 | PR |
|------|------|-----|
| OpenAI 相容端點 | 新增 /v1/models 和 /v1/embeddings | #53992 |
| Agent-first 相容 | OpenAI 相容層以 Agent 為中心 | — |
| Connect challenge timeout | 提升預設 connect challenge timeout | — |
| Channel registry pin | 啟動時 pin channel registry 以 survive registry swaps | — |
| Restart sentinel | 重啟後喚醒 session 並保留 thread routing | #53940 |
| Runtime state 關閉 | 啟動 abort 時關閉 runtime state | — |
| Channel cache 失效 | re-pin 時失效 channel caches | — |
| Minimal registry | 保持最小 gateway channel registry live | #53944 |
| Handshake timeout | 統一 gateway handshake timeout wiring | — |
| Plugin bootstrap | 拆分 gateway plugin bootstrap 和 registry surfaces | — |

---

### 5.2 CLI 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Container 定向 | 支援定向至容器化 OpenClaw 實例 | #52651 |
| /tools 視圖 | 新增 /tools runtime availability view | #54088 |
| Config validate | 重啟前驗證 config + 從實際狀態衍生 loaded | — |

---

### 5.3 Doctor 改善

| 項目 | 說明 |
|------|------|
| Service config repairs | 更新期間跳過 service config repairs |
| No-restart update | 更新 doctor fixes 時保留 no-restart |
| --fix non-interactive | 非互動模式下遵從 --fix |
| Config clobber forensics | 新增 config clobber 鑑識 |

---

### 5.4 Daemon 改善

```
fix(daemon): bootstrap stopped service on `gateway start`
refactor: centralize daemon service start state flow
```

- `gateway start` 現在會 bootstrap 已停止的 service
- 集中化 daemon service 啟動狀態流程

---

## 6. 模型與提供者

### 6.1 MiniMax

```
feat(minimax): add image generation provider and trim model catalog to M2.7
```

- 新增 image generation provider
- 模型目錄精簡至 M2.7 系列

### 6.2 Codex Auth 改善

```
auth: derive codex oauth profile ids from jwt claims
refactor(openai): extract codex auth identity helper
refactor(auth): separate profile ids from email metadata
fix(auth): protect fresher codex reauth state
refactor(auth): unify external CLI credential sync
```

Codex OAuth 認證系統重大改善：
- 從 JWT claims 衍生 profile IDs
- 提取 codex auth identity helper
- 分離 profile IDs 和 email metadata
- 保護較新的 reauth 狀態
- 統一外部 CLI credential 同步

### 6.3 其他

- 修復 assistant replay content 的 canonicalization
- 統一 live model auth gating

---

## 7. Plugin 與 Skills

### 7.1 Plugin 改善

| 項目 | 說明 |
|------|------|
| SDK export 消毒 | 消毒 sdk export subpaths |
| 啟動 SDK roots | 僅信任啟動時的 cli sdk roots |
| Terminal hook semantics | 強制 terminal hook 決策語意 |
| Channel registry reload | Deferred plugin reload 後 re-pin channel registry |
| Web search scope | 限定 web search plugin 載入範圍 |
| Plugin lazy-load | 5 項 lazy-load：interactive state、fallback state、command registry、hook runner、runtime registry |
| Install config policy | 集中化 plugin install config policy |
| Plugin version | 正規化 bundled plugin version reporting |
| Runtime version | 使用 runtime version 進行 plugin API 相容性檢查 |

### 7.2 Skills 改善

| 項目 | 說明 | PR |
|------|------|-----|
| Slug 驗證 | Skill slug 驗證收緊至 ASCII-only | — |
| Legacy clawhub | 保留 legacy clawhub skill updates | #53206 |
| ClawHub URL | 修正系統 prompt 中的 ClawHub URL | — |

---

## 8. 記憶與效能

### 8.1 記憶系統效能

```
perf(memory): builtin sqlite hot-path follow-ups (#53939)
perf(sqlite): use existence probes for empty memory search
perf(memory): avoid eager provider init on empty search
```

- **SQLite 熱路徑優化**：builtin SQLite 的 hot-path 後續改善
- **空搜尋存在探測**：空搜尋時使用 existence probes 避免完整查詢
- **延遲 provider 初始化**：空搜尋時避免 eager provider 初始化

### 8.2 LanceDB Proxy

```
fix: bootstrap proxy for LanceDB embeddings (#54119)
```

修復 LanceDB embeddings 的 proxy bootstrap，確保在需要 proxy 的環境中正常運作。

### 8.3 Plugin 載入效能

```
perf(plugins): scope web search plugin loads
refactor(plugins): make interactive state lazy
refactor(plugins): make command registry lazy
refactor(plugins): make hook runner global lazy
refactor(plugins): make runtime registry lazy
refactor(gateway): make plugin fallback state lazy
```

6 項 plugin 系統 lazy-load 優化，降低啟動時間。

---

## 9. Control UI

| 改進 | 說明 | PR |
|------|------|-----|
| Skills 翻新 | 全新 Skills 管理面板 | #53411 |
| Markdown 預覽 | 設定值 Markdown 預覽 | #53411 |
| Agent workspace | 工作區檔案瀏覽 | #53411 |
| macOS config tree | macOS 風格設定樹 | #53411 |
| Config diff 遮蔽 | 敏感值遮蔽 | — |
| Model provider 解析 | 從 catalog 解析 model provider | — |
| Default account | stale default account ID 時返回 null | — |
| Channel configured | 從 default account 衍生 channel configured 狀態 | — |
| Context-notice SVG | 修正 SVG icon 寬高 | — |
| CSS variables | 統一 border-radius 使用 CSS variables | #53238 |
| Chat compact retries | guard compact retries | — |
| Compact transport | 收緊 compact transport handling | — |
| Theme/config/usage | UI 清晰度改善 | #53272 |
| Agent file preview | 打磨 agent file preview 和 usage popovers | #53382 |
| CSP inline hashes | 支援 inline script hashes | #53307 |

---

## 10. Docker / Container

| 改進 | 說明 | PR |
|------|------|-----|
| Container CLI 定向 | 支援定向至容器化實例 | #52651 |
| Open WebUI smoke | 新增 Open WebUI docker smoke 測試 | — |
| Bin 複製順序 | docker install 前複製 openclaw bin | — |
| Setup namespace loop | 修復 docker setup CLI namespace loop | #53385 |
| Localhost origin | Docker seed localhost control UI origin | — |
| CLI phases | 釐清 docker setup CLI phases | — |

---

## 11. Cron 排程

| 改進 | 說明 | PR |
|------|------|-----|
| --tz + --at | 修復 --tz 與 --at 在 one-shot 工作中的搭配 | — |
| 隔離 delivery awareness | queue 隔離 delivery awareness | — |
| Best-effort delivery | 釐清 cron best-effort partial delivery 狀態 | #42535 |
| Zoned cron at-times | 拒絕不存在的 zoned cron at-times | — |
| Heartbeat prompt | 壓制 cron 觸發的 embedded runs 的 heartbeat prompt | — |
| NO_REPLY delivery | cron coverage 和 NO_REPLY delivery 修復 | #53366 |

---

## 12. 平台相容性

### macOS

| 項目 | 說明 |
|------|------|
| Edge 瀏覽器偵測 | 新增 Edge LaunchServices bundle IDs 用於 macOS 預設瀏覽器偵測 |
| osascript fallback | 覆蓋 macOS Edge osascript fallback 路徑 |

### Windows

| 項目 | 說明 |
|------|------|
| Parallels install | 記錄 Windows Parallels 安裝經驗 |
| Agent quoting | 修正 Windows Parallels agent 引號 |

### Node.js

```
fix(runtime): support Node 22.14 installs
fix(runtime): anchor bundled plugin npm staging to active node
```

- 支援 Node 22.14 安裝
- 錨定 bundled plugin npm staging 至活動 node

---

## 13. 重構

本次共 **42 個重構提交**，主要方向：

| 方向 | 說明 |
|------|------|
| Plugin lazy-load | 5 項 plugin 系統 lazy-load |
| Auth 分離 | profile IDs、email metadata、external CLI credential 分離 |
| Gateway 拆分 | plugin bootstrap 和 registry surfaces 拆分 |
| Exec policy | exec policy 和 execution targets 分離 |
| WhatsApp identity | 統一 identity handling |
| Daemon state | 集中化 service start state flow |
| ClawHub | 拆分 tracked update flows、清理 compatibility validation |
| Docker | 釐清 setup CLI phases |
| Cron | 提取 cron schedule 和 test runner helpers |

---

## 14. 測試與 CI

### 測試（122 個 test/ci 提交）

主要方向：
- **Suite 合併**：大量 `collapse` 開頭的提交，合併 Telegram、MS Teams、Nextcloud Talk、Synology Chat、Google Chat 的 helper/state/monitor suites
- **Open WebUI smoke**：新增 Docker Open WebUI 整合測試
- **Thread-safe isolation**：聲音通話臨時儲存隔離、安全審計 home skill 隔離
- **Windows 安全**：media local roots fixture Windows 安全化
- **Moonshot/MiniMax probe**：穩定 live probes
- **Shard 重平衡**：CI shard 重平衡和 critical path 縮短

### CI 改善

| 項目 | 說明 |
|------|------|
| Shard 重平衡 | 重平衡 sharded test lanes |
| Channel shard workers | 限制 channel shard workers |
| Main critical path | 縮短 main critical path |
| PR artifacts | 重用 PR artifacts |
| Required checks | 提前啟動 required checks |
| E2E docker cache | 恢復 e2e docker cache boundary |

---

## 15. 文件與發布

### 發布

| 版本 | 說明 |
|------|------|
| `2026.3.23-2` | 修正發布 |
| `2026.3.24-beta.1` | Beta 1 |
| `2026.3.24-beta.2` | Beta 2 |
| `2026.3.24` | 正式發布 |
| macOS appcast | 發布 2026.3.23 mac appcast |

### 文件更新

- 格式化 changelog for release
- 修正 2026.3.22 和 2026.3.23 release notes
- 新增遺漏的 changelog 項目
- 按使用者影響排序 changelog
- 更新 mac release 自動化指引
- 刷新 plugin-sdk api baseline
- 記錄 Windows Parallels 安裝經驗

---

## 16. 摘要統計

| 指標 | 數值 |
|------|------|
| 總提交數 | **463** |
| 影響檔案數 | **1,603** |
| 新增行數 | 79,320 |
| 刪除行數 | 38,867 |
| 淨增行數 | +40,453 |
| 版本跨度 | `2026.3.23` → `2026.3.24` |
| 新功能 | **8 項** |
| 安全修復 | **12+ 項** |
| Bug 修復 | **227** |
| 重構提交 | **~42** |
| 測試/CI 提交 | **~122** |
| 文件提交 | **~10** |
| 發布相關 | **~7** |

### 按模組分布

| 模組 | 變更強度 |
|------|---------|
| 測試 suite 合併 | ■■■■■ 極高 |
| Gateway（OpenAI 相容 + 穩定性） | ■■■■ 高 |
| WhatsApp（selfLid + identity） | ■■■ 中高 |
| Plugin lazy-load | ■■■ 中高 |
| Control UI 打磨 | ■■■ 中高 |
| Security（sandbox + auth + CSP） | ■■■ 中 |
| Telegram（topic + photos） | ■■■ 中 |
| Feishu（timestamps + groups） | ■■ 中 |
| Discord（autoThread + stability） | ■■ 中 |
| Codex Auth 重構 | ■■ 中 |
| Docker / Container | ■■ 低中 |
| Cron 排程 | ■■ 低中 |
| MS Teams（edit/delete） | ■ 低 |
| 記憶效能 | ■ 低 |

### 與前次分析（2026-03-23）比較

| 指標 | 2026-03-23 | 2026-03-26 | 變化 |
|------|-----------|-----------|------|
| 提交數 | 2,694 | 463 | -83%（正常週期） |
| 新功能 | 60+ | 8 | 穩定性聚焦 |
| 安全修復 | 25+ | 12+ | 持續強化 |
| Bug 修復 | 651 | 227 | 正常規模 |
| 重構 | ~441 | ~42 | 趨緩 |

本次更新的突出特點：

1. **OpenAI 相容 Gateway**：新增 `/v1/models`、`/v1/embeddings` 端點，Agent-first 設計
2. **WhatsApp selfLid 修復**：完整修復 reply-to-bot 偵測（FutureProofMessage 解包 + selfLid 比對）
3. **Control UI 大幅打磨**：Skills 翻新、Markdown 預覽、workspace 瀏覽、config tree
4. **Plugin 系統 6 項 lazy-load**：顯著降低啟動開銷
5. **Sandbox 安全修復**：alsoAllow 遵從、媒體根繞過封閉、session_status 可見性
6. **測試 suite 大規模合併**：Telegram、MS Teams、Nextcloud Talk 等多頻道測試整合

---

*本文件由 AI 自動分析生成，分析基準點：`f698774324`（2026.3.23）→ `cff6dc94e3`（2026.3.24）*
