# 第 6 章　Codespace 的雲端版 VS Code 及 Ubuntu CLI 入門 — 實作指令集

> 本檔收錄第 6 章實作時要複製貼上的指令與程式碼；觀念說明請參考課堂簡報。

---

## 使用前必讀

### 本檔出現的內容類型

| 類型 | 是什麼 | 怎麼用 |
|---|---|---|
| **CLI Command** | 直接在終端機輸入、立即執行的單行或多行指令 | 複製整個區塊，貼到 VS Code 下方的 **Terminal**，按 Enter |
| **Bash Script** | 多行指令組成的腳本，通常會寫成 `.sh` 檔再執行 | 依說明貼到終端機（會自動建立檔案）或存檔後執行 |
| **Python Script** | Python 程式碼，存成 `.py` 檔後由 Python 執行 | 用編輯器開啟檔案，整段貼上並存檔 |
| **設定檔** | JSON、YAML 等設定內容（非指令） | 用編輯器開啟檔案，整段貼上並存檔 |

> 本章以 CLI Command 為主，沒有獨立的 Python Script；第 7 章才開始撰寫 Python 程式。

### 貼上規則

1. **一次只貼一個區塊**，看完輸出再貼下一個。
2. 標示 ⚠️ **貼上前請替換** 的區塊，要先把 `<...>` 或範例值換成你自己的內容；`<` `>` 符號本身也要刪掉。
3. 標示 🔒 **單獨執行** 的區塊需要你輸入內容，**不可與其他指令一起貼上**。
4. 可複製區塊內刻意不放註解；每個指令的意義都寫在區塊上方的說明表格。

---

## A. Ubuntu CLI 入門

### 6-1 確認你所在的環境

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `nproc && free -h` | 顯示 CPU 核心數與記憶體大小，也就是底層 VM 分配到的規格 |
| `cat /etc/os-release` | 顯示 Dev Container 的作業系統版本 |
| `docker version` | 顯示 Docker 版本；**同時看到 Client 與 Server** 才代表 Docker 可用 |
| `echo $CODESPACE_NAME` | 顯示這台 codespace 的名稱 |

**複製貼上**：

```bash
nproc && free -h
cat /etc/os-release
docker version
echo $CODESPACE_NAME
```

---

### 6-2 絕對路徑與相對路徑

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `cat /workspaces/codespaces-lab/README.md` | **絕對路徑**：從 `/` 開始寫完整路徑，不論目前在哪個資料夾，結果都一樣 |
| `cat README.md` | **相對路徑**：結果取決於你「現在在哪個資料夾」 |

⚠️ **貼上前請替換**：`codespaces-lab` 換成你的 repo 名稱（可先執行 `ls /workspaces` 查看）。

> 原則：指令的結果不應取決於隱藏狀態。凡是牽涉設定檔的指令，本課程一律使用絕對路徑。

**複製貼上**：

```bash
cat /workspaces/codespaces-lab/README.md
```

```bash
cat README.md
```

---

### 6-3 環境變數只活在當下的終端機

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `export MY_NAME=Arthur` | 設定一個名為 `MY_NAME` 的環境變數 |
| `echo $MY_NAME` | 顯示該變數的值，應顯示 `Arthur` |

**步驟 1**：在目前的終端機執行，會顯示 `Arthur`。

```bash
export MY_NAME=Arthur
echo $MY_NAME
```

**步驟 2**：按終端機右上角的 **+** 開一個新的終端機分頁，再執行下面這行，結果是**空的**。

```bash
echo $MY_NAME
```

> 需要長期保存的設定要寫進檔案，或存成 Codespaces Secret（第 5 章）。

---

### 6-4 整段貼上指令的陷阱

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `read -rp "請輸入名字：" NAME` | 顯示提示文字，等待你輸入，並把輸入的內容存進變數 `NAME` |
| `echo "hello $NAME"` | 顯示 `hello` 加上剛才輸入的名字 |

