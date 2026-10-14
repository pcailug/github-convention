---
name: github-new-repo
description: 在 GitHub 上建立新的 repo（使用者的個人帳號或所屬 org）：擁有者判斷、命名規範、visibility 決策、撞名檢查（含改名 redirect）、push 前的祕密與個資掃描、建立後驗證，以及要用 GitHub Pages 時的設定順序。當使用者提到建立 / 開 / 新增 GitHub repo、專案、把本機 repo 推上 GitHub、gh repo create、要一個新的 git repo、開一個 GitHub Pages 子站、或問 repo 該怎麼命名時使用。要建的是 Claude Code plugin 時改用 github-new-plugin。
---

# 建立 GitHub Repo

**共用規範在 `../../references/conventions.md`（擁有者 O1-O3、命名 N1-N4、撞名 C1-C2、
visibility V1-V3、branch B1、個資 H1-H2、實測過的細節），動手前先讀。**

**必要** = 照做，**停止** = 等使用者回覆才能繼續。

## 第零條

每次只建**一個** repo。撞名、權限或欄位有疑義就停下來處理。

要建的是 Claude Code plugin → 改用 `github-new-plugin`，它會再回來引用本文件的 A、B、D 段。

## Repo 資料卡（填完才動手）

```
擁有者          <個人帳號或 org>          （conventions.md O1-O3）
名稱            <name>                    （小寫 kebab-case）
描述            <一句話>
visibility      public / private          （必須明確）
topics          <t1>, <t2>, <t3>
homepage        <有網站才填>
default branch  main
GitHub Pages    否 / branch 部署 / Actions 部署
首次內容        現有本機 repo / 空 repo + README
```

## A 動手前要先檢查

**全程只讀不寫。**

| | 規則 | |
|---|---|---|
| A1 | **必要** | `gh auth status` 確認登入的是**哪個帳號**、token 有 `repo` scope |
| A2 | **停止** | 決定擁有者（`conventions.md` O1-O3），不明確就問 |
| A3 | **停止** | 撞名檢查（`conventions.md` C1-C2，**注意舊名稱 redirect**） |
| A4 | **必要** | 擁有者是 org 時，確認自己是成員且有建 repo 的權限 |

```bash
gh auth status
gh api user -q .login
gh api user/memberships/orgs -q '.[] | "\(.organization.login) \(.role)"'
gh api repos/<owner>/<name> -q .full_name                 # 預期 404
```

## B 推送前的祕密與個資掃描

**只要會 push 本機既有的 repo，這段就不能跳。** 推上 public repo 就當作已經外流，
事後刪檔案也還在歷史裡（`conventions.md` H2）。

| | 規則 | |
|---|---|---|
| B1 | **必要** | 先有 `.gitignore`，擋掉憑證、金鑰、`.env`、`node_modules`、建置產物 |
| B2 | **停止** | 掃描**被追蹤的檔案**（不是工作目錄）有沒有祕密，命中就停 |
| B3 | **停止** | 掃描**個資**：電話、身分證字號、住址、私人 email。履歷、自我介紹、設定檔最常中 |
| B4 | **必要** | 用 `git ls-files` 核對實際要推的檔案，沒有非預期的大檔或產物 |
| B5 | **停止** | 本機 repo 已有歷史時，**整段歷史**也要掃（B2、B3 只看目前的檔案）；**commit 的作者資訊**也是個資，確認 `git log --format='%an <%ae>'` 是使用者願意公開的身分 |

```bash
# B2 祕密
git ls-files -z | xargs -0 grep -nIiE \
  "BEGIN (RSA |EC |OPENSSH |ENCRYPTED )?PRIVATE KEY|password|passwd|secret|token|api[_-]?key|credential" \
  && echo "⚠️ 有命中，停下來" || echo "無命中"

# B3 個資：電話號碼的樣式依地區調整（以下為台灣手機與身分證字號）
git ls-files -z | xargs -0 grep -nIE "09[0-9]{2}[- ]?[0-9]{3}[- ]?[0-9]{3}|\b[A-Z][12][0-9]{8}\b" \
  && echo "⚠️ 有命中，停下來" || echo "無命中"

# B4
git ls-files

# B5 整段歷史與作者資訊
git log --all --oneline -G"09[0-9]{2}[- ]?[0-9]{3}[- ]?[0-9]{3}|PRIVATE KEY|api[_-]?key"
git log --all --format='%an <%ae>' | sort -u
```

> 命中不一定是問題（例如程式碼裡的變數名 `token`），逐筆看過、回報使用者判斷。
> 公鑰、憑證指紋是**公開資訊**；private key、token、密碼、個資才是。

## C 執行

```bash
gh repo create <owner>/<name> --public --description "<描述>" --homepage "<homepage>" \
  --source . --remote origin --push
gh repo edit <owner>/<name> --add-topic <t1> --add-topic <t2>
```

- `--public` 或 `--private` 必須明確寫（V1）
- 沒有 homepage 就省略 `--homepage`
- 空 repo：改用 `--add-readme`，不帶 `--source --remote --push`
- 推之前確認 `git branch --show-current` 是 `main`，不是就先改名（B1）

### 要用 GitHub Pages 時

| 部署方式 | 適用 | 設定 |
|---|---|---|
| branch 部署 | 純靜態、不用建置 | `gh api -X POST repos/<owner>/<name>/pages -f "source[branch]=main" -f "source[path]=/"`；不用 Jekyll 時根目錄放 `.nojekyll` |
| Actions 部署 | 要先建置 | `gh api -X POST repos/<owner>/<name>/pages -f build_type=workflow`，**在 push workflow 之前開**（實測細節：競態） |

個人帳號的 repo 會出現在 `https://<user>.github.io/<name>/`，**而且優先於** user site
（`<user>.github.io` repo）裡同路徑的檔案（實測細節）。使用者有入口網站或連結清單時，提醒他加上新網址。

⚠️ **含祕密的值不要寫在指令列**（例如 `gh secret set X --body ...`）—— 會進 shell history，
也可能被工具的憑證防護擋下。請使用者自己在網頁或互動式 `gh secret set X` 設定。

## D 建立後驗證

| | 規則 | |
|---|---|---|
| D1 | **必要** | 重抓 repo，確認名稱／描述／visibility／預設分支／topics 與資料卡完全一致 |
| D2 | **必要** | 抓遠端檔案樹，跟 `git ls-files` 逐項核對 —— **確認沒有非預期的檔案上去** |
| D3 | **必要** | 確認有 README：description 只有一句話，打開 repo 的人要知道這是什麼、怎麼用 |
| D4 | **必要** | 有 Pages 時，等部署完成後實際打開網址確認 200 |
| D5 | **必要** | 把網頁 URL、clone URL（有 Pages 再加網站 URL）回報使用者 |

```bash
gh repo view <owner>/<name> --json name,description,visibility,defaultBranchRef,repositoryTopics,url,sshUrl
gh api "repos/<owner>/<name>/git/trees/main?recursive=1" -q '.tree[] | select(.type=="blob") | .path' | sort > /tmp/remote.txt
git ls-files | sort | diff - /tmp/remote.txt && echo "檔案一致"
```

---

**任何標成「停止」的規則都要等使用者回覆才能繼續** —— 不要用猜的跳過去、
不要對 API 錯誤盲目重試、不要順手多建資料卡以外的 repo。
