# 第 9 章　以 Codespace 空容器執行映像 — 實作指令集

> 本檔收錄第 9 章實作時要複製貼上的指令與程式碼；觀念說明請參考課堂簡報。

---

## 使用前必讀

### 本檔出現的內容類型

| 類型 | 是什麼 | 怎麼用 |
|---|---|---|
| **CLI Command** | 直接在終端機輸入、立即執行的單行或多行指令 | 複製整個區塊，貼到 VS Code 下方的 **Terminal**，按 Enter |
| **Bash Script** | 多行指令組成的腳本，存成 `.sh` 檔後用 `bash` 執行 | 本章的區塊會用 here-document 自動把腳本寫成檔案，整段貼到終端機即可 |

> 本章沒有新的 Python Script；部署的是第 8 章推上 GHCR 的映像，部署機上**完全沒有原始碼**。

### 貼上規則

1. **一次只貼一個區塊**，看完輸出再貼下一個。
2. 標示 ⚠️ **貼上前請替換** 的區塊，要先把 `<...>` 換成你自己的內容；`<` `>` 符號本身也要刪掉。
3. 標示 🔒 **單獨執行** 的區塊需要你輸入內容，**不可與其他指令一起貼上**。
4. 可複製的 CLI 區塊內刻意不放註解；每個指令的意義都寫在區塊上方的說明表格。
5. **注意每個區塊標示在哪一台機器執行**：

| 代號 | 是哪一台 | 用途 | 有沒有原始碼 |
|---|---|---|---|
| **【開發機】** | 你一直在用的 repo codespace | 查詢要部署的版本號、製造新版本 | 有 |
| **【部署機】** | 本章新開的**空白** codespace | 拉映像、注入組態、執行 | **沒有** |

【開發機】的指令請在 repo 根目錄執行（`pwd` 應為 `/workspaces/<repo>`；若不是，先執行 `cd /workspaces/<repo>`）。【部署機】的指令一律使用絕對路徑，在哪個資料夾執行都可以。

### 本章流程

```
【開發機】取得 SHA ──────────────────────────────┐
                                                 ▼
【部署機】安裝工具（一次）→ 設定（一次）→ bash deploy.sh <SHA> → ✅
                                                 ▲
                         之後每次部署或回溯，都只需要這一行
```

---

## A. 【開發機】取得要部署的版本號

### 9-1 【開發機】取得 main 最新的 commit SHA

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 切到 main 並同步遠端最新內容 |
| `git rev-parse HEAD` | 顯示目前 commit 完整的 **40 碼 SHA**，也就是映像的 tag → **複製下來** |
| `gh run list --branch main --limit 1` | 查看 main 最近一次 CI 的狀態；必須是 `completed` / `success`，否則映像還沒推上去 |

**複製貼上**：

```bash
git switch main && git pull
git rev-parse HEAD
gh run list --branch main --limit 1
```

同時到 GitHub 個人頁面的 **Packages** 分頁點開這個 package，確認：
- tag 清單中有這個 SHA
- 頁面上方 `docker pull ghcr.io/...` 那一行——9-6 設定時直接整行貼上即可

---

## B. 【瀏覽器】建立部署機

此節為瀏覽器操作，沒有指令：

1. 新分頁開啟 `https://github.com/codespaces`
2. 找到範本 **Blank**（沒看到請點 **See all** 或 **Explore templates**）
3. 按 **Use this template**，等待 1～2 分鐘出現 VS Code 介面

⚠️ **不要**在 repo 頁面再按一次 Create codespace，那樣部署機上就有原始碼，實驗前提不成立。

---

## C. 【部署機】確認這是一台乾淨的機器