如果把這兩行一起貼上，第二行會被當成 `NAME` 的值，而不會被執行。正確做法是**分兩次貼上**：

🔒 **單獨執行**：貼上後輸入你的名字，按 Enter。

```bash
read -rp "請輸入名字：" NAME
```

接著再貼上：

```bash
echo "hello $NAME"
```

> 原則：**互動輸入的指令一律單獨貼上執行。**

---

### 6-5 CLI 綜合練習

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `cd ~ && pwd` | 回到家目錄，並顯示目前位置 |
| `mkdir -p ~/practice && cd ~/practice` | 建立練習資料夾並進入 |
| `echo "hello codespace" > note.txt` | `>`：把文字**寫入**檔案（檔案已存在時會覆蓋） |
| `echo "second line" >> note.txt` | `>>`：把文字**附加**到檔案結尾 |
| `cat note.txt` | 顯示檔案內容 |
| `grep second note.txt` | 找出含有 `second` 的行 |
| `chmod 600 note.txt && ls -l note.txt` | 設為只有自己能讀寫，並查看權限，應為 `-rw-------` |

**複製貼上**：

```bash
cd ~ && pwd
mkdir -p ~/practice && cd ~/practice
echo "hello codespace" > note.txt
echo "second line" >> note.txt
cat note.txt
grep second note.txt
chmod 600 note.txt && ls -l note.txt
```

---

### 6-6 清除練習資料夾

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `cd ~` | 回到家目錄（不能在要刪除的資料夾裡面刪除它） |
| `rm -r ~/practice` | 刪除整個練習資料夾。⚠️ Linux **沒有資源回收筒**，刪除後無法復原 |

**複製貼上**：

```bash
cd ~ && rm -r ~/practice
```

✅ **驗收**：能說出目前所在位置、建立與刪除檔案、設定並讀取環境變數。

---

## B. CLI 實作：驗證 API 金鑰

> 前提：第 5 章已把 Google AI Studio 金鑰存成 Codespaces Secret `LLM_API_KEY`，並已重新啟動 codespace。

### 6-7 確認金鑰已載入

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `echo "${LLM_API_KEY:0:4}****"` | 只顯示金鑰的**前 4 碼**，確認變數有值。**永遠不要輸出完整金鑰** |

顯示 `AIza****` 代表已載入；只顯示 `****` 代表 Secret 沒有生效，請依第 5 章的步驟重新啟動 codespace。

**複製貼上**：

```bash
echo "${LLM_API_KEY:0:4}****"
```

---

### 6-8 步驟 1：列出這把金鑰能用的模型

**類型**：CLI Command

**指令意義**：

| 指令片段 | 意義 |
|---|---|
| `curl -s "https://.../v1beta/models?key=$LLM_API_KEY"` | 呼叫 Google 的 ListModels API，取得可用模型清單（JSON） |
| `\|`（管線） | 把前一個指令的輸出交給下一個指令處理 |
| `python3 -c "..."` | 用一行 Python 解析 JSON，逐行印出模型名稱與支援的方法 |

**排錯的第一步永遠是這個**：先確認金鑰能用哪些模型，再決定要呼叫哪一個。

**複製貼上**：

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key=$LLM_API_KEY" \
  | python3 -c "import sys,json; [print(m['name'], m.get('supportedGenerationMethods')) for m in json.load(sys.stdin).get('models', [])]"
