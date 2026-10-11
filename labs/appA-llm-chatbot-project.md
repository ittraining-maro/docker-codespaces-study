# 專案實作：大語言模型聊天機器人網站 — 實作指令集

> 本檔收錄專案實作要複製貼上的指令與程式碼；觀念說明請參考課堂簡報。

---

## 使用前必讀

### 本檔出現的內容類型

| 類型 | 是什麼 | 怎麼用 |
|---|---|---|
| **CLI Command** | 直接在終端機輸入、立即執行的單行或多行指令 | 複製整個區塊，貼到 VS Code 下方的 **Terminal**，按 Enter |
| **Markdown 片段** | 要貼進 `README.md` 的內容（非指令） | 用編輯器開啟 `README.md`，貼上並存檔 |

> 本單元沒有新的 Python Script；沿用第 7 章寫好的 `app/llm.py` 與 `app/main.py`，只透過環境變數切換供應商。

### 貼上規則

1. **一次只貼一個區塊**，看完輸出再貼下一個。
2. 標示 ⚠️ **貼上前請替換** 的區塊，要先把 `<...>` 換成你自己的內容；`<` `>` 符號本身也要刪掉。
3. 可複製的 CLI 區塊內刻意不放註解；每個指令的意義都寫在區塊上方的說明表格。
4. 本單元所有指令都在**你的 repo codespace（開發機）** 執行。

### 開始前的前提

| 前提 | 來自哪裡 | 怎麼確認 |
|---|---|---|
| 本機有 `ai-demo:dev` 映像 | 第 8 章 8-4 建置 | `docker images ai-demo` 看得到 `dev` 這個 tag |
| （選用）已建立 Codespaces Secret `GROQ_API_KEY` | 第 5 章 | 只有要做 P-6 的 Groq 情境才需要 |

---

## A. 選定主題：前職領域 × AI 助手

此節為編輯工作，沒有指令。所有人使用相同的程式骨架，但各自選定主題，建議公式是「**過去的工作領域 × AI 助手**」：

| 背景 | 主題範例 |
|---|---|
| 餐飲業 | 菜單過敏原查詢助手 |
| 會計 | 發票分類說明助手 |
| 行政 | 公文用語改寫助手 |
| 本科系 | commit message 產生器 |

只需修改三處：`SYSTEM_PROMPT`、前端文案、範例資料。

> ⚠️ 作品集 repo 是公開的，範例資料一律使用虛構或去識別化的內容。

---

## B. 供應商切換與 Ollama 保底

這一節要證明兩件事：

1. **同一個映像，只換組態就能切換供應商**。
2. 即使所有外部供應商都失效，也有一道**完全不依賴外部服務**的最後防線（本地模型 Ollama）。

```
                    ┌─ LLM_PROVIDER=ollama ─▶ 本地 Ollama（P-3～P-5）
同一個 ai-demo:dev ─┼─ LLM_PROVIDER=groq   ─▶ Groq（P-6～P-8，選用）
                    ├─ 沒有金鑰            ─▶ mock，回報原因（P-9～P-11）
                    └─ provider 拼錯       ─▶ 拒絕啟動（P-12）
```

### P-1 釋放 8000 埠

**類型**：CLI Command

**指令意義**：

| 指令片段 | 意義 |
|---|---|
| `docker rm -f ai-demo` | 強制停止並刪除第 8 章的 `ai-demo` 容器，讓出 8000 埠 |
| `2>/dev/null` | 容器不存在時，不顯示錯誤訊息 |

**複製貼上**：

```bash
docker rm -f ai-demo 2>/dev/null
```

---

### P-2 安裝並啟動 Ollama、下載本地模型

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 |
|---|---|
| `curl -fsSL https://ollama.com/install.sh \| sh` | 下載 Ollama 官方安裝腳本並執行 |
| `(ollama serve > /tmp/ollama.log 2>&1 &)` | 在背景啟動 Ollama 服務（監聽 `localhost:11434`），輸出寫進 log 檔，不佔用終端機 |
| `sleep 3 && ollama pull qwen2.5:0.5b` | 等 3 秒讓服務就緒，再下載約數百 MB 的小型模型 `qwen2.5:0.5b` |