### 9-2 【部署機】檢查機器狀態

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 | 預期結果 |
|---|---|---|
| `pwd` | 顯示目前位置 | `/workspaces/` 下的空白範本資料夾，而不是你的 repo |
| `ls -A` | 列出所有檔案（含隱藏檔） | 沒有 `Dockerfile`、沒有 `app/` |
| `docker version` | 確認 Docker 可用 | Client 與 Server 都要出現 |
| `docker images` | 列出本機的映像 | 空的：這台機器沒有任何你建置過的映像 |
| `echo "${LLM_API_KEY:-（空）}"` | 顯示金鑰變數；沒有值時顯示「（空）」 | 「（空）」：Secret 綁定 repo，這台沒有 repo 所以拿不到 |

最後一行是關鍵觀察：**部署機不會自動擁有開發機的機密，組態必須由部署者提供。**

**複製貼上**：

```bash
pwd
ls -A
docker version
docker images
echo "${LLM_API_KEY:-（空）}"
```

建議把終端機分頁改名以免混淆：分頁標題按右鍵 → **Rename** → 輸入 `DEPLOY`。

---

## D. 【部署機】安裝部署工具（只需一次）

以下三個區塊都可以**整段一次貼上**（裡面沒有需要輸入的指令），會在 `~/deploy/` 建立兩支腳本：

| 腳本 | 做什麼 | 對應 PaaS 的哪個畫面 |
|---|---|---|
| `make-env.sh` | 設定要部署哪個映像、金鑰與模型，寫入 `deploy.conf` 與 `prod.env` | 服務設定頁、環境變數 / Secrets 設定 |
| `deploy.sh` | 拉取指定版本 → 重建容器 → 健康檢查 → 驗證組態與版本 | 「Deploy」按鈕與部署紀錄 |

### 9-3 【部署機】建立 `make-env.sh`（部署設定腳本）

**類型**：Bash Script

**腳本意義**：

這支腳本以**互動問答**方式收集三項設定，並寫成兩個檔案：`deploy.conf`（要部署哪個映像）與 `prod.env`（容器執行時的環境變數）。它會自動清理常見的輸入錯誤，重跑時直接按 Enter 可保留原本的值。

| 區段 | 意義 |
|---|---|
| `cat > ~/deploy/make-env.sh <<'SCRIPT'` … `SCRIPT` | **here-document**：把兩個 `SCRIPT` 之間的內容原封不動寫入檔案，貼上時不會執行 |
| `#!/usr/bin/env bash` | 第一行（shebang）：指定用 bash 執行這支腳本 |
| `set -euo pipefail` | 任何一步出錯就立刻停止、使用未定義變數視為錯誤，避免帶著錯誤繼續執行 |
| `DIR=...`、`CONF=...`、`ENV_FILE=...` | 以腳本所在資料夾為基準，算出設定檔的**絕對路徑** |
| `clean()` | 清理輸入：去除 Windows 換行、前後空白、誤加的單雙引號 |
| `old_image` / `old_model` / `old_key` | 讀取既有設定作為預設值，重跑時按 Enter 即保留 |
| **1. image 路徑** | `read -rp` 讀取輸入；自動去掉 `docker pull ` 前綴、轉小寫、移除誤帶的 tag，最後以正規表示式驗證格式為 `ghcr.io/<帳號>/<repo>` |
| **2. API key** | `read -rsp` 讀取時**不顯示字元**；空值直接中止；格式不像 Google 金鑰（`AIza` 開頭、長 39 碼）時提出警告 |
| **3. model ID** | 讀取模型名稱，自動去掉 `models/` 前綴；空值直接中止 |
| `umask 077` + `mktemp` + `mv -f` | **原子性寫入**：先寫暫存檔再改名，權限只有自己可讀寫，不會留下寫到一半的檔案 |
| 最後的 `echo` | 顯示設定摘要；金鑰只顯示前 4 碼與長度 |

**複製貼上**（整段一次貼上）：

