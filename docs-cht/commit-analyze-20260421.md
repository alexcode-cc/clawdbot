# OpenClaw 提交分析報告 — 2026-04-21

> **分析範圍**：從 `9bc775d50953ed478bbddf3a85a64b70991f2071`（doc: 更新繁體中文文件版本號至 2026.4.20）至最新提交（`f788c88b4c`，docs: update 2026.4.21 release notes），共 **11 個非合併提交**，涵蓋 2026-04-20 至 2026-04-21。

---

## 目錄

1. [版本路徑概覽](#1-版本路徑概覽)
2. [新功能](#2-新功能)
3. [安全修復](#3-安全修復)
4. [頻道修復](#4-頻道修復)
5. [Browser 修復](#5-browser-修復)
6. [安裝與維運](#6-安裝與維運)
7. [摘要統計](#7-摘要統計)

---

## 1. 版本路徑概覽

```
2026.4.20   → 穩定版（上次更新終點）
2026.4.21   → 穩定版（6 項功能/修復 backport）
```

2026.4.21 是一個**小型 patch 版本**，主要包含幾個針對安全、頻道、瀏覽器和安裝問題的 backport 修復，以及 OpenAI 圖片生成預設模型升級。

---

## 2. 新功能

### OpenAI 圖片生成預設升級至 gpt-image-2

```
feat(openai): default images to gpt-image-2
```

**重要變更**：bundled image-generation provider 和 live media smoke tests 現在預設使用 **`gpt-image-2`**：

- `gpt-image-2` 為目前 OpenAI 圖片生成的最新推薦模型
- 圖片生成文件和 tool metadata 新增更新的 **2K/4K OpenAI 尺寸提示**（size hints）
- 現有使用 `gpt-image-1` 的配置仍然可用

配置示例：
```yaml
models:
  providers:
    openai:
      images:
        defaultModel: "gpt-image-2"   # 新預設（2026.4.21）
        # 可用 sizes: 1024x1024, 2048x2048 (2K), 4096x4096 (4K)
```

---

## 3. 安全修復

### Owner-Enforced Commands 身份驗證強化（重要）

```
fix: require owner identity for owner-enforced commands (#69774)
```

**重要安全修復**：當 `enforceOwnerForCommands=true` 且 `commands.ownerAllowFrom` 未設定時，非 owner 發送者不再能透過寬鬆的 fallback 觸達 owner-only 命令。

**修復前行為（漏洞）**：
- Wildcard channel `allowFrom` 或 empty owner-candidate list 會被視為足夠的 owner 身份驗證
- 非 owner 發送者可以透過寬鬆的 `allowFrom` 觸達 owner-only 命令

**修復後行為**：
- Owner-enforced commands 必須有 owner-candidate match 或 internal `operator.admin`
- Wildcard `allowFrom` 或空 owner-candidate list 不再被視為充分授權

> ⚠️ **升級注意**：若您使用 `enforceOwnerForCommands=true` 但未設定 `commands.ownerAllowFrom`，需要確認並補齊 owner allowlist 配置以維持正常操作。

---

## 4. 頻道修復

### Slack Thread Aliases 保留修復

```
fix(slack): preserve thread aliases in runtime outbound sends (#62947)
```

修復 Slack runtime outbound sends 中 thread aliases 丟失的問題：
- Generic runtime sends 現在在呼叫者提供 `threadTs` 時正確保持在預期的 Slack thread 中
- 修復多 workspace 或自訂 thread routing 場景下回覆落在錯誤 thread 的問題

---

## 5. Browser 修復

### Browser ax<N> Accessibility Refs 即時拒絕

```
fix(browser): reject ax<N> refs in act path instead of timing out (#69924)
```

修復 Browser `act` 路徑中無效的 `ax<N>` accessibility refs 處理方式：

**修復前**：無效的 `ax<N>` refs 會等待 browser action timeout 才失敗（可能需要幾十秒）
**修復後**：無效的 `ax<N>` refs 在 act path 中**立即被拒絕**，不再浪費 timeout 等待時間

---

## 6. 安裝與維運

### Doctor Plugin Runtime Dep 修復

```
Plugins/doctor: repair bundled plugin runtime dependencies from doctor paths
fix: adapt doctor runtime dep backport
fix: backport release install repairs
```

`openclaw doctor` 現在可以從 doctor 路徑修復 bundled plugin runtime 依賴：
- Packaged installs 可以在不需要廣泛 core 依賴安裝的情況下恢復缺失的 channel/provider 依賴
- 修復了 packaged 版本中 Discord、WhatsApp、Slack、Telegram 和 provider plugins 可能缺失依賴的問題

### Image Generation Fallback 日誌改善

```
fix(image-generation): log provider fallback failures
```

自動 provider fallback 之前，現在在 warn level 記錄失敗的 provider/model candidates：
- OpenAI 圖片生成失敗（即使後續 provider 成功）在 gateway log 中可見
- 更好的 fallback 可見性，便於診斷圖片生成問題

### npm node-domexception 別名修復

```
npm/install: mirror node-domexception alias into root package.json overrides
```

修復 npm install 顯示棄用警告的問題：
- 將 `node-domexception` alias 鏡像至 root `package.json` overrides
- 停止 `google-auth-library → gaxios → node-fetch → fetch-blob → node-domexception` 棄用鏈的警告

---

## 7. 摘要統計

### 版本里程碑

| 版本 | 日期 | 性質 |
|------|------|------|
| 2026.4.20 | 2026-04-20 | Stable（起始點，含 14 項新功能） |
| 2026.4.21 | 2026-04-21 | Stable（Patch 版本，6 項功能/修復） |

### 變更分類

| 類別 | 數量 |
|------|------|
| 新功能 | 1（gpt-image-2 預設） |
| 安全修復 | 1（owner-enforced commands） |
| 頻道修復 | 1（Slack thread aliases） |
| Browser 修復 | 1（ax refs 即時拒絕） |
| 安裝/維運 | 3（doctor repair, image log, npm alias） |

### 主要社群貢獻

| 貢獻者 | 主要貢獻 |
|--------|---------|
| @Patrick-Erichsen | Browser ax refs 修復 |
| @bek91 | Slack thread aliases 修復 |
| @drobison00 | Owner-enforced commands 安全修復 |
| @vincentkoc | npm node-domexception alias 修復 |