```

---

### 6-9 步驟 2：把 model ID 存成變數

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `export MODEL="..."` | 把步驟 1 清單中的 model ID 存成終端機變數 `MODEL` |

⚠️ **貼上前請替換**：`gemini-3.x-flash-lite` 換成步驟 1 清單上 Gemini 3.x Flash-Lite 的**實際 ID**，並去掉前綴 `models/`。

> `MODEL` 是終端機變數，**開新的終端機分頁就會消失**，需要重新執行這一行。

**複製貼上**：

```bash
export MODEL="gemini-3.x-flash-lite"
```

---

### 6-10 步驟 3：用 OpenAI 相容端點呼叫模型

**類型**：CLI Command

**指令意義**：

| 指令片段 | 意義 |
|---|---|
| `curl -sS <網址>` | 發出 HTTP 請求；`-s` 不顯示進度條，`-S` 失敗時仍顯示錯誤 |
| `-H "Authorization: Bearer $LLM_API_KEY"` | 以 Bearer token 方式帶上金鑰 |
| `-H "Content-Type: application/json"` | 宣告送出的內容是 JSON |
| `-d "{...}"` | 請求內容：指定模型，並送出一則 user 訊息 |
| 行尾的 `\` | 指令太長時換行續寫，整段仍是同一個指令 |

成功時會回傳含有模型回答的 JSON。

**複製貼上**：

```bash
curl -sS https://generativelanguage.googleapis.com/v1beta/openai/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"用一句話解釋容器\"}]}"
```

---

### 6-11 步驟 4：確認模型是否支援 system 角色

**類型**：CLI Command

**指令意義**：與步驟 3 相同，但多送一則 `system` 訊息（系統提示）。

| 結果 | 意義 |
|---|---|
| 正常回答 | 支援 system 角色。Gemini 支援 |
| 回 400 且訊息提到 system | 不支援 system 角色（例如 Gemma）；第 7 章程式的 `auto` 模式會自動處理 |

**複製貼上**：

```bash
curl -sS https://generativelanguage.googleapis.com/v1beta/openai/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"system\",\"content\":\"用繁體中文回答\"},{\"role\":\"user\",\"content\":\"hi\"}]}"
```

**看錯誤碼判斷成因**：

| HTTP / status | 意思 | 處理 |
|---|---|---|
| 400 `INVALID_ARGUMENT` | `API key not valid` → 金鑰錯誤（Google 對無效金鑰回 400，不是 401）；提到 model → 名稱格式錯誤；提到 system → 不支援 system 角色 | 重新複製金鑰 / 對照步驟 1 的清單 |
| 401 / 403 | 認證或權限問題 | 確認 API 已啟用；重建金鑰 |
| 404 `NOT_FOUND` | 模型名稱不存在 | 從步驟 1 的清單重新複製 model ID |
| 429 `RESOURCE_EXHAUSTED` | 額度用盡 | 等待重置，或改用其他模型 |

✅ **驗收**：步驟 1 列得出模型清單，步驟 3 回傳含有模型回答的 JSON。

---

## C. Commit 按鈕的陷阱與工作區設定

### 6-12 建立並開啟工作區設定檔

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `mkdir -p .vscode` | 在 repo 根目錄建立 `.vscode` 資料夾 |
| `code .vscode/settings.json` | 在編輯器中開啟（不存在時建立）`settings.json` |

> 請先確認終端機位於 repo 根目錄（執行 `pwd`，應為 `/workspaces/<repo>`）。

**複製貼上**：

```bash
mkdir -p .vscode
code .vscode/settings.json
```

---

### 6-13 `.vscode/settings.json` — 釘死 Git 面板行為

**類型**：設定檔（貼到編輯器，不是貼到終端機）

**用途**：讓 Source Control 面板的每個動作與一條 Git 指令一對一，避免「commit 自己跑到遠端」。存檔即生效，不需要重建容器。

| 設定 | 作用 |
|---|---|
| `git.postCommitCommand: none` | commit 後不自動 push / sync |
| `git.rememberPostCommitCommand: false` | 不記住上次用過的 Commit & Push |
| `git.enableSmartCommit: false` | 暫存區為空時不自動 stage 全部 |
| `git.showActionButton` | 藍色主按鈕只負責 commit，不會變身成 Sync / Publish |
| `git.confirmSync: true` | Sync 前跳出確認，避免無聲的 pull |
| `git.autofetch: true` | 背景自動 fetch，狀態列才看得到遠端的新進度 |

**貼到編輯器並存檔**：

```jsonc
// .vscode/settings.json —— 工作區設定：隨 repo 進版控，存檔即生效，不需要重建容器
{
  "git.postCommitCommand": "none",          // commit 後不自動 push / sync
  "git.rememberPostCommitCommand": false,   // 不記住上次的 Commit & Push
  "git.enableSmartCommit": false,           // 暫存區為空時不自動 stage 全部
  "git.showActionButton": {                 // 主按鈕只負責 commit，不會變身成 Sync / Publish
    "commit": true,
    "publish": false,
    "sync": false
  },
  "git.confirmSync": true,                  // Sync 前跳確認，避免無聲 pull
  "git.autofetch": true
}
```

---

### 6-14 開啟擴充套件建議清單

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `code .vscode/extensions.json` | 在編輯器中開啟（不存在時建立）`extensions.json` |

**複製貼上**：

```bash
code .vscode/extensions.json
```

---

### 6-15 `.vscode/extensions.json` — 建議的擴充套件

**類型**：設定檔（貼到編輯器，不是貼到終端機）

**用途**：列出本課程建議的擴充套件。VS Code **只在開啟工作區時**檢查這份清單，存檔不會跳出提示。

| 擴充套件 | 用途 |
|---|---|
| `GitHub.vscode-pull-request-github` | 在 VS Code 內建立與審查 PR |
| `eamodio.gitlens` | 顯示每一行程式碼的修改紀錄 |
| `ms-python.python` | Python 語法提示與除錯 |

**貼到編輯器並存檔**：

```jsonc
// .vscode/extensions.json —— 建議的擴充套件：「開啟」codespace 時會跳出安裝提示（存檔不會觸發）
{
  "recommendations": [
    "GitHub.vscode-pull-request-github",
    "eamodio.gitlens",
    "ms-python.python"
  ]
}
```

**在目前的 codespace 立即安裝**：`F1` → **Extensions: Show Recommended Extensions** → 在 **Workspace Recommendations** 標題右側點雲朵圖示 **Install Workspace Recommended Extensions**。

---

### 6-16 提交工作區設定

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add <檔案...>` | 把兩個設定檔放進暫存區 |
| `git commit -m "..."` | 建立一個 commit，並寫上說明訊息 |
| `git push` | 把 commit 推送到 GitHub |