```bash
mkdir -p ~/deploy
cat > ~/deploy/make-env.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
DIR="$(cd "$(dirname "$0")" && pwd)"
CONF="$DIR/deploy.conf"
ENV_FILE="$DIR/prod.env"

clean() {
  local v="${1//$'\r'/}"
  v="${v#"${v%%[![:space:]]*}"}"
  v="${v%"${v##*[![:space:]]}"}"
  v="${v#\"}"; v="${v%\"}"; v="${v#\'}"; v="${v%\'}"
  printf '%s' "$v"
}

old_image=""; old_model=""; old_key=""
[[ -f "$CONF" ]]     && old_image="$(sed -n 's/^IMAGE=//p' "$CONF")"
[[ -f "$ENV_FILE" ]] && old_model="$(sed -n 's/^LLM_MODEL=//p' "$ENV_FILE")"
[[ -f "$ENV_FILE" ]] && old_key="$(sed -n 's/^LLM_API_KEY=//p' "$ENV_FILE")"

read -rp "image 路徑，例如 ghcr.io/帳號/repo${old_image:+ [Enter 保留：$old_image]}: " image
image="$(clean "${image:-$old_image}")"
image="${image#docker pull }"
image="$(printf '%s' "$image" | tr '[:upper:]' '[:lower:]')"
if [[ "$image" == ghcr.io/*/*:* ]]; then
  echo "ℹ️  已移除 image 路徑中的 tag「:${image##*:}」，部署時再指定版本"
  image="${image%:*}"
fi
if [[ ! "$image" =~ ^ghcr\.io/[a-z0-9._-]+/[a-z0-9._-]+$ ]]; then
  echo "❌ image 路徑格式錯誤：「$image」（應為 ghcr.io/<帳號>/<repo>）" >&2; exit 1
fi

read -rsp "Google AI Studio API key（輸入時不顯示）${old_key:+ [Enter 保留原金鑰]}: " key; echo
key="$(clean "${key:-$old_key}")"
[[ -n "$key" ]] || { echo "❌ 金鑰是空的，未寫入任何檔案" >&2; exit 1; }
if [[ "$key" != AIza* || ${#key} -ne 39 ]]; then
  echo "⚠️  金鑰格式與常見格式不同（開頭 ${key:0:4}、長度 ${#key}），請確認是否貼對" >&2
fi

read -rp "model ID${old_model:+ [Enter 保留：$old_model]}: " model
model="$(clean "${model:-$old_model}")"; model="${model#models/}"
[[ -n "$model" ]] || { echo "❌ model ID 是空的，未寫入任何檔案" >&2; exit 1; }

umask 077
tmp="$(mktemp "$DIR/.tmp.XXXXXX")"
printf 'IMAGE=%s\n' "$image" > "$tmp" && mv -f "$tmp" "$CONF"
tmp="$(mktemp "$DIR/.tmp.XXXXXX")"
printf 'LLM_PROVIDER=google\nLLM_MODEL=%s\nLLM_MERGE_SYSTEM=auto\nLLM_API_KEY=%s\n' "$model" "$key" > "$tmp" && mv -f "$tmp" "$ENV_FILE"

echo "✅ 設定完成"
echo "   image：$image"
echo "   model：$model"
echo "   金鑰 ：${key:0:4}****（長度 ${#key}）"
echo "   下一步：bash $DIR/deploy.sh <要部署的 SHA 或版本號>"
SCRIPT
```

---

### 9-4 【部署機】建立 `deploy.sh`（部署與驗證腳本）

**類型**：Bash Script

**腳本意義**：

這支腳本就是一個**最陽春的 PaaS**：拉映像 → 注入組態 → 啟動 → 健康檢查 → 驗證。用法是 `bash ~/deploy/deploy.sh <SHA 或版本號>`，任何一個階段失敗都會停下，並以 `❌` 說明原因。