⚠️ 模型檔會佔用 Codespace 的儲存額度，用完請執行 P-13 刪除。下載需要數分鐘。

**複製貼上**：

```bash
curl -fsSL https://ollama.com/install.sh | sh
(ollama serve > /tmp/ollama.log 2>&1 &)
sleep 3 && ollama pull qwen2.5:0.5b
```

---

### P-3 情境 1：以本地模型啟動

**類型**：CLI Command

**指令意義**：

| 參數 | 意義 |
|---|---|
| `docker run -d --name ai-local` | 以背景模式啟動容器，命名為 `ai-local` |
| `--network host` | 容器直接使用 codespace 的網路，才連得到 codespace 上的 `localhost:11434`（Ollama）。服務會直接佔用 codespace 的 8000 埠，所以不需要 `-p` |
| `-e LLM_PROVIDER=ollama` | 切換成本地 Ollama；不需要金鑰，模型預設為 `qwen2.5:0.5b` |
| `ai-demo:dev` | 第 8 章建置的同一個映像 |

**複製貼上**：

```bash
docker run -d --name ai-local --network host -e LLM_PROVIDER=ollama ai-demo:dev
```

---

### P-4 情境 1：驗證本地模型

**類型**：CLI Command

**指令意義**：

| 指令 | 意義 | 預期結果 |
|---|---|---|
| `curl -s localhost:8000/version` | 查看版本與模式 | `provider` 為 `ollama`、`mode` 為 `live` |
| `curl -s -X POST localhost:8000/api/ask ...` | 送出一個問題 | 回傳本地模型的回答，`is_mock` 為 `false` |

> 服務啟動需要 2～3 秒；若出現 `Connection refused`，稍等再貼一次。2-core 機器上本地模型回答較慢，請耐心等待。

**複製貼上**：

```bash
curl -s localhost:8000/version
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"用一句話解釋容器"}'
```

---

### P-5 情境 1：清除容器

**類型**：CLI Command

**指令意義**：`docker rm -f ai-local` 強制停止並刪除這個容器，讓出 8000 埠給下一個情境。

**複製貼上**：

```bash
docker rm -f ai-local
```

---

### P-6 情境 2（選用）：以 Groq 啟動

**類型**：CLI Command

**適用對象**：已申請 Groq 金鑰並建立 Codespaces Secret `GROQ_API_KEY` 的學員；沒有的話跳到 P-9。

**指令意義**：

| 參數 | 意義 |
|---|---|
| `-p 8000:8000` | 把容器的 8000 埠對應到 codespace 的 8000 埠 |
| `-e LLM_PROVIDER=groq` | 切換成 Groq |
| `-e LLM_MODEL="llama-3.3-70b-versatile"` | Groq 上的 model ID |
| `-e LLM_API_KEY="$GROQ_API_KEY"` | 把 Groq 的金鑰傳給程式。程式只認 `LLM_API_KEY` 這個名稱，所以在這裡把值轉交過去 |

⚠️ **貼上前請確認**：`llama-3.3-70b-versatile` 是否仍在 Groq console 的模型清單上；若已更名，換成清單上的確切 ID。

**複製貼上**：

```bash
docker run -d --name ai-groq -p 8000:8000 \
  -e LLM_PROVIDER=groq -e LLM_MODEL="llama-3.3-70b-versatile" -e LLM_API_KEY="$GROQ_API_KEY" ai-demo:dev
```

---

### P-7 情境 2（選用）：驗證 Groq

**類型**：CLI Command

**指令意義**：以 POST 送出問題；預期 `provider` 為 `groq`、`is_mock` 為 `false`。服務啟動需要 2～3 秒。

**複製貼上**：

```bash
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"hi"}'
```

---

### P-8 情境 2（選用）：清除容器

**類型**：CLI Command

**指令意義**：`docker rm -f ai-groq` 強制停止並刪除這個容器。