**複製貼上**：

```bash
git add .vscode/settings.json .vscode/extensions.json
git commit -m "chore: pin git behavior and recommend extensions"
git push
```

---

### 6-17 （選用）移除舊版的 `.devcontainer/`

**類型**：CLI Command

**適用對象**：只有先前依 v5 建立過 `.devcontainer/` 的學員才需要執行。

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git rm -r .devcontainer` | 刪除 `.devcontainer` 資料夾，並把這個刪除動作放進暫存區 |
| `git commit -m "..."` | 提交刪除 |
| `git push` | 推送到 GitHub |

執行後 `F1` → **Codespaces: Rebuild Container**；或直接刪除這台 codespace、重新開一台，通常更快。

**複製貼上**：

```bash
git rm -r .devcontainer
git commit -m "chore: drop custom dev container in favor of workspace settings"
git push
```

---

## D. Git 操作：用終端機完成版本控制

### 6-18 做法 A：Git 基本循環

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `echo "## Hello" >> README.md` | 在 README.md 結尾加一行，製造一個變更 |
| `git status` | 隨時確認現在的狀態：哪些檔案改了、哪些已暫存 |
| `git add README.md` | 工作目錄 → **暫存區** |
| `git commit -m "docs: add greeting"` | 暫存區 → **本地儲存庫** |
| `git push` | 本地儲存庫 → **遠端（GitHub）** |

**複製貼上**：

```bash
echo "## Hello" >> README.md
git status
git add README.md
git commit -m "docs: add greeting"
git push
```

---

### 6-19 做法 A：建立分支並推送

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch -c feature/hello` | 建立名為 `feature/hello` 的分支並切換過去 |
| `echo "print('hi')" > hello.py` | 建立一個只有一行的 Python 檔 |
| `git add hello.py && git commit -m "..."` | 暫存並提交 |
| `git push -u origin feature/hello` | 推送分支；`-u` 讓 Git 記住對應的遠端分支，之後只要打 `git push` |

