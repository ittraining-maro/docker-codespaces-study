# 第 7 章　VS Code 程式碼直接以容器技術執行 — 實作指令集

> 本檔收錄第 7 章實作時要複製貼上的指令與程式碼；觀念說明請參考課堂簡報。

---

## 使用前必讀

### 本檔出現的內容類型

| 類型 | 是什麼 | 怎麼用 |
|---|---|---|
| **CLI Command** | 直接在終端機輸入、立即執行的單行或多行指令 | 複製整個區塊，貼到 VS Code 下方的 **Terminal**，按 Enter |
| **Python Script** | Python 程式碼，存成 `.py` 檔後由 Python 執行 | 用編輯器開啟檔案，整段貼上並存檔 |
| **設定檔** | 套件清單等設定內容（非指令） | 用編輯器開啟檔案，整段貼上並存檔 |

### 貼上規則

1. **一次只貼一個區塊**，看完輸出再貼下一個。
2. 標示 ⚠️ **貼上前請替換** 的區塊，要先把範例值換成你自己的內容。
3. 可複製的 CLI 區塊內刻意不放註解；每個指令的意義都寫在區塊上方的說明表格。
4. 本章所有指令都在**你的 repo codespace（開發機）** 執行，終端機請位於 repo 根目錄（`pwd` 應為 `/workspaces/<repo>`）。若不是，先執行 `cd /workspaces/<repo>`（`<repo>` 換成你的 repo 名稱）。

### 本章會產生的檔案與 commit

```
feat/ai-demo 分支
 ├─ commit 1  chore: ignore secrets and build noise          ← .gitignore、.dockerignore
 ├─ commit 2  feat: add provider-agnostic LLM client ...     ← app/llm.py
 ├─ commit 3  feat: add FastAPI endpoints ...                ← app/main.py
 └─ commit 4  build: declare python dependencies             ← requirements.txt
 （dev.env 只存在本機，永遠不進版控）
```

---

## A. 開一條工作分支

### 7-1 建立並切換到工作分支

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git switch main && git pull` | 先切回 main，並拉取第 6 章合併後的最新內容，讓新分支從最新的 main 開始 |
| `git switch -c feat/ai-demo` | 建立 `feat/ai-demo` 分支並切換過去；所有開發都在分支上進行，main 保持可用 |
| `git status` | 確認分支已切換、工作目錄乾淨 |

> 前提：第 6 章的 PR 已經在網頁上合併。若沒有先切回 main，新分支會建立在第 6 章的 `feature/hello` 上，本章最後驗收的 commit 數會對不上。

**複製貼上**：

```bash
git switch main && git pull
git switch -c feat/ai-demo
git status
```

---

## B. 先寫 `.gitignore` 與 `.dockerignore`

### 7-2 建立兩個忽略清單並提交（第一個 commit）

**類型**：CLI Command

**指令意義**：

| 指令片段 | 意義 |
|---|---|
| `cat > .gitignore <<'IGN'` … `IGN` | **here-document**：把兩個 `IGN` 之間的每一行寫入 `.gitignore`。`'IGN'` 加引號代表內容原封不動寫入 |
| `.gitignore` 內容 | 擋住 **Git**：任何 `*.env`、金鑰檔與 Python 建置雜物都不准進版控 |
| `.dockerignore` 內容 | 擋住 **docker build**：`.git`、機密、文件等不需要的檔案不要進映像 |
| `git add ... && git commit` | 提交這兩個檔案，作為分支上的第一個 commit |

**為什麼是第一個 commit**：先擋住不該進版控的東西，再開始寫程式。

| 檔案 | 擋住誰 | 目的 |
|---|---|---|
| `.gitignore` | Git | 機密與雜物**不要進版控** |
| `.dockerignore` | `docker build` | 不需要的檔案**不要進映像**：縮小體積、避免把 `.git` 與機密打包進去 |

**複製貼上**（整段一次貼上）：

```bash
cat > .gitignore <<'IGN'
.env
*.env
*.key
__pycache__/
*.pyc
.venv/
IGN

cat > .dockerignore <<'IGN'
.git
.github
.devcontainer
.vscode
.env
*.env
__pycache__/
*.pyc
.venv/
labs/
README.md
IGN

