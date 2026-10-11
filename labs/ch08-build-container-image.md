# 第 8 章　程式碼打包成容器映像 — 實作指令集

> 本檔收錄第 8 章實作時要複製貼上的指令與程式碼；觀念說明請參考課堂簡報。

---

## 使用前必讀

### 本檔出現的內容類型

| 類型 | 是什麼 | 怎麼用 |
|---|---|---|
| **CLI Command** | 直接在終端機輸入、立即執行的單行或多行指令 | 複製整個區塊，貼到 VS Code 下方的 **Terminal**，按 Enter |
| **Bash Script** | 多行指令組成的腳本。本章的 Bash Script 嵌在 CI workflow（`ci.yml`）的 `run:` 欄位中，由 GitHub Actions 執行 | 隨 `ci.yml` 一起貼到編輯器，不需要在終端機執行 |
| **設定檔** | `Dockerfile`、`ci.yml` 等宣告式內容（非指令） | 用編輯器開啟檔案，整段貼上並存檔 |

> 本章沒有新的 Python Script；打包的是第 7 章寫好的 `app/llm.py` 與 `app/main.py`。

### 貼上規則

1. **一次只貼一個區塊**，看完輸出再貼下一個。
2. 標示 ⚠️ **貼上前請替換** 的區塊，要先把 `<...>` 換成你自己的內容；`<` `>` 符號本身也要刪掉。
3. 可複製的 CLI 區塊內刻意不放註解；每個指令的意義都寫在區塊上方的說明表格。
4. 本章所有指令都在**你的 repo codespace（開發機）** 執行，終端機請位於 repo 根目錄。
5. 開始前請確認第 7 章的 `ai-dev` 容器已用 `Ctrl+C` 停止，否則 8000 埠會被佔用。

---

## A. 撰寫 Dockerfile

### 8-1 開啟 `Dockerfile`

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `code Dockerfile` | 在編輯器中開啟（不存在時建立）repo 根目錄的 `Dockerfile` |

**複製貼上**：

```bash
code Dockerfile
```

---

### 8-2 `Dockerfile` — 多階段建置、非 root 執行

**類型**：設定檔（貼到編輯器並存檔，不是貼到終端機）

**用途**：描述「如何把程式碼、套件與執行環境封裝成一個映像」。第 7 章是把原始碼**掛進**容器；這份 Dockerfile 把一切**封進**映像，讓它成為可以交給任何機器的成品。

**內容簡介**：

| 區段 / 指令 | 作用 |
|---|---|
| `FROM python:3.12-slim AS builder` | **第一階段（builder）**：只用來安裝套件 |
| `RUN pip install --prefix=/install/deps ...` | 把套件安裝到獨立資料夾，方便下一階段整包複製 |
| `FROM python:3.12-slim` | **第二階段（runtime）**：重新從乾淨的映像開始，不含建置工具，體積更小、攻擊面更小 |
| `ARG GIT_SHA=dev` / `ENV GIT_SHA=$GIT_SHA` | 建置時接收版本號並寫入環境變數，`/version` 端點會回報它 |
| `ENV PATH=... PYTHONPATH=...` | 讓 Python 找得到從 builder 複製過來的套件 |
| `COPY --from=builder /install/deps /deps` | 只從 builder 取回安裝好的套件 |
| `COPY app/ ./app/` | 把程式碼封進映像 |
| `RUN useradd ...` / `USER appuser` | 建立並切換到一般使用者，**不以 root 執行** |
| `EXPOSE 8000` | 宣告服務使用 8000 埠 |
| `HEALTHCHECK ...` | 每 30 秒呼叫一次 `/healthz`，讓 Docker 知道服務是否健康 |
| `CMD ["uvicorn", ...]` | 容器啟動時執行的指令 |

**貼到編輯器並存檔**：

```dockerfile
# ---- builder：只用來安裝套件 ----
FROM python:3.12-slim AS builder
WORKDIR /install
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install/deps -r requirements.txt

# ---- runtime：不含建置工具，體積更小、攻擊面更小 ----
FROM python:3.12-slim
ARG GIT_SHA=dev
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/deps/bin:$PATH" \
    PYTHONPATH="/deps/lib/python3.12/site-packages" \
    GIT_SHA=$GIT_SHA
COPY --from=builder /install/deps /deps
WORKDIR /srv
COPY app/ ./app/
RUN useradd --create-home --uid 10001 appuser
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/healthz', timeout=3)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 8-3 提交 Dockerfile（第五個 commit）

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add Dockerfile && git commit -m "..."` | 暫存並提交 Dockerfile |

**複製貼上**：

```bash
git add Dockerfile && git commit -m "build: containerize app with multi-stage build"
```

---

## B. 建置並執行映像