| 區段 | 意義 |
|---|---|
| `set -euo pipefail` | 任何一步出錯就立刻停止 |
| `NAME="prod"`、`PORT=8000` | 容器名稱與對外的埠號 |
| `die()` | 印出錯誤訊息並結束腳本的小函式 |
| 參數檢查 | 必須剛好給一個版本號；必須已執行過 `make-env.sh`；映像路徑與版本號都要符合格式 |
| **[1/5] 拉取** | `docker pull` 下載指定版本；失敗時提示確認 tag 是否存在、package 是否已設為 Public |
| **[2/5] 移除** | `docker rm -f` 刪除舊容器；env 檔只在**建立容器時**讀取，必須重建才會套用新組態 |
| **[3/5] 啟動** | `docker run -d` 以絕對路徑注入 `prod.env` 並啟動 |
| **[4/5] 健康檢查** | 最多等 30 秒，輪詢 `/healthz` 直到服務就緒；失敗時印出容器最近 30 行 log |
| **[5/5] 驗證** | ① `docker inspect` 檢查容器**實際收到**的 `LLM_MODEL`、`LLM_API_KEY` 不是空值；② 呼叫 `/version`，若給的是 40 碼 SHA 則比對是否一致；③ 實際呼叫一次 `/api/ask`，若回到示範模式（mock）就顯示供應商的錯誤並判定失敗 |
| 最後的 `echo` | 顯示部署完成與瀏覽方式 |

**複製貼上**（整段一次貼上）：

```bash
mkdir -p ~/deploy
cat > ~/deploy/deploy.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
DIR="$(cd "$(dirname "$0")" && pwd)"
CONF="$DIR/deploy.conf"
ENV_FILE="$DIR/prod.env"
NAME="prod"
PORT=8000

die() { echo "❌ $*" >&2; exit 1; }

[[ $# -eq 1 ]] || die "用法：bash $DIR/deploy.sh <SHA 或版本號>"
TAG="$(printf '%s' "$1" | tr -d '[:space:]')"
[[ -f "$CONF" && -f "$ENV_FILE" ]] || die "找不到設定檔，請先執行：bash $DIR/make-env.sh"
IMAGE="$(sed -n 's/^IMAGE=//p' "$CONF")"
[[ "$IMAGE" =~ ^ghcr\.io/[a-z0-9._-]+/[a-z0-9._-]+$ ]] || die "deploy.conf 的 IMAGE 格式錯誤：「$IMAGE」，請重新執行 make-env.sh"
[[ "$TAG" =~ ^[A-Za-z0-9_][A-Za-z0-9._-]{0,127}$ ]] || die "版本號格式錯誤：「$TAG」"
REF="$IMAGE:$TAG"

echo "▶ [1/5] 拉取 image：$REF"
docker pull --quiet "$REF" >/dev/null || die "拉取失敗。確認此版本存在於 GHCR 的 tag 清單，且 package 已設為 Public"

echo "▶ [2/5] 移除舊容器（若有）"
docker rm -f "$NAME" >/dev/null 2>&1 || true

echo "▶ [3/5] 以 $ENV_FILE 注入組態並啟動"
docker run -d --name "$NAME" -p "$PORT:8000" --env-file "$ENV_FILE" "$REF" >/dev/null

echo "▶ [4/5] 等待健康檢查"
for _ in $(seq 1 30); do
  curl -fsS "localhost:$PORT/healthz" >/dev/null 2>&1 && break
  sleep 1
done
if ! curl -fsS "localhost:$PORT/healthz" >/dev/null 2>&1; then
  echo "--- 容器最近的 log ---" >&2; docker logs --tail 30 "$NAME" >&2 || true
  die "服務未能啟動"
fi

echo "▶ [5/5] 驗證版本與組態"
empty="$(docker inspect "$NAME" --format '{{range .Config.Env}}{{println .}}{{end}}' \
         | awk -F= '/^LLM_(MODEL|API_KEY)=/ { if ($2 == "") print $1 }')"
[[ -z "$empty" ]] || die "容器收到的下列變數為空值：$empty。請重新執行 make-env.sh 後再部署"

version="$(curl -fsS "localhost:$PORT/version")"
echo "   /version：$version"
if [[ "$TAG" =~ ^[0-9a-f]{40}$ ]] && ! grep -q "\"$TAG\"" <<<"$version"; then
  die "執行中的版本與指定的 SHA 不一致"
fi

reply="$(curl -fsS -X POST "localhost:$PORT/api/ask" -H 'Content-Type: application/json' -d '{"prompt":"ping"}')"
if grep -q '"is_mock":true' <<<"$reply"; then
  reason="$(sed -nE 's/.*"reason":"([^"]*)".*/\1/p' <<<"$reply")"
  echo "--- 供應商回傳的錯誤 ---" >&2
  docker logs "$NAME" 2>&1 | grep -E '呼叫失敗|連線失敗' | tail -3 >&2 || true
  die "服務以示範模式運作（原因：$reason）"
fi

echo "✅ 部署完成：$REF（live）"
echo "   瀏覽器檢視：下方 Ports 分頁 → $PORT → 地球圖示"
SCRIPT
```