git add .gitignore .dockerignore
git commit -m "chore: ignore secrets and build noise"
```

---

## C. `app/llm.py` — provider 抽象層與降級策略

### 7-3 建立 `app` 資料夾並開啟 `llm.py`

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `mkdir -p app` | 建立放程式碼的 `app` 資料夾 |
| `code app/llm.py` | 在編輯器中開啟（不存在時建立）`app/llm.py` |

**複製貼上**：

```bash
mkdir -p app
code app/llm.py
```

---

### 7-4 `app/llm.py`

**類型**：Python Script（貼到編輯器並存檔，不是貼到終端機）

**功能說明**：

這是聊天機器人的 **LLM 客戶端**，負責「把問題送給大語言模型、拿回答案」。它的設計重點是**不綁定單一供應商**：只要改環境變數，就能在 Google、Groq、本地 Ollama 之間切換；任何供應商失效時，服務不會整個崩潰，而是降級成示範模式（mock）並說明原因。

錯誤被刻意分成兩類，處理方式不同：

| 錯誤類型 | 例子 | 處理方式 | 為什麼 |
|---|---|---|---|
| **設定錯誤** | provider 拼錯、有金鑰卻沒指定 model | 拋出 `ConfigError`，服務**啟動即失敗**（fail fast） | 若也降級成 mock，問題會被靜默掩蓋 |
| **執行期錯誤** | 401 金鑰失效、404 模型不存在、429 限流、網路中斷 | **降級為 mock** 並回報原因，服務不中斷 | 外部服務出問題時，網站仍要能回應 |

**程式碼簡介**：

| 區塊 | 作用 |
|---|---|
| `PRESETS` | 各供應商的 OpenAI 相容端點網址。新增供應商只需加一行 |
| `DEFAULT_MODELS` / `KEYLESS_PROVIDERS` | 只有本地 Ollama 有預設模型、且不需要金鑰；雲端供應商必須明確指定 model ID |
| `ConfigError` | 代表設定錯誤的例外類別 |
| `LLMConfig` | 一次讀好的設定（供應商、網址、模型、金鑰、是否合併 system）。`is_mock` 判斷是否必然走示範模式 |
| `LLMResult` | 一次回答的結果，包含 `reason`（降級原因），讓前端與 log 都看得到「為什麼是 mock」 |
| `load_config()` | 從環境變數 `LLM_PROVIDER`、`LLM_MODEL`、`LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MERGE_SYSTEM` 讀取並驗證設定 |
| `ask()` | 送出一次對話請求；模型不支援 system 角色時（如 Gemma）會把系統提示併入 user 訊息；任何執行期錯誤都轉成 mock 結果，不向上拋例外 |

**貼到編輯器並存檔**：

```python
"""LLM 客戶端：provider 可切換，執行期失敗自動降級。

錯誤分兩類，處理方式刻意不同：
- 設定錯誤（provider 拼錯、有金鑰卻沒指定 model）：拋出 ConfigError，讓服務啟動即失敗（fail fast）。
  若把設定錯誤也降級成 mock，問題會被靜默掩蓋——服務看似正常，其實從未真正呼叫過模型。
- 執行期錯誤（401 金鑰失效 / 404 模型不存在 / 429 限流 / 網路中斷）：降級為 mock 並回報原因，服務不中斷。

設計背景：2026-07 GitHub Models 退役——端點必須可替換，任何供應商失效時應用程式都不能整個崩潰。
"""
from __future__ import annotations

import logging
import os
from dataclasses import dataclass

log = logging.getLogger(__name__)

# OpenAI 相容端點。新增供應商只需在此加一行。
PRESETS: dict[str, str] = {
    "google": "https://generativelanguage.googleapis.com/v1beta/openai/",
    "groq": "https://api.groq.com/openai/v1",
    "ollama": "http://localhost:11434/v1",  # Codespace 內本地執行
}

# 只有本地 ollama 給預設 model。雲端供應商的 model ID 變動頻繁，必須明確指定（從控制台或 ListModels 複製），
# 寫死預設值只會在改版後變成一個「靜默的 404」。
DEFAULT_MODELS: dict[str, str] = {"ollama": "qwen2.5:0.5b"}
KEYLESS_PROVIDERS: frozenset[str] = frozenset({"ollama"})

MOCK_REPLY = "【示範模式】目前未連上 LLM，此為預先準備的範例回應。設定有效的金鑰與模型後即可取得真實回覆。"


