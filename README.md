# GitHub Convention

通用的 GitHub 作業規範，打包成 Claude Code plugin。裝了之後，Claude 在你要建 GitHub repo
或做 Claude Code plugin 時會自動照這些規則走。內容不綁定任何帳號或 org，執行時才判斷。

## 安裝

從收錄本 plugin 的 marketplace 安裝（`<org>` 是本 repo 所在的 GitHub org）：

```
/plugin marketplace add <org>/marketplace
/plugin install github-convention@<org>
```

## 內容

| Skill | 用在什麼時候 |
|---|---|
| `github-new-repo` | 在個人帳號或所屬 org 建 repo、把本機 repo 推上 GitHub、開 GitHub Pages 子站 |
| `github-new-plugin` | 在 plugin 專用 org 建 Claude Code plugin repo，並自動上架到該 org 的 marketplace |

兩個 skill 共用 `references/conventions.md`：擁有者怎麼判斷、命名、撞名（**含改名後的舊名稱 redirect**）、
visibility、push 前的祕密與**個資**掃描（包含 commit 的作者資訊）、已經推上去時怎麼清歷史，
以及一節**實測過的細節**。

`github-new-plugin` 搭配的 marketplace 機制：org 裡有一個 `marketplace` repo，CI 掃描 org 內所有
public repo，有 `.claude-plugin/plugin.json` 的自動上架。plugin repo 只要放一個呼叫
`<org>/marketplace/.github/workflows/notify.yml` 的 workflow 就能即時註冊。

## 為什麼值得打包

這些規則的價值在**踩過的坑**，例如：

- GitHub Pages 的 project site（`/<repo>/`）會**優先於** user site 裡同路徑的檔案
- 用 Actions 部署 Pages 要**先開 Pages 再 push**，不然是在賭競態
- `git filter-repo --replace-text` **先套字面規則再套 regex**，「刪整行」的規則會對不到
- 推上 public repo 的個資，改寫歷史也清不掉別人的 fork，只能找 GitHub Support
- 同名的 Claude marketplace **後加的會直接取代先加的**，已裝的 plugin 會 `failed to load`
- plugin 有寫 `version` 卻沒 bump，使用者 update 就收不到新內容

## 維護

- 新增規則 → 改對應 skill；**兩個 skill 都適用的** → 改 `references/conventions.md`
- 新的實測結論 → 加進 `conventions.md` 的「實測過的細節」，註明來源（實測／文件）與版本
- 內容保持通用：**不寫任何帳號、org、個人或公司名稱**，用 `<owner>`、`<org>` 等佔位符
- 發佈新版 → bump `.claude-plugin/plugin.json` 的 `version`

**改動 `.claude-plugin/plugin.json` 才會觸發 marketplace 重新註冊**（清單只含 name／source／description）。
改 skill 內容不需要重新註冊，但要 bump `version`，使用者更新時才拿得到。
