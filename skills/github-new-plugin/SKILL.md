---
name: github-new-plugin
description: 在 GitHub 的 plugin 專用 org 建立 Claude Code plugin repo，並透過 org 的 marketplace repo（<org>/marketplace，由 CI 掃描 org 自動產生清單）自動上架：plugin.json、註冊用 workflow、命名、visibility、下架、驗證上架、疑難排解。當使用者提到做一個 Claude Code plugin、把 skill 或 MCP server 打包成 plugin、發布 / 上架 plugin、新增 plugin repo、plugin 沒出現在 marketplace、marketplace 沒更新、或問 plugin 要放哪裡時使用。
---

# 建立 Plugin Repo 並上架

**共用規範在 `../../references/conventions.md`，建 repo 的完整流程用 `github-new-repo`
（A 檢查、B 掃描、D 驗證照它做）。本文件只講 plugin 特有的部分。**

## 這套機制

- 使用者有一個 **plugin 專用的 GitHub org**（以下稱 `<org>`），裡面有一個 **`<org>/marketplace`** repo
- marketplace 的 CI 會掃描 org 裡所有 public repo，**有 `.claude-plugin/plugin.json` 的就自動上架**，不需要手動註冊
- marketplace 名稱就是 org 名稱，安裝方式：

  ```
  /plugin marketplace add <org>/marketplace
  /plugin install <name>@<org>
  ```

運作細節見 `<org>/marketplace` 的 README。

## A0 先找出 `<org>`

| | 規則 | |
|---|---|---|
| A0-1 | **必要** | 從使用者已加入的 marketplace 找來源是 `<某 org>/marketplace` 的 GitHub marketplace |
| A0-2 | **必要** | 用 API 確認該 repo 存在，而且有 `.github/workflows/notify.yml`（代表是這套機制） |
| A0-3 | **停止** | 找不到、或找到不只一個時問使用者。**使用者還沒有這套機制**時，說明需要先建 org（只能在網頁建）與 marketplace repo，不要自己發明替代做法 |

```bash
claude plugin marketplace list                                   # 看 Source 是 GitHub 的那些
gh api repos/<org>/marketplace/contents/.github/workflows/notify.yml -q .name   # 預期 notify.yml
```

## 上架的三個硬條件

缺任何一個都不會出現在清單裡：

| | 條件 | 不符合會怎樣 |
|---|---|---|
| P1 | repo 在 **`<org>`** 底下，**public**、未封存、不是 fork | 掃不到 |
| P2 | 預設分支根目錄有 **`.claude-plugin/plugin.json`**，`name` 合規 | 判定「不是 plugin」或名稱不合規，略過並在 CI 發警告 |
| P3 | 有 `.github/workflows/marketplace.yml`（呼叫 C1） | 仍會被每日排程掃到，但不是即時 |

P2 是「是不是 plugin」的**唯一**判斷依據。

## 必要檔案

### `.claude-plugin/plugin.json`

```json
{
  "name": "<與 repo 名稱相同>",
  "description": "<一句話，會顯示在 marketplace 清單>",
  "version": "0.1.0",
  "license": "MIT",
  "keywords": ["...", "..."]
}
```

| 欄位 | 必要性 |
|---|---|
| `name` | **必要**。與 repo 名稱相同，也是安裝時的 id（`<name>@<org>`） |
| `description` | **必要**。清單上顯示的那一行 |
| `version` | **必要** ⚠️ 見下方 |
| `author`、`homepage`、`repository` | 選填。**問使用者要不要公開身分**，不要擅自填入姓名或 email |

- `name` 與 `description` 是 marketplace 清單的**唯一來源**。**repo 的 GitHub description
  也用同一段文字**，之後改描述兩邊一起改：

  ```bash
  gh repo edit <org>/<name> --description "$(python3 -c "import json; print(json.load(open('.claude-plugin/plugin.json'))['description'])")"
  ```

- ⚠️ **`version` 不可省略，而且每次發佈都要 bump**：
  - C1 只在 `plugin.json` 有異動時觸發，沒有 version 就沒有東西可改，**新 repo 第一次 push 後就不會再註冊**
  - 有寫 version 時，Claude Code 用它判斷有沒有新版 —— **沒 bump 的話使用者 `claude plugin update` 收不到更新**

### `.github/workflows/marketplace.yml`

```yaml
# 註冊到本 org 的 marketplace。改動 .claude-plugin/plugin.json 並 push 到 main 後清單會自動更新。
# 不要在本 repo 自帶 .claude-plugin/marketplace.json。
name: Register to marketplace
on:
  push: { branches: [main], paths: ['.claude-plugin/plugin.json'] }
  workflow_dispatch:
jobs:
  notify:
    uses: <org>/marketplace/.github/workflows/notify.yml@main
    secrets: inherit
```

`uses:` 不能用變數，`<org>` 要寫實際名稱。

### `README.md`

一句話說明、安裝指令（上面那兩行）、內容（skill／MCP／hook 各是什麼、什麼時候用）、需求（作業系統、要先裝什麼）。

## 命名

| | 規則 | |
|---|---|---|
| PN1 | **必要** | 照 `conventions.md` N1：小寫 kebab-case |
| PN2 | **停止** | **不能**以 `claude-`、`anthropic-`、`anthropics-`、`cc-plugin-` 開頭，也不能叫 `claude`、`anthropic`、`claude-code`、`claude-mods`（`claude plugin validate` 會拒絕，清單會略過） |
| PN3 | **必要** | 不要再帶 `plugin` —— `<org>/xxx-plugin` 讀起來重複 |
| PN4 | **必要** | 先看 org 裡現有 plugin 的命名，沿用同一套家族（例如 `<領域>-copilot`、`<工具>-mcp`、`<平台>-convention`） |