### 8-4 建置映像並啟動容器

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `REPO="$(git rev-parse --show-toplevel)"` | 取得 repo 根目錄的絕對路徑（開新終端機時需重設） |
| `docker build -t ai-demo:dev "$REPO"` | 依 Dockerfile 建置映像，命名為 `ai-demo`、標記（tag）為 `dev` |
| `docker run -d --name ai-demo -p 8000:8000 --env-file "$REPO/dev.env" ai-demo:dev` | 以背景模式（`-d`）啟動容器，注入 `dev.env` 組態。**不再需要 `-v` 掛載原始碼，也不必 `pip install`** |
| `docker logs ai-demo` | 查看容器的 log，啟動訊息應顯示 `mode=live` |

**複製貼上**：

```bash
REPO="$(git rev-parse --show-toplevel)"
docker build -t ai-demo:dev "$REPO"
docker run -d --name ai-demo -p 8000:8000 --env-file "$REPO/dev.env" ai-demo:dev
docker logs ai-demo
```

---

### 8-5 驗證容器

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 | 預期結果 |
|---|---|---|
| `curl -s localhost:8000/healthz` | 健康檢查 | `{"status":"ok"}` |
| `curl -s localhost:8000/version` | 版本資訊 | `git_sha` 為 `dev` |

`git_sha=dev` 代表這個映像是在開發機上、用工作目錄建置的，可能包含尚未 commit 的修改，所以**沒有版本身分**。要成為可追溯的成品，必須交給 CI/CD 在乾淨環境中建置。

**複製貼上**：

```bash
curl -s localhost:8000/healthz
curl -s localhost:8000/version
```

---

### 8-6 改完程式後重建並重跑

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `docker rm -f ai-demo` | 強制停止並刪除舊容器 |
| `docker build -t ai-demo:dev "$REPO"` | 以最新的程式碼重新建置映像 |
| `docker run -d ...` | 以新映像重新啟動容器 |
| `&&` 與行尾 `\` | 前一步成功才執行下一步；`\` 讓三個步驟串成同一行指令 |

**複製貼上**：

```bash
REPO="$(git rev-parse --show-toplevel)"
docker rm -f ai-demo && docker build -t ai-demo:dev "$REPO" && \
docker run -d --name ai-demo -p 8000:8000 --env-file "$REPO/dev.env" ai-demo:dev
```

---

## C. 把開發成果推上 GitHub

### 8-7 確認 commit 並推送分支、開 PR

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git log --oneline main..HEAD` | 只列出這條分支新增的 commit，應為 **5 個**（chore / feat / feat / build / build） |
| `git status` | 應為 `working tree clean`；`dev.env` 被忽略，不算變更 |
| `git push -u origin feat/ai-demo` | 推送分支；`-u` 建立 upstream，之後狀態列才會顯示 `↑N / ↓N` |
| `gh pr create --fill` | 建立 PR，用 commit 訊息自動填入標題與內容（也可在 GitHub 網頁按 **Compare & pull request**） |

**複製貼上**：

```bash
git log --oneline main..HEAD
git status
git push -u origin feat/ai-demo
gh pr create --fill
```

✅ **驗收**：

1. `git log --oneline main..HEAD` 看得到五個語意清楚的 commit（chore / feat / feat / build / build）
2. `git status` 顯示 working tree clean，且 `dev.env` 從未出現在任何 commit 中
3. PR 頁面看得到完整的檔案異動
4. 用手機行動網路（未登入 GitHub）能打開 Public 網址並得到回覆

---

## D. 先把開發分支的 PR 合併

### 8-8 合併 PR 並更新本地 main

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `gh pr merge --merge --delete-branch` | 以 merge commit 方式合併 PR，並刪除遠端的功能分支（也可在網頁按 **Merge pull request**） |
| `git switch main && git pull` | 切回 main，並拉取遠端最新內容，讓本地 main 跟上 |

> 只有推進 main 的 commit 才會有對應的映像——功能分支上的 SHA 不會。

**複製貼上**：

```bash
gh pr merge --merge --delete-branch
git switch main && git pull
```

---

## E. 撰寫 CI/CD：`.github/workflows/ci.yml`

