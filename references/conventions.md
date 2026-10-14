# GitHub 共用規範

`github-new-repo` 與 `github-new-plugin` 兩個 skill 都引用這份。**改規則只改這裡。**

文中的 `<owner>`、`<org>`、`<user>`、`<name>` 都是佔位符，執行時用實際的帳號、org、repo 名稱代入。

## 擁有者

先弄清楚使用者有哪些可以放 repo 的地方，再決定放哪裡：

```bash
gh api user -q .login                       # 個人帳號
gh api user/memberships/orgs -q '.[] | "\(.organization.login) \(.role)"'   # 所屬 org 與角色
```

| | 規則 | |
|---|---|---|
| O1 | **必要** | **依產物型態分，不依主題。** 使用者若有「專放 Claude plugin」的 org，plugin 就放那裡，不論它講的是什麼領域；不是 plugin 的東西不要放進去 |
| O2 | **停止** | 擁有者不明確（有多個 org、不確定哪個是 plugin 專用）時問使用者，**不要猜** |
| O3 | **停止** | 工作上（雇主、客戶）的東西放到個人帳號或個人 org 之前，先跟使用者確認公司規定 |

## 命名

GitHub 的 repo 只有一個名稱欄位，網址、clone、plugin 名稱都用它。

| | 規則 | |
|---|---|---|
| N1 | **必要** | repo 名稱全部小寫，單字之間用 `-`。不可有底線、大寫、空白 |
| N2 | **必要** | `description` 不可留空，一句話講清楚「這是什麼」。自己草擬直接送出，不清楚用途時再問 —— **不卡流程** |
| N3 | **必要** | 加 2–5 個 topics（小寫、`-` 連接），方便搜尋 |
| N4 | **必要** | 要開 GitHub Pages 的 repo，名稱就是網址路徑（`<user>.github.io/<name>/`），取名時想的是「網址長怎樣」 |

使用者既有的舊 repo 不符合 N1 時，**不要順手改名** —— 改名後舊網址只剩 redirect（見撞名 C2）。

## 撞名檢查

| | 規則 | |
|---|---|---|
| C1 | **停止** | 用 API 查目標名稱。**回 404 才算沒撞名**；有回傳時看 `full_name` —— 跟查詢的名稱不同，表示那是某個 repo **改名前的舊名稱**（GitHub 會把舊名稱 redirect 到新 repo） |
| C2 | **停止** | 撞到舊名稱 redirect 時要跟使用者確認：**建同名新 repo 會讓舊 redirect 失效**，指向舊名稱的連結、clone、書籤都會改指到新 repo |

```bash
gh api repos/<owner>/<name> -q .full_name    # 預期 404 Not Found
```

## Visibility

| | 規則 | |
|---|---|---|
| V1 | **停止** | `public` 或 `private` 必須明確指定並經使用者確認，**不可依賴預設值** |
| V2 | **必要** | 判斷依據是**內容**：含內部系統、內網主機、帳號、客戶資料、個資 → 不能 public |
| V3 | **必要** | 要上架到公開 marketplace 的 plugin 必須 public（private 不會被列出，別人也裝不起來） |

`internal` visibility 只有 GitHub Enterprise 有。

## Default branch

| | 規則 | |
|---|---|---|
| B1 | **必要** | 一律 `main`，不用問。建立後要驗證 —— 第一次 push 的分支會成為預設分支 |

## 個資與 git 歷史

| | 規則 | |
|---|---|---|
| H1 | **停止** | push 前掃描**被追蹤的檔案與整段歷史**有沒有祕密與**個資**（電話、身分證字號、地址），命中就停下來處理 |
| H2 | **必要** | 推到 public repo 就當作**已經外流**。刪檔案沒用（還在歷史裡）；改寫歷史也只能清掉自己 repo 的部分 |

已經推上去時的處理順序：

1. `git clone --mirror` 備份
2. `git filter-repo --replace-text` 改寫**所有分支**的歷史；含個資的二進位檔（PDF、圖片）用 `--path <檔案> --invert-paths` 整個移除
3. 掃描改寫後的**所有歷史**確認清乾淨，再強制 push 所有分支
4. 刪掉含舊內容的 Actions 執行紀錄與 artifact（例如 Pages 部署的 artifact 裡有舊網頁）
5. 本機 repo `reset --hard` 到新歷史，`git reflog expire --expire=now --all && git gc --prune=now`
6. 到 <https://support.github.com/contact/private-information> 申請清除孤立 commit 與快取。**別人的 fork 只能靠這個**
7. 其他電腦上的舊 clone 要重新 clone，不要 pull 後再 push（會把舊歷史推回去）

## 實測過的細節

**踩過才知道**的事。換版本後請重新驗證。來源：「實測」＝實際觀察到；「文件」＝官方文件、未實測。

| 事項 | 結論 | 來源 |
|---|---|---|
| 名稱是否可用 | `gh api users/<name>` 回 404 表示沒有帳號或 org 使用，但 GitHub 另有保留名稱，**最終以網頁建立時為準** | 實測 2026-10 |
| 建立 org | **API 不能建**（非 Enterprise），要在 <https://github.com/account/organizations/new> 手動建 | 文件 |
| repo 改名 | `gh repo rename <new> -R <owner>/<old> --yes`，舊名稱會 redirect 到新名稱 | 實測 |
| Pages：user site 與 project site 同路徑 | project site（repo `<name>` → `/<name>/`）**優先於** user site（`<user>.github.io` repo）裡同路徑的檔案 | 實測 2026-10 |
| Pages 用 Actions 部署 | `gh api -X POST repos/<owner>/<name>/pages -f build_type=workflow` 開啟。**先開 Pages 再 push**，否則第一次部署可能因 Pages 尚未啟用而失敗 | 實測（兩者幾乎同時時剛好成功，屬於競態） |
| `git filter-repo --replace-text` 的套用順序 | **先套字面規則，再套 `regex:` 規則**。字面規則先刪掉值，後面「刪整行」的 regex 就對不到，會留下空標籤 → 只用 regex，或跑第二輪 | 實測 git-filter-repo a40bce548d2c |
| 強制 push 改寫後的歷史 | Claude Code 的 auto mode 會擋下 `git push --force` → **請使用者自己執行**（提示列用 `!` 開頭）。指令要短，太長貼上時會被自動換行切斷 | 實測 |
| Actions 產生的 commit | 用 `GITHUB_TOKEN` push 的 commit **不會觸發新的 workflow** | 文件 |
| 可重用 workflow 的 `secrets: inherit` | 呼叫端與被呼叫的 workflow 要在**同一個 org**（或 enterprise） | 文件 |
| `repository_dispatch` 的權限 | fine-grained PAT 需要目標 repo 的 **Contents: write** | 文件 |
| org secret 與 Free 方案 | Free 方案的 org secret 只給 **public** repo 使用 | 文件 |
| Claude marketplace 撞名 | 同一台電腦加入**同名** marketplace 時，**後加的直接取代先加的**，只顯示一行提示。從舊來源裝的 plugin 若新來源沒有同名的 → `failed to load`；有同名的 → 下次更新改裝新來源的版本 | 實測 Claude Code 2.1.299 |
| 移除 marketplace | **會連帶移除從它安裝的 plugin**，加回來也不會自動裝回。切換來源的順序：uninstall → install → marketplace remove | 實測 |
| `claude plugin validate` | 不需要登入，CI 可直接用 `npx -y @anthropic-ai/claude-code plugin validate <dir>`；`--strict` 會把「marketplace 沒有 plugin」也當錯誤 | 實測 2.1.299 |