---

### 9-5 【部署機】設定腳本權限並確認

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `chmod 700 ~/deploy/make-env.sh ~/deploy/deploy.sh` | 設為只有自己能讀、寫、執行 |
| `ls -l ~/deploy` | 列出資料夾內容，應看到兩支腳本，權限為 `-rwx------` |

**複製貼上**：

```bash
chmod 700 ~/deploy/make-env.sh ~/deploy/deploy.sh
ls -l ~/deploy
```

---

## E. 【部署機】設定（只需一次）

### 9-6 【部署機】執行部署設定

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `bash ~/deploy/make-env.sh` | 執行 9-3 建立的設定腳本，依提示輸入三項設定 |

🔒 **單獨執行**：這一行需要互動輸入，**必須單獨貼上**。若和其他指令一起貼上，貼上的下一行文字會被當成你的輸入。

**複製貼上**：

```bash
bash ~/deploy/make-env.sh
```

依提示輸入：

| 提示 | 輸入什麼 | 說明 |
|---|---|---|
| `image 路徑` | 9-1 時在 Packages 頁面看到的 `docker pull ghcr.io/...` **整行貼上** | 腳本會自動去掉 `docker pull`、tag，並轉成小寫 |
| `API key` | Google AI Studio 的金鑰：到 Google AI Studio 的 **API Keys** 頁面複製 | 部署機沒有 Codespaces Secret，必須手動貼上；輸入時畫面不顯示任何字元，這是正常的 |
| `model ID` | 第 4 章取得的 model ID | 不用加引號；多餘的引號與 `models/` 前綴會被自動清除 |

看到 `✅ 設定完成` 並列出三項摘要即可。之後要換金鑰或模型，重新執行這一行；**不想改的項目直接按 Enter** 保留原值。不要用編輯器手動修改 `prod.env`。

---

## F. 【部署機】部署

### 9-7 【部署機】部署指定版本

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `bash ~/deploy/deploy.sh <SHA>` | 執行部署腳本，部署指定的版本，並自動完成五階段驗證 |

⚠️ **貼上前請替換**：`<SHA>` 換成 9-1 複製的 40 碼 SHA（也可以用版本號，例如 `v0.1.0`）。

**複製貼上**：

```bash
bash ~/deploy/deploy.sh <SHA>
```

正常的輸出：

```text
▶ [1/5] 拉取 image：ghcr.io/<帳號>/<repo>:7ae9e9b1...
▶ [2/5] 移除舊容器（若有）
▶ [3/5] 以 /home/codespace/deploy/prod.env 注入組態並啟動
▶ [4/5] 等待健康檢查
▶ [5/5] 驗證版本與組態
   /version：{"git_sha":"7ae9e9b1...","provider":"google","mode":"live"}
✅ 部署完成：ghcr.io/<帳號>/<repo>:7ae9e9b1...（live）
```