**複製貼上**：

```bash
docker rm -f ai-groq
```

---

### P-9 情境 3：沒有金鑰時啟動

**類型**：CLI Command

**指令意義**：

| 參數 | 意義 |
|---|---|
| `-e LLM_PROVIDER=google` | 指定 Google，但**刻意不給** `LLM_API_KEY` |

沒有金鑰是**可預期的執行狀態**，服務應該照常啟動，以示範模式（mock）運作。

**複製貼上**：

```bash
docker run -d --name ai-mock -p 8000:8000 -e LLM_PROVIDER=google ai-demo:dev
```

---

### P-10 情境 3：驗證 mock 模式

**類型**：CLI Command

**指令意義**：以 POST 送出問題；預期 `is_mock` 為 `true`，且 `reason` 說明原因（`未設定 LLM_API_KEY`）。服務啟動需要 2～3 秒。

**複製貼上**：

```bash
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"hi"}'
```

---

### P-11 情境 3：清除容器

**類型**：CLI Command

**指令意義**：`docker rm -f ai-mock` 強制停止並刪除這個容器。

**複製貼上**：

```bash
docker rm -f ai-mock
```

---

### P-12 情境 4：設定錯誤時拒絕啟動（fail fast）

**類型**：CLI Command

**指令意義**：

| 參數 | 意義 |
|---|---|
| `docker run --rm` | 以前景模式啟動，容器結束後自動刪除 |
| `-e LLM_PROVIDER=googel` | **故意把 google 拼錯** |

預期結果：容器**立即退出**，畫面印出 `ConfigError: 未知的 provider：'googel'…`。

**複製貼上**：

```bash
docker run --rm -e LLM_PROVIDER=googel ai-demo:dev
```

**情境 3 與 4 的對比是這一節的重點**：

| 情境 | 屬於哪種狀況 | 程式的處理 | 為什麼 |
|---|---|---|---|
| 3：沒有金鑰 | 可預期的執行狀態 | **降級**為 mock，並回報原因 | 服務仍能回應，使用者看得到原因 |
| 4：provider 拼錯 | 部署錯誤 | **拒絕啟動** | 若也降級，錯誤會被靜默掩蓋，看似正常其實從未呼叫模型 |

---

### P-13 刪除本地模型，釋放儲存空間

**類型**：CLI Command

**指令意義**：`ollama rm qwen2.5:0.5b` 刪除 P-2 下載的模型檔，避免持續佔用 Codespace 的儲存額度。

**複製貼上**：

```bash
ollama rm qwen2.5:0.5b
```

✅ **驗收**：同一個映像，在 `google`（第 7 章已驗證）、`groq`、`ollama`、無金鑰（mock，含 reason）四種組態下都能回應；拼錯 provider 時容器拒絕啟動。這就是「供應商鎖定風險」的現場示範。

---

## C. 作品集元素

### P-14 README 的 CI 狀態徽章

**類型**：Markdown 片段（貼到 `README.md`，不是貼到終端機）

**用途**：在 README 顯示 CI 目前是通過還是失敗，讓看作品集的人一眼知道專案有自動化測試。

⚠️ **貼上前請替換**：`<帳號>` 與 `<repo>` 換成你的 GitHub 帳號與 repo 名稱。

**貼到 `README.md` 並存檔**：

```markdown
![ci](https://github.com/<帳號>/<repo>/actions/workflows/ci.yml/badge.svg)
```

### 其他作品集元素（網頁或編輯操作，沒有指令）

| 元素 | 怎麼做 |
|---|---|
| **README** | 寫上專案簡介、架構圖、Demo GIF、環境變數說明、如何在本機執行 |
| **Open in Codespaces 按鈕** | 讓面試官一鍵開啟（用面試官自己的額度；面試官沒有你的金鑰，服務會以 mock 模式運作，畫面會標示原因） |
| **GitHub Pages** | Settings → Pages，建立常駐的作品集頁 |
| **Profile README** | 建立與帳號同名的 repo，釘選作品 |