**複製貼上**：

```bash
git switch -c feature/hello
echo "print('hi')" > hello.py
git add hello.py && git commit -m "feat: add hello.py"
git push -u origin feature/hello
```

---

### 6-20 用 GitHub CLI 開 PR

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `gh pr create --fill` | 建立 PR，用 commit 訊息自動填入標題與內容 |
| `gh pr view --web` | 在瀏覽器開啟這個 PR |
| `gh pr checks` | 查看 PR 上的 CI 檢查是否跑完 |

**複製貼上**：

```bash
gh pr create --fill
```

```bash
gh pr view --web
```

```bash
gh pr checks
```

---

## E. 衝突（conflict）怎麼處理

### 6-21 拉取並找出衝突檔案

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git pull` | 拉取遠端變更；若與本地修改同一處，會出現 `CONFLICT` 訊息 |
| `git status` | 列出哪些檔案有衝突（標示為 `both modified`） |

**複製貼上**：

```bash
git pull
git status
```

**接著手動處理**：用編輯器開啟衝突的檔案，刪掉 `<<<<<<<`、`=======`、`>>>>>>>` 三種標記，只保留想要的內容，存檔。

---

### 6-22 完成合併

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add <file>` | 告訴 Git「這個檔案的衝突已解決」 |
| `git commit --no-edit` | 用預設的合併訊息完成合併，不會跳出編輯器 |

⚠️ **貼上前請替換**：`<file>` 換成衝突的檔名，例如 `README.md`。

**複製貼上**：

```bash
git add <file>
git commit --no-edit
```

---

## F. 救命指令

以下每個區塊對應一種情境，**遇到時才使用**。

### 6-23 不知道現在是什麼狀態

**類型**：CLI Command

**指令意義**：`git status` 顯示目前分支、哪些檔案已修改、哪些已暫存。

```bash
git status
```

### 6-24 commit 訊息打錯（尚未 push）

**類型**：CLI Command

**指令意義**：`git commit --amend -m` 以新訊息取代最後一個 commit 的訊息。**已經 push 的 commit 不要這樣做。**

⚠️ **貼上前請替換**：`新訊息` 換成正確的 commit 訊息。

```bash
git commit --amend -m "新訊息"
```

### 6-25 想退回上一個 commit 但保留檔案

**類型**：CLI Command

**指令意義**：`git reset --soft HEAD~1` 撤銷最後一個 commit，修改內容仍留在暫存區。

```bash
git reset --soft HEAD~1
```

### 6-26 檔案改爛了想還原

**類型**：CLI Command

**指令意義**：`git restore <file>` 把檔案還原成最後一次 commit 的內容。⚠️ **未 commit 的變更會消失，無法復原。**

⚠️ **貼上前請替換**：`<file>` 換成要還原的檔名。

```bash
git restore <file>
```

### 6-27 查看歷史

**類型**：CLI Command

**指令意義**：`git log --oneline --graph --all` 以一行一個 commit、加上分支圖形的方式，顯示所有分支的歷史。按 `q` 離開。

```bash
git log --oneline --graph --all
```

### 6-28 誤刪 commit 想找回

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git reflog` | 列出 HEAD 曾經指向過的所有位置，包含已被「刪掉」的 commit |
| `git switch -c rescue <hash>` | 從找到的 commit 建立一條名為 `rescue` 的新分支，把它救回來 |

**步驟 1**：找出要救回的 commit hash（按 `q` 離開）。

```bash
git reflog
```

**步驟 2**：⚠️ **貼上前請替換**：`<hash>` 換成步驟 1 找到的 hash。

```bash
git switch -c rescue <hash>
```

---

✅ **本章驗收**：用 CLI 與 GUI 各做一次「改檔 → commit → push」，並開出一個 PR 後合併；main 分支上看得到 `hello.py`。