想用瀏覽器看畫面：下方 **Ports** 分頁 → 8000 → 點地球圖示。

📝 **請把這次部署的 SHA 記下來**（例如貼在記事本），9-14 回溯時要用。

---

## G. 部署失敗時

### 9-8 【部署機】檢查容器實際收到的組態

**類型**：CLI Command

**指令意義**：

| 指令片段 | 意義 |
|---|---|
| `docker inspect prod --format '{{range .Config.Env}}{{println .}}{{end}}'` | 列出 `prod` 容器**實際收到**的所有環境變數，一行一個 |
| `grep '^LLM'` | 只保留 `LLM` 開頭的變數 |
| `sed -E 's/^(LLM_API_KEY=.{0,4}).*/\1****/'` | 把金鑰遮蔽成只顯示前 4 碼 |

> 原則：**檢查「容器實際收到什麼」，而不是「檔案裡寫了什麼」。**

**複製貼上**：

```bash
docker inspect prod --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep '^LLM' | sed -E 's/^(LLM_API_KEY=.{0,4}).*/\1****/'
```

**錯誤訊息對照**：

| 錯誤訊息 | 原因 | 處理 |
|---|---|---|
| `找不到設定檔` | 尚未執行設定 | 執行 9-6 |
| `版本號格式錯誤` | SHA 複製不完整，或夾帶了其他字元 | 從 9-1 重新複製 |
| `拉取失敗` | 此版本不在 GHCR，或 package 仍是 private | 確認 Packages 頁面的 tag 清單；確認第 8 章已將 package 設為 Public（8-14） |
| `服務未能啟動` | 容器啟動即退出（log 會一併顯示） | 多半是 `ConfigError`：依 log 訊息重新執行 9-6 |
| `容器收到的下列變數為空值` | 組態檔的值是空的 | 重新執行 9-6 |
| `執行中的版本與指定的 SHA 不一致` | 該 tag 對應的映像內容與預期不同 | 確認 CI 是否以正確的 SHA 建置 |
| `服務以示範模式運作（原因：google 回應 HTTP 400）` | 供應商拒絕請求 | `API key not valid` → 金鑰錯誤；提到 model → model ID 錯誤；修正後重新執行 9-6 與 9-7 |

---

## H. 保留部署機，供版本回溯使用

**先不要刪除部署機**。若中途關掉分頁：`https://github.com/codespaces` → 找到標示為 unpublished 的那一台 → 右側「**…**」→ **Open in browser**。重新打開後不需要重設任何變數，所有設定都存在 `~/deploy/` 的檔案裡。

---

## I. 版本回溯（rollback）

### 9-9 【開發機】建立新版本的工作分支

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 切到最新的 main |
| `git switch -c feat/new-prompt` | 建立並切換到 `feat/new-prompt` 分支 |
| `code app/main.py` | 在編輯器開啟 `main.py` |

**複製貼上**：

```bash
git switch main && git pull
git switch -c feat/new-prompt
code app/main.py
```

⚠️ **若出現 `fatal: a branch named 'feat/new-prompt' already exists`**：代表這條分支之前已經建立過，這次沒有切換成功，你仍停在 main。請貼下面這行切過去，確認 `git branch --show-current` 顯示 `feat/new-prompt` 後再進行下面的修改（不要在 main 上繼續）：

```bash
git switch feat/new-prompt
```

**接著手動修改**：把 `SYSTEM_PROMPT` 的預設值改成看得出差異的內容，例如加上「回答時以『您好』開頭」，存檔。

---