class ConfigError(ValueError):
    """設定錯誤：應在啟動時就讓服務失敗，而不是在執行期被降級掩蓋。"""


@dataclass(frozen=True)
class LLMConfig:
    provider: str
    base_url: str
    model: str | None
    api_key: str | None
    merge_system: bool

    @property
    def is_mock(self) -> bool:
        # 雲端供應商沒有金鑰 → 必然走 mock
        return not self.api_key and self.provider not in KEYLESS_PROVIDERS


@dataclass(frozen=True)
class LLMResult:
    text: str
    provider: str
    is_mock: bool
    reason: str | None = None  # 降級原因，前端與 log 都會顯示，避免「看起來正常其實是 mock」


def load_config() -> LLMConfig:
    provider = os.getenv("LLM_PROVIDER", "google").strip().lower()
    base_url = os.getenv("LLM_BASE_URL") or PRESETS.get(provider)
    if not base_url:
        raise ConfigError(f"未知的 provider：{provider!r}；可用值為 {sorted(PRESETS)}，或改設 LLM_BASE_URL")

    model = os.getenv("LLM_MODEL") or DEFAULT_MODELS.get(provider)

    # LLM_MERGE_SYSTEM：auto（預設）＝ 依模型自動判斷；1 ＝ 一律合併；0 ＝ 不合併
    merge_mode = os.getenv("LLM_MERGE_SYSTEM", "auto").strip().lower()
    if merge_mode not in {"auto", "0", "1"}:
        raise ConfigError(f"LLM_MERGE_SYSTEM 只能是 auto、0 或 1，目前為 {merge_mode!r}")
    if merge_mode == "auto":
        merge_system = (model or "").lower().startswith("gemma")  # Gemma 系列不接受 system 角色
    else:
        merge_system = merge_mode == "1"

    cfg = LLMConfig(
        provider=provider,
        base_url=base_url,
        model=model,
        api_key=os.getenv("LLM_API_KEY") or None,
        merge_system=merge_system,
    )
    # 沒有金鑰時一定走 mock，不需要 model；有金鑰卻沒有 model 才是設定錯誤
    if not cfg.is_mock and not cfg.model:
        raise ConfigError(f"provider {provider!r} 需要設定 LLM_MODEL（請從供應商控制台或 ListModels 複製確切的 model ID）")
    return cfg


def _mock(reason: str) -> LLMResult:
    return LLMResult(MOCK_REPLY, provider="mock", is_mock=True, reason=reason)


def ask(prompt: str, cfg: LLMConfig, system: str | None = None, timeout: float = 30.0) -> LLMResult:
    """送出一次對話請求。執行期失敗一律降級為 mock 並附上原因，不向上拋例外。"""
    if cfg.is_mock:
        return _mock("未設定 LLM_API_KEY")

    from openai import APIError, APIStatusError, OpenAI  # 延遲載入：mock 模式不需要這個依賴

    client = OpenAI(base_url=cfg.base_url, api_key=cfg.api_key or "ollama", timeout=timeout, max_retries=2)

    messages: list[dict[str, str]] = []
    if system and cfg.merge_system:
        # Gemma 等不支援 system 角色的模型：把系統提示併入 user 訊息
        messages.append({"role": "user", "content": f"{system}\n\n---\n\n{prompt}"})
    else:
        if system:
            messages.append({"role": "system", "content": system})
        messages.append({"role": "user", "content": prompt})

    try:
        resp = client.chat.completions.create(model=cfg.model, messages=messages)
    except APIStatusError as exc:  # 伺服器有回應但狀態碼非 2xx（400/401/404/429/5xx）
        log.error("%s 呼叫失敗：HTTP %s %s", cfg.provider, exc.status_code, exc.message)
        return _mock(f"{cfg.provider} 回應 HTTP {exc.status_code}")
    except APIError as exc:  # 連線失敗、逾時等，伺服器沒有回應
        log.error("%s 連線失敗：%s", cfg.provider, type(exc).__name__)
        return _mock(f"{cfg.provider} 連線失敗（{type(exc).__name__}）")

    text = resp.choices[0].message.content if resp.choices else None
    if not text:
        return _mock(f"{cfg.provider} 回傳空白內容")
    return LLMResult(text, provider=cfg.provider, is_mock=False)