### 8-9 開新分支並建立 workflow 檔案

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch -c ci/pipeline` | 從最新的 main 建立 `ci/pipeline` 分支 |
| `mkdir -p .github/workflows` | 建立 GitHub Actions 規定的 workflow 資料夾 |
| `code .github/workflows/ci.yml` | 在編輯器中開啟（不存在時建立）`ci.yml` |

**複製貼上**：

```bash
git switch -c ci/pipeline
mkdir -p .github/workflows
code .github/workflows/ci.yml
```

---

### 8-10 `.github/workflows/ci.yml` — 建置、測試、推送映像

**類型**：設定檔（YAML），內含兩段 Bash Script（貼到編輯器並存檔，不是貼到終端機）

**用途**：定義 GitHub Actions 的 CI/CD 流水線。每次 PR 或推送時，GitHub 會開一台乾淨的機器，從零 clone 程式碼、建置映像、測試，通過後才推送到 GHCR（GitHub Container Registry）。

**觸發條件與行為**：

| 事件 | 建置 | 測試 | 推送映像 | 推送的 tag |
|---|---|---|---|---|
| 開 PR / PR 更新 | ✅ | ✅ | ❌ | — |
| 推進 main（含 PR 合併） | ✅ | ✅ | ✅ | `<SHA>`、`latest` |
| 推送 `v*` 版本 tag | ✅ | ✅ | ✅ | `<SHA>`、版本號（如 `v0.1.0`） |

**步驟簡介**：

| 步驟 | 作用 |
|---|---|
| `actions/checkout@v5` | 從零 clone 程式碼：這就是「乾淨建置」 |
| `Build image` | 以 `--build-arg GIT_SHA=${{ github.sha }}` 把 commit SHA 寫進映像 |
| `Smoke test`（Bash Script） | 見下表 |
| `Publish to GHCR`（Bash Script） | 見下表；`if: github.event_name == 'push'` 是品質閘門，PR 階段不會執行 |

**`Smoke test` 這段 Bash Script 的意義**：

| 指令 | 意義 |
|---|---|
| `docker run -d --name smoke ... -e LLM_PROVIDER=google ai-demo:ci` | 不給金鑰啟動容器，App 會以 mock 模式運作（CI 不持有任何 LLM 金鑰） |
| `for i in $(seq 1 20); do ... done` | 最多重試 20 次、每次間隔 2 秒，等 `/healthz` 回應 |
| `if [ "$ok" != 1 ]; then docker logs smoke; exit 1; fi` | 服務始終沒起來就印出 log 並讓 CI 失敗 |
| `curl ... /version \| grep -q '"<SHA>"'` | 確認映像回報的版本就是這次建置的 SHA |
| `curl ... /api/ask \| grep -q '"is_mock":true'` | 確認服務能回應，且確實處於 mock 模式 |
| `docker rm -f smoke` | 清除測試容器 |

**`Publish to GHCR` 這段 Bash Script 的意義**：

| 指令 | 意義 |
|---|---|
| `echo "$REGISTRY_TOKEN" \| docker login ghcr.io ... --password-stdin` | 用 GitHub 自動發放的 `GITHUB_TOKEN` 登入 GHCR |
| `IMAGE=$(echo "ghcr.io/..." \| tr '[:upper:]' '[:lower:]')` | 組出映像路徑並轉成小寫（GHCR 路徑必須小寫） |
| `TAGS=...` 與 `if [ ... = "tag" ]` | 一律推 `<SHA>`；打 tag 時加推版本號，推 main 時加推 `latest` |
| `for t in $TAGS; do docker tag ...; docker push ...; done` | 逐一標記並推送每個 tag |

**貼到編輯器並存檔**：

```yaml
name: ci

on:
  pull_request:                 # PR：只建置與測試，不產生交付物
  push:
    branches: [main]            # main：建置、測試、推送 <SHA> 與 latest
    tags: ['v*']                # 發版 tag：建置、測試、推送 <SHA> 與版本號

jobs:
  build-test-publish:
    runs-on: ubuntu-latest      # 標準 runner：public repo 免費
    timeout-minutes: 15
    permissions:
      contents: read
      packages: write           # 推送 GHCR 所需；來自 fork 的 PR 會被 GitHub 自動降為唯讀
    steps:
      - uses: actions/checkout@v5          # 從零 clone：這就是「乾淨建置」（v5 起使用 Node 24；v4 綁 Node 20，已被移除）

      - name: Build image
        run: docker build --build-arg GIT_SHA=${{ github.sha }} -t ai-demo:ci .

      # CI 不持有任何 LLM 金鑰 → App 以 mock 模式啟動。
      # 驗收標準是「服務能啟動、能回應、且確實處於 mock 模式」，而非模型回答內容。
      - name: Smoke test
        run: |
          docker run -d --name smoke -p 8000:8000 -e LLM_PROVIDER=google ai-demo:ci
          ok=0
          for i in $(seq 1 20); do
            if curl -fsS localhost:8000/healthz; then ok=1; break; fi
            sleep 2
          done
          if [ "$ok" != 1 ]; then docker logs smoke; exit 1; fi
          curl -fsS localhost:8000/version | tee /dev/stderr | grep -q '"${{ github.sha }}"'
          curl -fsS -X POST localhost:8000/api/ask \
            -H 'Content-Type: application/json' -d '{"prompt":"ping"}' \
            | tee /dev/stderr | grep -q '"is_mock":true'
          docker rm -f smoke

      - name: Publish to GHCR
        if: github.event_name == 'push'      # 品質閘門：只有進入 main 或打 tag 才產生交付物
        env:
          REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "$REGISTRY_TOKEN" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin
          IMAGE=$(echo "ghcr.io/${{ github.repository }}" | tr '[:upper:]' '[:lower:]')   # GHCR 路徑必須小寫
          TAGS="${{ github.sha }}"                        # 不可變、可追溯
          if [ "${{ github.ref_type }}" = "tag" ]; then
            TAGS="$TAGS ${{ github.ref_name }}"           # 例如 v0.1.0
          else
            TAGS="$TAGS latest"                           # 只有 main 的推送才移動 latest
          fi
          for t in $TAGS; do
            docker tag ai-demo:ci "$IMAGE:$t"
            docker push "$IMAGE:$t"
          done