### 9-10 【開發機】提交並開 PR

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add app/main.py && git commit -m "..."` | 暫存並提交修改 |
| `git push -u origin feat/new-prompt` | 推送分支並建立 upstream |
| `gh pr create --fill` | 建立 PR，觸發 PR 階段的 CI |

**複製貼上**：

```bash
git add app/main.py && git commit -m "feat: refine default system prompt"
git push -u origin feat/new-prompt
gh pr create --fill
```

---

### 9-11 【開發機】等待 CI 並合併

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `gh pr checks --watch` | 持續顯示 PR 的 CI 進度，等到全綠才結束 |
| `gh pr merge --merge --delete-branch` | 合併 PR → 觸發 main 的 CI → 推出新映像 |

⚠️ `gh pr checks --watch` 會持續更新畫面，全部顯示 `pass` 後才會繼續執行合併。若出現 `fail`，請按 `Ctrl+C` 中止，先排除錯誤。

⚠️ 若畫面顯示 `no checks reported`，代表 GitHub 還沒開始執行 CI，**不會合併**。等 10～20 秒再貼一次即可。

**複製貼上**：

```bash
gh pr checks --watch && gh pr merge --merge --delete-branch
```

---

### 9-12 【開發機】取得新版 SHA

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 切回 main 並同步合併結果 |
| `git rev-parse HEAD` | 顯示新版的 SHA → **複製下來** |
| `gh run list --branch main --limit 1` | 等到 `completed` / `success` 才繼續，否則新映像還沒推上去 |

⚠️ 剛合併完時，新的 CI 可能還沒排入佇列，列出的會是**上一次**已完成的紀錄。請確認這一筆的標題含有 `feat/new-prompt`（也就是剛才的合併），而且狀態是 `completed` / `success`；不是的話，等 10～20 秒再執行一次 `gh run list --branch main --limit 1`。

**複製貼上**：

```bash
git switch main && git pull
git rev-parse HEAD
gh run list --branch main --limit 1
```

---

### 9-13 【部署機】部署新版

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `bash ~/deploy/deploy.sh <新版 SHA>` | 以同一支腳本部署新版本 |

⚠️ **貼上前請替換**：`<新版 SHA>` 換成 9-12 複製的 SHA。

**複製貼上**：

```bash
bash ~/deploy/deploy.sh <新版 SHA>
```

用瀏覽器問一個問題，回答應反映新的提示詞。

---

### 9-14 【部署機】假設新版出了問題：回溯

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `bash ~/deploy/deploy.sh <舊版 SHA>` | 用**舊版本號**再部署一次，就是回溯 |

⚠️ **貼上前請替換**：`<舊版 SHA>` 換成 9-7 部署過的舊版 SHA。

**複製貼上**：

```bash
bash ~/deploy/deploy.sh <舊版 SHA>
```

`/version` 顯示舊版的 SHA，舊版被原封不動地叫回來。回溯時**沒有**重新建置、沒有 `git revert`、也沒有等待 CI——只是用另一個版本號再部署一次。

> - 回溯的候選版本應從 **GHCR 的 tag 清單**挑選，而不是從 `git log` 隨便挑一個 SHA（只有推進 main 的 commit 才有映像）。
> - 回溯只是讓服務先恢復正常的緊急措施；之後仍要在開發機修正程式碼（或 `git revert`），走完整流程發出新版。

---

### 9-15 收工：刪除部署機

此步驟為瀏覽器操作，沒有指令：

1. 開啟 `https://github.com/codespaces`
2. 找到那台 unpublished 的部署機 → 右側「**…**」→ **Delete** → 確認

刪除時會提示「unpublished 的變更將會遺失」。部署機上只有部署工具與組態，而組態（含金鑰）本來就不該被保存，直接確認即可。

---

✅ **本章驗收**：

1. CI 的 PR 階段全綠且**沒有**推映像；合併後 GHCR 出現 `<SHA>` 與 `latest`
2. 在**部署機**（Blank 範本建立、無原始碼、無 Secret）執行 `deploy.sh <SHA>`，輸出 `✅ 部署完成（live）`
3. 完成一次「新版 → 回溯舊版」，`/version` 分別顯示兩個 SHA；完成後已刪除部署機
4. 打 `v0.1.0` tag 後，GHCR 出現同名 tag