```

---

### 7-5 提交 `app/llm.py`（第二個 commit）

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add app/llm.py && git commit -m "..."` | 暫存並提交 `llm.py`；每完成一個檔案就 commit 一次 |

**複製貼上**：

```bash
git add app/llm.py && git commit -m "feat: add provider-agnostic LLM client with explicit fallback"
```

---

## D. `app/main.py` — 最小 Web 服務

### 7-6 開啟 `main.py`

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `code app/main.py` | 在編輯器中開啟（不存在時建立）`app/main.py` |

**複製貼上**：

```bash
code app/main.py
```

---

### 7-7 `app/main.py`

**類型**：Python Script（貼到編輯器並存檔，不是貼到終端機）

**功能說明**：

這是聊天機器人的 **Web 服務**，以 FastAPI 撰寫，提供一個簡單的網頁讓使用者輸入問題，並把問題交給 `llm.py` 取得回答。它同時提供兩個「給機器看」的端點：健康檢查（`/healthz`）與版本資訊（`/version`），這兩個端點是第 8、9 章 CI 測試、部署驗證與版本回溯的基礎。

**程式碼簡介**：

| 區塊 | 作用 |
|---|---|
| `logging.basicConfig(...)` | 設定 log 格式；log 等級可用環境變數 `LOG_LEVEL` 調整 |
| `CONFIG = load_config()` | **啟動時就驗證設定（fail fast）**：設定錯誤時容器直接退出，錯誤訊息出現在 `docker logs` |
| `GIT_SHA` | 版本身分，建置時由 CI 寫入；本地執行時為 `dev` |
| `SYSTEM_PROMPT` | 系統提示，可用環境變數覆寫；第 9 章會修改它的預設值來製造新版本 |
| `log.info("startup ...")` | 啟動時記錄版本、供應商、模型與模式（live / mock），**絕不記錄金鑰** |
| `AskRequest` | 驗證請求內容：問題長度必須在 1～2000 字之間 |
| `GET /healthz` | 健康檢查，回傳 `{"status":"ok"}` |
| `GET /version` | 回傳執行中的版本（`git_sha`）、供應商與模式 |
| `POST /api/ask` | 接收問題，回傳答案、供應商、是否為示範模式與降級原因 |
| `GET /` | 回傳一個極簡的 HTML 聊天頁面，以 JavaScript 呼叫 `/api/ask` |

**貼到編輯器並存檔**：

```python
import logging
import os

from fastapi import FastAPI
from fastapi.responses import HTMLResponse
from pydantic import BaseModel, Field

from app.llm import ask, load_config

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"), format="%(asctime)s %(levelname)s %(name)s %(message)s")
log = logging.getLogger("app")

# 啟動時就驗證設定（fail fast）：設定錯誤時容器直接退出，錯誤訊息出現在 docker logs
CONFIG = load_config()
GIT_SHA = os.getenv("GIT_SHA", "dev")  # 建置時由 CI 寫入，用來證明「線上跑的是哪一版」
SYSTEM_PROMPT = os.getenv("SYSTEM_PROMPT", "你是一位樂於助人的助理，請用繁體中文簡潔回答。")

log.info("startup version=%s provider=%s model=%s mode=%s",
         GIT_SHA, CONFIG.provider, CONFIG.model, "mock" if CONFIG.is_mock else "live")  # 絕不記錄金鑰

app = FastAPI(title="AI Assistant Demo")


class AskRequest(BaseModel):
    prompt: str = Field(min_length=1, max_length=2000)


@app.get("/healthz")
def healthz() -> dict[str, str]:
    """健康檢查：容器、CI 與部署平台都靠它判斷服務是否活著。"""
    return {"status": "ok"}


@app.get("/version")
def version() -> dict[str, str]:
    """版本資訊：回溯（rollback）後用它確認實際執行的版本。"""
    return {"git_sha": GIT_SHA, "provider": CONFIG.provider, "mode": "mock" if CONFIG.is_mock else "live"}


@app.post("/api/ask")
def api_ask(req: AskRequest) -> dict[str, object]:
    r = ask(req.prompt, CONFIG, system=SYSTEM_PROMPT)
    return {"text": r.text, "provider": r.provider, "is_mock": r.is_mock, "reason": r.reason}


@app.get("/", response_class=HTMLResponse)
def index() -> str:
    return """<!doctype html><meta charset="utf-8"><title>AI Assistant Demo</title>
<h1>AI Assistant Demo</h1>
<textarea id="q" rows="3" style="width:100%"></textarea>
<button onclick="send()">送出</button><pre id="a"></pre>
<script>
async function send() {
  const a = document.getElementById('a'); a.textContent = '思考中…';
  try {
    const r = await fetch('/api/ask', {method:'POST', headers:{'Content-Type':'application/json'},
      body: JSON.stringify({prompt: document.getElementById('q').value})});
    if (!r.ok) { a.textContent = '請求失敗：HTTP ' + r.status; return; }
    const d = await r.json();
    a.textContent = (d.is_mock ? '[示範模式：' + d.reason + '] ' : '[' + d.provider + '] ') + d.text;
  } catch (e) { a.textContent = '連線失敗：' + e; }
}
</script>"""
```