```

> YAML 以**縮排**表示層級，請整段貼上、不要自行調整空白；也不要使用 Tab。

---

### 8-11 提交 workflow 並開 PR

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add .github/workflows/ci.yml` | 暫存 workflow 檔案 |
| `git commit -m "ci: ..."` | 提交，訊息以 `ci:` 開頭 |
| `git push -u origin ci/pipeline` | 推送分支並建立 upstream |
| `gh pr create --fill` | 建立 PR；**這會觸發 PR 階段的 CI** |

**複製貼上**：

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build, smoke test and publish image to GHCR"
git push -u origin ci/pipeline
gh pr create --fill
```

---

### 8-12 等待 PR 的 CI 通過後合併

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `gh pr checks --watch` | 持續顯示 PR 上 CI 的進度，直到全部完成。PR 階段**只建置與測試，不會推映像** |
| `&&` | 前一個指令成功（CI 全部 `pass`）才執行後面的指令；CI 失敗就**不會合併** |
| `gh pr merge --merge --delete-branch` | 合併 PR 並刪除遠端分支；合併會產生「推進 main」的事件 |

⚠️ 這個指令會持續更新畫面，等到全部顯示 `pass` 才會接著合併（約 1～3 分鐘）。若出現 `fail`，合併不會執行，請先排除錯誤。

⚠️ 若畫面顯示 `no checks reported`，代表 GitHub 還沒開始執行 CI，**不會合併**。等 10～20 秒再貼一次即可。

**複製貼上**：

```bash
gh pr checks --watch && gh pr merge --merge --delete-branch
```

---

### 8-13 切回 main，觸發推送映像

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 切回 main 並同步合併結果。**8-12 的合併觸發了 main 的 CI，這一次才會推映像到 GHCR** |

**複製貼上**：

```bash
git switch main && git pull
```

---

## F. 讓映像可被取用

先在網頁操作：GitHub 個人頁面 → **Packages** 分頁 → 選這個 package → **Package settings** → **Change visibility** → **Public**。

### 8-14 驗證不登入也能拉取映像

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `docker logout ghcr.io` | 先登出 GHCR，避免用到 codespace 既有的憑證而得到錯誤的結論 |
| `docker pull ghcr.io/<帳號>/<repo>:latest` | 以匿名身分拉取映像；成功即代表已公開 |

⚠️ **貼上前請替換**：`<帳號>` 與 `<repo>` 換成你的 GitHub 帳號與 repo 名稱，**一律小寫**。

**複製貼上**：

```bash
docker logout ghcr.io
docker pull ghcr.io/<帳號>/<repo>:latest
```

---

## G. 發版（release）與版本語意

### 8-15 打版本 tag 並推送

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 確認在最新的 main 上打 tag |
| `git tag -a v0.1.0 -m "first working demo"` | 建立 **annotated tag**：帶有作者、時間與說明的版本標記 |
| `git push origin v0.1.0` | 推送 tag，觸發 CI → GHCR 多出 `v0.1.0` 這個 tag |

**複製貼上**：

```bash
git switch main && git pull
git tag -a v0.1.0 -m "first working demo"
git push origin v0.1.0
```

**三種 tag 的分工**：

| Tag | 誰在用 | 特性 |
|---|---|---|
| `<SHA>` | CI/CD、部署、回溯 | 唯一、不可變，**正式環境應該指定這個** |
| `v0.1.0` | 人、發版紀錄、對外溝通 | 有語意，對應 Git tag |
| `latest` | 開發與臨時展示 | 會移動，**正式環境不該依賴** |

---

✅ **本章驗收**：

1. PR 階段的 CI 全綠且**沒有**推送映像；合併後 GHCR 出現 `<SHA>` 與 `latest`
2. package 已設為 Public，登出後仍可 `docker pull`
3. 打 `v0.1.0` tag 後，GHCR 出現同名 tag