```bash
gh repo list <org> --limit 100 --json name -q '.[].name'
```

## 目錄結構

```
<plugin>/
  .claude-plugin/plugin.json          ← 必要
  .github/workflows/marketplace.yml   ← C1
  README.md
  skills/<skill-name>/SKILL.md        ← 注意是 skills/，不是 .claude/skills/
  references/                         ← 多個 skill 共用的內容
  .mcp.json                           ← 有 MCP server 時
  hooks/hooks.json  agents/  commands/ ← 視需要
```

- ⚠️ **skill 要放 `skills/`，不是 `.claude/skills/`**。後者是「打開這個 repo 時」才生效的專案層 skill，不會被打包。既有 repo 改造成 plugin 時要搬
- ⚠️ skill 引用 plugin 內其他檔案時**寫相對路徑**（例如 `../../references/x.md`）；`.mcp.json`、hooks 裡用 `${CLAUDE_PLUGIN_ROOT}`
- ⚠️ 根目錄的 `CLAUDE.md` **不會**被當成 plugin 的 context 載入

## 禁止

| | 規則 | |
|---|---|---|
| P4 | **停止** | **不可自帶 `.claude-plugin/marketplace.json`。** 註冊一律靠 C1；自帶會出現兩個來源，名稱與描述各自漂移。CI 掃到會警告 |
| P5 | **停止** | 工作上（雇主、客戶）開發的 plugin 放進個人 org 前，先確認公司規定（`conventions.md` O3） |

## 流程

1. A0 找出 `<org>`，再照 `github-new-repo` 的 A 段檢查（擁有者固定 `<org>`、visibility 固定 `public`）
2. 建好上面的必要檔案，本機先驗證：

   ```bash
   claude plugin validate .
   ```

   有錯誤（✘）才算不通過。沒填 `author` 會出現警告 —— 使用者選擇不公開身分時這是預期的，
   所以不用 `--strict`；有填 `author` 時改用 `--strict`，把其他警告也當錯誤處理。

3. 照 `github-new-repo` 的 B 段掃描，C 段建立（`gh repo create <org>/<name> --public ...`）
4. 照 `github-new-repo` 的 D 段驗證，再做下面的「驗證上架」

## 觸發時機

**只有 `.claude-plugin/plugin.json` 有異動時才會重新註冊。** 清單只含 name／source／description，
全部來自它 —— 改 skill、README 或程式碼不需要、也不會觸發（使用者 update 時會拿到新內容，
前提是 version 有 bump）。

不產生 commit 的改動（**改 repo 名稱、改 visibility、封存**）不會觸發 C1，手動跑一次：

```bash
gh workflow run marketplace.yml -R <org>/<name>       # 從 plugin repo 通知
gh workflow run regenerate.yml -R <org>/marketplace   # 或直接在 marketplace 重新產生
```

或等每日排程。

## 下架

| 做法 | 效果 |
|---|---|
| **封存（archive）** | 推薦。內容保留、可還原，下次重新產生時移除（排程或手動） |
| 刪掉 `.claude-plugin/plugin.json` | 會觸發 C1，**立即**從清單移除，repo 照常運作 |
| 改成 private | 下次重新產生時移除 |
| 刪除 repo | 會移除，但無法還原 —— **刪除是破壞性操作，要跟使用者確認** |

```bash
gh repo archive <org>/<name> --yes
gh repo unarchive <org>/<name> --yes
```

## 驗證上架

```bash
# 1. P 的 C1 有沒有成功
gh run list -R <org>/<name> --workflow marketplace.yml --limit 1

# 2. M 有沒有被觸發（event 應為 repository_dispatch）
gh run list -R <org>/marketplace --workflow regenerate.yml --limit 1

# 3. 清單裡有沒有
gh api repos/<org>/marketplace/contents/.claude-plugin/marketplace.json -q .content | base64 -d | grep '"name"'

# 4. 本機實裝
claude plugin marketplace update <org>
claude plugin install <name>@<org>
claude plugin list
```

## 疑難排解

| 症狀 | 原因 |
|---|---|
| plugin 沒出現在清單 | P1-P3 缺一；名稱不合規（看 M 最近一次執行的 warning）；或改的不是 `plugin.json`（不會觸發） |
| C1 失敗：`MARKETPLACE_DISPATCH_TOKEN` 是空的 | repo 不在 `<org>` 底下，或 workflow 沒寫 `secrets: inherit` |
| C1 失敗：HTTP 401 | org secret 的 token 過期 → 請使用者重建 token 並更新 org secret（見 marketplace README） |
| C1 失敗：HTTP 403／404 | token 沒有 `<org>/marketplace` 的 Contents 寫入權限 |
| 清單有，但使用者 update 沒拿到新內容 | `version` 沒 bump |
| 使用者裝了之後 `failed to load` | 他另外加了一個同名的 marketplace，把這個蓋掉了（`conventions.md` 實測細節：marketplace 撞名） |
| 移除 marketplace 後 plugin 不見了 | 正常：移除 marketplace 會連帶移除從它安裝的 plugin |