---

### 7-8 提交 `app/main.py`（第三個 commit）

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add app/main.py && git commit -m "..."` | 暫存並提交 `main.py` |

**複製貼上**：

```bash
git add app/main.py && git commit -m "feat: add FastAPI endpoints with health and version checks"
```

---

## E. 相依套件與開發組態

### 7-9 開啟 `requirements.txt`

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `code requirements.txt` | 在編輯器中開啟（不存在時建立）repo 根目錄的 `requirements.txt` |

**複製貼上**：

```bash
code requirements.txt
```

---

### 7-10 `requirements.txt`

**類型**：設定檔（貼到編輯器並存檔，不是貼到終端機）

**用途**：宣告程式需要的 Python 套件，`pip install -r requirements.txt` 會依此安裝。

| 套件 | 用途 |
|---|---|
| `fastapi` | Web 框架，`main.py` 用它定義 API |
| `uvicorn[standard]` | 執行 FastAPI 的 ASGI 伺服器 |
| `openai` | OpenAI 相容 API 的客戶端，`llm.py` 用它呼叫各家供應商 |

`==x.y.*` 代表鎖定主次版本、允許修補版更新，是教學版的折衷做法；商業專案會再用 lock 檔固定所有間接依賴。

**貼到編輯器並存檔**：

```text
fastapi==0.115.*
uvicorn[standard]==0.32.*
openai==1.*
```

---

### 7-11 提交 `requirements.txt`（第四個 commit）

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git add requirements.txt && git commit -m "..."` | 暫存並提交套件清單 |

**複製貼上**：

```bash
git add requirements.txt && git commit -m "build: declare python dependencies"
```

---

### 7-12 設定 model ID 變數

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `export MODEL="..."` | 把第 4 章取得的確切 model ID 存成終端機變數，供下一步寫入設定檔 |

⚠️ **貼上前請替換**：`gemini-3.x-flash-lite` 換成你的實際 model ID（可用第 6 章 6-8 的指令查詢）。開新終端機分頁時需要重設。

**複製貼上**：

```bash
export MODEL="gemini-3.x-flash-lite"
```

---

### 7-13 產生開發組態檔 `dev.env`

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `REPO="$(git rev-parse --show-toplevel)"` | 取得 repo 根目錄的**絕對路徑**，不論目前在哪個子資料夾都正確 |
| `printf '...' "$MODEL" "$LLM_API_KEY" > "$REPO/dev.env"` | 依格式把供應商、模型、system 合併模式與金鑰寫入 `dev.env`；`%s` 依序被後面的兩個變數取代 |
| `chmod 600 "$REPO/dev.env"` | 設為只有自己能讀寫，保護金鑰 |
| `git status --short` | 簡短列出變更；**`dev.env` 不應出現**，證明 `.gitignore` 的 `*.env` 生效 |

⚠️ 請在 **7-12 同一個終端機分頁**執行。換了分頁的話 `$MODEL` 會是空的，寫出的 `LLM_MODEL=` 沒有值，7-14 的容器會一啟動就因為 `ConfigError` 退出；這時回到 7-12 重新設定，再執行一次本題即可。

**為什麼要寫成設定檔**：之後啟動容器時用 `--env-file` 一次注入，不必每次手打一長串 `-e`（手打正是漏設參數、默默掉進 mock 的主因）。

**複製貼上**：

```bash
REPO="$(git rev-parse --show-toplevel)"
printf 'LLM_PROVIDER=google\nLLM_MODEL=%s\nLLM_MERGE_SYSTEM=auto\nLLM_API_KEY=%s\n' "$MODEL" "$LLM_API_KEY" > "$REPO/dev.env"
chmod 600 "$REPO/dev.env"
git status --short
```

---

## F. 以容器直接執行程式碼（不打包映像）

### 7-14 用官方 Python 容器掛載原始碼執行

**類型**：CLI Command

**指令意義**：

```
 【Codespace 的工作目錄】                 【官方 python:3.12-slim 容器】
  app/llm.py  app/main.py   ──掛載（-v）──▶  /srv/app/...
  requirements.txt                          pip install → uvicorn 啟動
  dev.env  ──────────────────注入（--env-file）──▶  環境變數
```

| 參數 | 意義 |
|---|---|
| `REPO="$(git rev-parse --show-toplevel)"` | 取得 repo 根目錄的絕對路徑 |
| `docker run` | 建立並啟動一個容器 |
| `--rm` | 容器結束後自動刪除，不留垃圾 |
| `-it` | 互動模式：log 直接顯示在終端機，按 `Ctrl+C` 即可停止 |
| `--name ai-dev` | 容器名稱 |
| `-p 8000:8000` | 把容器的 8000 埠對應到 Codespace 的 8000 埠 |
| `--env-file "$REPO/dev.env"` | 以絕對路徑注入組態（含金鑰） |
| `-e PYTHONDONTWRITEBYTECODE=1` | 不在你的工作目錄產生 `__pycache__` |
| `-v "$REPO":/srv` | **把原始碼掛載進容器**：容器內的 `/srv` 就是你的 repo |
| `-w /srv` | 容器內的工作目錄 |
| `python:3.12-slim` | Docker Hub 上的官方 Python 映像 |
| `sh -c "pip install ... && uvicorn ..."` | 容器啟動後：先安裝套件，再啟動服務 |
| `--reload` | 偵測到程式碼變更時自動重啟服務 |

⚠️ 這個指令會**佔住目前的終端機**持續顯示 log。看到 `Application startup complete` 後，請**另開一個終端機分頁**執行下一步。

**複製貼上**：

```bash
REPO="$(git rev-parse --show-toplevel)"
docker run --rm -it --name ai-dev -p 8000:8000 \
  --env-file "$REPO/dev.env" \
  -e PYTHONDONTWRITEBYTECODE=1 \
  -v "$REPO":/srv -w /srv \
  python:3.12-slim \
  sh -c "pip install --no-cache-dir -q -r requirements.txt && uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
```

---

### 7-15 驗證服務（在另一個終端機分頁）

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 | 預期結果 |
|---|---|---|
| `curl -s localhost:8000/healthz` | 呼叫健康檢查端點 | `{"status":"ok"}` |
| `curl -s localhost:8000/version` | 呼叫版本端點 | `git_sha` 為 `dev`、`mode` 為 `live` |
| `curl -s -X POST localhost:8000/api/ask -H ... -d ...` | 以 POST 送出一個問題給聊天 API | 回傳含有模型回答的 JSON，`is_mock` 為 `false` |

`git_sha` 為 `dev`，代表這不是一個有版本身分的成品。

**複製貼上**：

```bash
curl -s localhost:8000/healthz
curl -s localhost:8000/version
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"用一句話解釋容器"}'
```

---

## G. 對外公開（臨時展示用）

此節為 GUI 操作，沒有指令：切到 **Ports** 分頁 → 8000 → 點地球圖示開啟；在 port 上按右鍵 → **Port Visibility** 可切換 Private / Public。

⚠️ 這個網址綁在這台 codespace 上，停機即失效，只適合臨時展示，不是部署。

完成後回到第一個終端機（7-14 的那一個），按 `Ctrl+C` 停止容器。

---

## 本章驗收

### 7-16 確認分支上的 commit

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `git log --oneline main..HEAD` | 只列出這條分支比 main 多出來的 commit，應為 **4 個** |

**複製貼上**：

```bash
git log --oneline main..HEAD
```

✅ **驗收**：

1. 以官方 Python 容器掛載原始碼，服務正常回應，且 `/version` 的 mode 為 live
2. `git log --oneline main..HEAD` 看得到四個 commit，`dev.env` 不在其中
