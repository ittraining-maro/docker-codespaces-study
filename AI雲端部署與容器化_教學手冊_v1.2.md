# AI 雲端部署與容器化 — 教學手冊

> **課程**：數據分析暨機器學習應用班：AI 雲端部署與容器化（24 小時）
> **授課教師**：黃懷慶
> **版本**：v1.2（2026-10-11），由《GitHub Codespaces 教學平台上手手冊 v6.0》依課綱教學大綱重新編排；主力模型為 Gemini 3.x Flash-Lite；修正實作流程的銜接與驗收
> **適用對象**：講師備課、助教帶課、學員課後複習

---

## 第 0 章　使用說明

### 0.1 本手冊與課綱的對應

本手冊的第 1～9 章，依序對應課綱「教學大綱」的第 1～9 項；課綱第 10 項「專案實作」收錄於附錄 A。

| 章 | 主題 | 性質 | 建議時數 |
|---|---|---|---|
| 1 | 傳統部署方式與容器部署的差異 | 講授 | 2 |
| 2 | 常見容器方案 | 講授 | 2 |
| 3 | 帳號申請（不綁信用卡） | 實作（建議課前完成） | 1 |
| 4 | Google AI Studio 試用模型及產生金鑰 | 實作 | 1 |
| 5 | GitHub、Repo、Codespace 設置 | 實作 | 2 |
| 6 | 雲端版 VS Code 及 Ubuntu CLI 入門 | 實作 | 4 |
| 7 | VS Code 程式碼直接以容器技術執行 | 實作 | 3 |
| 8 | 程式碼打包成容器映像 | 實作 | 4 |
| 9 | 以 Codespace 空容器執行映像 | 實作 | 3 |
| 附錄 A | 專案實作：大語言模型聊天機器人網站 | 專題 | 2 |
| | | **合計** | **24** |

> 時數為建議值，請依學員程度調整。第 6、8 章最容易卡關，建議保留彈性時間。

### 0.2 全課程的主線

```
第 1～2 章：為什麼要容器？有哪些方案？（觀念）
        ↓
第 3～5 章：帳號、金鑰、雲端開發環境（準備）
        ↓
第 6 章：操作工具（VS Code、CLI、Git）
        ↓
第 7 章：程式碼「掛進」容器執行 ── Inner Loop（開發）
        ↓
第 8 章：程式碼「封進」映像，CI/CD 自動建置 ── Outer Loop（交付）
        ↓
第 9 章：在沒有原始碼的機器上部署與回溯 ── 親手做一次 PaaS 的工作
```

### 0.3 貫穿全書的六條原則

這六條原則在各章反覆出現，是本課程真正要帶走的能力：

| # | 原則 | 首次出現 |
|---|---|---|
| P1 | **映像是成品，repo 是原料**。交付的是映像，不是原始碼。 | 第 1 章 |
| P2 | **程式碼與組態分離**。程式碼進映像，金鑰與設定在執行時注入。 | 第 7 章 |
| P3 | **沒有 commit 就沒有版本**。可追溯、可回溯的前提是版本控制。 | 第 7 章 |
| P4 | **檢查實際狀態，而不是假設的狀態**。看容器實際收到什麼，而非檔案寫了什麼。 | 第 9 章 |
| P5 | **指令的結果不應取決於隱藏狀態**。一律使用絕對路徑；互動輸入的指令單獨執行。 | 第 7 章 |
| P6 | **不綁定單一供應商**。服務必須能切換 LLM 供應商，失效時優雅降級。 | 第 7 章 |

### 0.4 名詞與標示約定

- **映像（image）**：本手冊行文統一稱「映像」；指令與程式碼中保留英文 `image`。
- **部署**：統一使用「部署」。
- **【開發機】**：有原始碼、用來寫程式的 codespace。**【部署機】**：第 9 章建立、沒有原始碼的空白 codespace。
- ✅ **驗收**：每節結束時學員應達成的狀態。
- 💡 **講師提示**：課堂帶領的建議，學員可略過。
- ⚠️：容易出錯、務必注意的地方。

---
## 第 1 章　傳統部署方式與容器部署的差異

> **對應課綱**：教學大綱第 1 項｜**形式**：投影片講授｜**建議時數**：2 小時

### 學習目標

- 說明傳統部署的流程與痛點
- 區分實體機、虛擬機（VM）與容器的架構差異
- 說明映像、容器、映像倉庫三者的關係
- 說明容器在**移植性**與**自動擴展**上的優勢

### 1.1 傳統部署：逐機安裝、逐機更新

傳統做法是把程式「放到每一台機器上」：

```
開發者電腦 ──手動複製／FTP──▶ 伺服器 A：安裝 Python 3.x、套件、設定檔、更新程式
                         └──▶ 伺服器 B：再做一次（版本可能已經不同）
                         └──▶ 伺服器 C：再做一次……
```

| 痛點 | 具體情境 |
|---|---|
| **環境不一致** | 開發機是 Python 3.12，伺服器是 3.8；套件版本各自不同 |
| **「在我的電腦上可以執行」** | 程式依賴某個只存在於開發者電腦的檔案或設定 |
| **擴充緩慢** | 流量暴增時，新增一台伺服器要花數小時到數天安裝設定 |
| **更新風險高** | 逐台更新檔案，中途失敗就會有部分機器是新版、部分是舊版 |
| **難以回溯** | 新版出問題時，要把每台機器的檔案「改回去」 |

**根本原因**：交付的是「原始碼 + 一份安裝說明」，而安裝說明每執行一次，結果都可能不同。

### 1.2 實體機、虛擬機與容器

```
   實體機                  虛擬機（VM）                  容器
┌───────────┐     ┌─────────┬─────────┐     ┌──────┬──────┬──────┐
│   App     │     │  App A  │  App B  │     │App A │App B │App C │
│  函式庫    │     │ 函式庫   │ 函式庫   │     │函式庫 │函式庫 │函式庫 │
│           │     │ Guest OS│ Guest OS│     ├──────┴──────┴──────┤
│   OS      │     ├─────────┴─────────┤     │   容器執行環境       │
│           │     │    Hypervisor     │     │  （Docker 等）      │
│  硬體      │     │   Host OS／硬體    │     │   Host OS／硬體      │
└───────────┘     └───────────────────┘     └────────────────────┘
```

| 比較 | 實體機 | 虛擬機 | 容器 |
|---|---|---|---|
| 隔離層級 | 無 | 整個作業系統 | 行程（process）層級，共用核心 |
| 啟動時間 | 分鐘 | 分鐘 | **秒** |
| 體積 | — | GB 級（含完整 OS） | MB～數百 MB |
| 單機密度 | 1 | 數個～數十個 | 數十個～數百個 |
| 環境一致性 | 低 | 中（映像檔可複製，但笨重） | **高** |

### 1.3 容器的三個核心概念

```
  Dockerfile ──build──▶  映像（Image）──push──▶ 映像倉庫（Registry）
  （食譜）              （冷凍調理包）           （中央倉庫）
                              │
                            run（可同時跑多份）
                              ▼
                        容器（Container）
                         （加熱上桌的那一份）
```

| 概念 | 是什麼 | 類比 |
|---|---|---|
| **映像（image）** | 程式碼＋執行環境＋相依套件打包成的**唯讀**檔案 | 冷凍調理包 |
| **容器（container）** | 映像執行起來的實例；可同時跑很多個 | 加熱後的那一份 |
| **映像倉庫（registry）** | 存放與分發映像的服務，例如 Docker Hub、GHCR | 中央倉庫 |

> **P1：映像是成品，repo 是原料。** 傳統部署交付的是原料加食譜，每台機器自己煮；容器部署交付的是成品，每台機器加熱就好。第 9 章會在一台沒有任何原始碼的機器上親手驗證這一點。

### 1.4 容器部署的優勢

#### 移植性：一次打包，到處執行

同一個映像，在開發機、測試機、雲端平台上執行的結果相同，因為執行環境已經被封裝在映像裡。

```
            同一個映像 app:7ae9e9b
   ┌──────────────┼──────────────┐
   ▼              ▼              ▼
 開發機         測試環境        雲端 PaaS
（Codespace）               （Cloud Run 等）
```

**唯一會變的是組態**（金鑰、連線位址），而組態是在執行時注入的（P2，第 7 章）。

#### 自動擴展：複製容器，而不是安裝機器

因為映像已經包含所有東西，「擴充一台」就只是「多跑一個容器」，以秒計算：

```
流量低：  [容器]
流量高：  [容器][容器][容器][容器]     ← 平台依 CPU／請求數自動增減
流量降：  [容器][容器]
```

這是 Kubernetes 與雲端 PaaS 自動擴展（autoscaling）的基礎。傳統部署要做到同樣的事，得先自動化整個安裝流程，而且仍然要以分鐘計。

#### 版本管理：切換映像即可部署或回溯

每個映像都有標籤（tag）。部署新版是「改跑新標籤」，回溯是「改回舊標籤」，不需要重新安裝、也不會有半新半舊的機器。第 9 章會實際操作一次。

### 1.5 小結

| | 傳統部署 | 容器部署 |
|---|---|---|
| 交付物 | 原始碼＋安裝說明 | 映像 |
| 環境一致性 | 依賴人工 | 由映像保證 |
| 擴充 | 安裝新機器（小時～天） | 多跑一個容器（秒） |
| 回溯 | 逐台改回舊檔 | 改跑舊標籤 |

💡 **講師提示**：本章以觀念為主，不需實機操作。可以請學員分享過去「在我電腦可以跑、換台電腦就壞掉」的經驗作為開場。

---
## 第 2 章　常見容器方案

> **對應課綱**：教學大綱第 2 項｜**形式**：投影片講授｜**建議時數**：2 小時

### 學習目標

- 認識地端、容器編排與雲端三類容器方案
- 理解雲端 PaaS 容器平台替你自動化了哪些工作
- 了解本課程為何選用 GitHub Codespaces 作為教學平台

### 2.1 方案地圖

```
 單機 ─────────────────────────────────────────────▶ 大規模
 自己管 ───────────────────────────────────────────▶ 平台代管

 Docker        Docker Compose      Kubernetes (K8s)       雲端 PaaS 容器平台
 跑一個容器    一台機器跑多個容器   多台機器的容器編排       丟映像上去就好
```

### 2.2 地端方案：Docker 與 Docker Compose

- **Docker**：建置映像、執行容器的標準工具。本課程第 7～9 章都使用 Docker。
- **Docker Compose**：用一個 YAML 檔描述「網站＋資料庫＋快取」等多個容器，一個指令一起啟動。適合單機的多服務應用與開發環境。

### 2.3 容器編排：Kubernetes（K8s）

當容器數量多到需要跨多台機器管理時，就需要**容器編排**（orchestration）：

| K8s 負責的事 | 說明 |
|---|---|
| 排程 | 決定每個容器跑在哪台機器上 |
| 自我修復 | 容器掛掉自動重啟、機器故障自動搬移 |
| 自動擴展 | 依負載增減容器數量 |
| 滾動更新與回溯 | 逐步替換成新版，出問題自動退回 |
| 服務發現與負載平衡 | 讓流量平均分配到各個容器 |

K8s 功能強大但學習曲線陡、維運成本高，通常是大型團隊或有專職維運人員時才自建。

### 2.4 雲端 SaaS / PaaS 容器平台

「我只提供映像，其他交給平台」：

| 平台 | 供應商 | 特性 |
|---|---|---|
| **Cloud Run** | Google Cloud | 以請求量自動擴展，沒有流量時可縮到 0 |
| **ECS / Fargate** | AWS | 不需管理伺服器的容器執行環境 |
| **Azure Container Apps** | Microsoft Azure | 以 K8s 為基礎的代管服務 |
| Render、Railway 等 | 第三方 | 介面簡單，常見於個人專案 |

**PaaS 替你自動化了哪些事**——第 9 章結束時，你會親手做過其中最核心的部分：

| 本課程手動做的事 | PaaS 幫你做的事 |
|---|---|
| `deploy.sh`：拉取映像、啟動、健康檢查 | 自動拉取並啟動、健康檢查，掛掉自動重啟 |
| `make-env.sh` 產生組態檔 | 控制台或 secret 管理介面 |
| 自己開連接埠 | 自動配置網域與 TLS 憑證 |
| `deploy.sh <舊版本>` | 一鍵回溯（rollback） |
| 一台機器 | 自動擴縮與負載平衡 |

### 2.5 雲端開發環境：GitHub Codespaces

Codespaces 本身就是容器技術的應用：**GitHub 開一台 VM，在裡面跑一個開發用的容器，再把瀏覽器版 VS Code 接上去**。

本課程選用它的理由：

| 需求 | Codespaces 如何滿足 |
|---|---|
| 不花錢、不綁信用卡 | 個人帳號每月有免費額度，沒有付款方式時用完即停，不會扣款 |
| 不需在學員電腦安裝軟體 | 只要瀏覽器 |
| 每個人環境一致 | 所有人開出來的環境相同 |
| 能跑 Docker | 環境內建 Docker |
| 能做成作品集 | 程式碼、CI 紀錄、映像都留在學員自己的 GitHub 帳號 |

### 2.6 如何選擇

| 情境 | 建議方案 |
|---|---|
| 學習、開發、單一服務 | Docker |
| 單機多服務（網站＋資料庫） | Docker Compose |
| 小團隊、想專注在產品 | 雲端 PaaS |
| 大規模、多服務、有維運團隊 | Kubernetes |

💡 **講師提示**：本章的 PaaS 對照表在第 9 章結尾會再出現一次。建議在這裡先讓學員留下印象，第 9 章再回頭對照，學員會很有感。

---
## 第 3 章　帳號申請（不綁信用卡）

> **對應課綱**：教學大綱第 3 項｜**形式**：實作（**建議學員於開課前完成**）｜**建議時數**：1 小時

### 學習目標

- 完成 GitHub 與 Google AI Studio 帳號申請，全程不綁定信用卡
- 完成帳號的基本安全設定
- 理解「如何確認自己沒有在花錢」

### 3.1 ⚠️ 請在開課前完成申請

**請學員在開課前，於家中以個人網路自行完成本章所有帳號申請。**

教室中多人在同一個網路環境、短時間內同時註冊，容易被平台的防濫用機制判定為異常，導致被要求額外驗證、甚至帳號遭到暫停。

課前需備妥：
- 一個可收信的 email
- 一支可接收驗證碼的手機

> 本課程使用的所有平台皆為免費方案，**全程不需要、也不應該綁定信用卡**。

### 3.2 GitHub 帳號

- 到 github.com 註冊，使用 **GitHub Free**。
- Settings → Password and authentication → **啟用 2FA**。
- Settings → Emails → 勾選 **Keep my email addresses private**（避免 email 永久留在公開 commit 紀錄）。

✅ **驗收**：能登入 GitHub，且已啟用 2FA。

### 3.3 Google AI Studio 帳號

1. 前往 `aistudio.google.com`，以 Google 帳號登入（沒有 Google 帳號請先註冊）。
2. 首次登入時同意服務條款。
3. **不需要、也不要**建立 Google Cloud 帳單帳戶。

#### 為什麼不會產生費用：認識計費的分界

免費模型**並非「沒有付費層」** —— Google 的 rate limits 表中，同一個模型通常同時列在 Free、Tier 1、Tier 2、Tier 3。真正讓學員不會產生費用的機制是：

> **新帳號一律從 Free Tier 開始；要進入付費層，必須主動為 Google Cloud 專案啟用 Cloud Billing 並連結帳單帳戶。**

只要專案停留在 Free Tier，超量的結果就只是 429，不會計費。**所以要防的不是模型，而是「學員自己去啟用了 Cloud Billing」。**

⚠️ **必須明確告知學員的三件事**：
1. **全程不要啟用 Cloud Billing、不要連結帳單帳戶。**
2. 到 AI Studio 的 API keys 頁面確認專案的 Plan 欄顯示為 **Free**。
3. 一旦有任一專案啟用帳單，該帳號的 AI Studio 使用就會落入付費服務條款的範圍。

（各模型在免費層的實際額度，以 AI Studio 控制台顯示與實測為準，見第 4 章 4.2 的模型選擇表。）

✅ **驗收**：能登入 Google AI Studio，且未建立任何帳單帳戶。

### 3.4 （選用）Groq 帳號

Groq 是另一家提供免費 LLM API 的平台，作為備援供應商，在附錄 A 的「供應商切換」中使用。

前往 `console.groq.com`，以 email 註冊即可，同樣不需要信用卡。

---
## 第 4 章　Google AI Studio 試用模型及產生金鑰

> **對應課綱**：教學大綱第 4 項｜**形式**：實作｜**建議時數**：1 小時

### 學習目標

- 在 AI Studio 的網頁介面試用免費模型
- 產生 API 金鑰，並記下確切的 model ID
- 理解 API 金鑰的安全管理原則

### 4.1 在 Playground 試用模型

1. 登入 `aistudio.google.com`，進入對話（Chat／Playground）介面。
2. 在模型選單中選擇 **Gemini 3.x Flash-Lite**（本課程的主力模型）；也可以切換到 Gemma 4，比較兩者的回應速度。
3. 輸入幾個問題試試看，觀察回應速度與品質。
4. 試著加入「系統指示」（system instruction），觀察模型是否接受。

這一步讓學員先在網頁上感受模型的能力，之後的程式只是用 API 做同一件事。

### 4.2 產生 API 金鑰並記下 model ID

1. 前往 `aistudio.google.com`，用 Google 帳號登入。
2. 找到 **Get API key** → **Create API key**。
3. 複製金鑰，**不需要建立計費帳戶、不需要信用卡**。
4. 在模型清單中確認標示為免費的模型，**把確切的 model ID 原樣複製下來**（例如 Gemini 3.x Flash-Lite 的完整 ID）。model ID 會隨版本更動，不要憑印象輸入。
5. 把控制台顯示的 RPM / RPD 截圖存檔，寫進本文件開頭的現況表。
6. ⚠️ 免費層的輸入內容可能被用於改善 Google 服務，**課堂上嚴禁輸入真實個資**。

#### 選用哪個模型（講師實測，2026-10-02）

| 模型 | 免費層每日請求數 | 回應速度 | 本課程定位 |
|---|---|---|---|
| **Gemini 3.x Flash-Lite** | 約 500 次 | 快 | **主力** |
| Gemma 4 | 約 1,600 次 | 很慢 | 備援（主力額度用完時） |
| 其他 Gemini 3.x 模型 | 不到 100 次 | — | 不建議 |
| Gemini 2.x（含 2.5 Flash-Lite） | 費率表上仍有列出 | — | **實際無法呼叫，不要使用** |

選擇主力模型的考量：

- **額度要夠學員回家練習**。學員課後會自行修改程式、反覆除錯，每日不到 100 次的模型很快就會用完，學習會被迫中斷。
- **速度影響學習體驗**。Gemma 4 額度最多，但每次回應等待過久，不適合反覆嘗試，因此只作為備援。
- **費率表列出不代表可用**。Gemini 2.x 仍出現在費率表上，實際卻無法呼叫。一律以第 6 章 6.3 的 ListModels 與實際呼叫結果為準。

> 額度與可用模型會隨 Google 的政策調整，每期開課前請重新實測（見附錄 C）。

⚠️ **若改用 Gemma 4**：Gemma 系列可能不支援 `system` 角色。第 7 章的 `llm.py` 預設 `LLM_MERGE_SYSTEM=auto`，偵測到 model ID 以 `gemma` 開頭時，會自動把系統提示併入第一則 user 訊息，不需要手動設定。

### 4.3 （選用）產生 Groq 金鑰

1. 前往 `console.groq.com/keys`，用 email 註冊。
2. **Create API Key**，複製金鑰。
3. 到 console 的 limits 頁面截圖記錄實際額度（樣本值約 30 RPM、每日 1,000 次請求）。

### 4.4 金鑰的安全原則

API 金鑰就是「你本人」的身分。任何拿到金鑰的人，都能以你的名義使用額度。

| 原則 | 做法 | 本課程在哪裡實踐 |
|---|---|---|
| **不寫進程式碼** | 程式只從環境變數讀取金鑰 | 第 7 章 `llm.py` |
| **不上傳到 GitHub** | 用 `.gitignore` 擋住含金鑰的檔案 | 第 7 章 7.3 |
| **加密保存** | 存進 Codespaces Secrets | 第 5 章 5.5 |
| **不顯示在畫面上** | 輸入時不回顯；檢查時只顯示前 4 碼 | 第 6 章 6.3、第 9 章 |
| **外洩就作廢** | 到 AI Studio 刪除舊金鑰、重新產生 | — |

> 本章只在網頁上產生金鑰。用命令列驗證金鑰是否可用，安排在第 6 章 6.3，作為 Ubuntu CLI 的第一個實作練習。

✅ **驗收**：已取得 API 金鑰，並記下確切的 model ID。

---
## 第 5 章　GitHub、Repo、Codespace 設置

> **對應課綱**：教學大綱第 5 項｜**形式**：實作｜**建議時數**：2 小時

### 學習目標

- 理解 GitHub 的核心名詞，並能對照到 GCP／AWS 的概念
- 完成 Codespaces 的防超量設定
- 建立 repo、開啟第一台 Codespace
- 以 Codespaces Secrets 安全保存 API 金鑰
- 掌握 Codespace 的停止與刪除，管理免費額度

### 5.1 名詞地圖（GitHub ↔ GCP ↔ AWS）

| GitHub 名詞 | 白話解釋 | GCP | AWS |
|---|---|---|---|
| **Repository（repo）** | 專案的程式碼倉庫，含完整修改歷史 | Cloud Source Repositories | CodeCommit |
| **Codespace** | 雲端開發機，瀏覽器打開就是 VS Code | Cloud Workstations / Cloud Shell | Cloud9 |
| **Dev Container** | Codespace 裡真正讓你寫程式的那個容器 | Workstation container 映像 | — |
| **`devcontainer.json`** | 描述 Dev Container 長相的設定檔，放在 repo 裡 | Workstation config | — |
| **Machine type** | Codespace 規格（2-core、4-core…） | Machine type | Instance type |
| **Codespaces Secrets** | 加密機密，開機時注入成環境變數 | Secret Manager | Secrets Manager |
| **PAT** | 代表「你本人」的 token，可限縮權限 | 使用者憑證 | IAM access key |
| **`GITHUB_TOKEN`** | 代表「這次工作流程」的臨時 token | Service account | IAM role |
| **Actions / Workflow / Runner** | 事件觸發的自動化流水線 / 流程定義 / 執行機器 | Cloud Build | CodeBuild / CodePipeline |
| **GHCR** | GitHub 的容器映像倉庫 | Artifact Registry | ECR |
| **GitHub Pages** | 靜態網站託管 | Cloud Storage 靜態站 | S3 static hosting |

> 表中部分 GCP / AWS 服務（如 Cloud Source Repositories、Cloud9）已停止受理新客戶，**僅作概念類比**。

**Git 協作名詞**：Commit（存檔快照）、Push（上傳）、Branch（平行開發線）、Pull Request（合併請求與審查）、Fork（複製到自己帳號）、Template repository（可被「一鍵複製成新專案」的範本）。

**一句話心智模型**：Codespace＝GitHub 幫你開一台 VM，在裡面起一個容器（repo 有 `devcontainer.json` 就依它建置，沒有就用預設映像），再把 VS Code 接上去。



### 5.2 Codespaces 防超量設定

前往 `github.com/settings/codespaces`：

| 設定 | 建議值 | 原因 |
|---|---|---|
| Default idle timeout | 30 分鐘或更短 | 停機後不再消耗運算時數 |
| Default retention period | 7 天 | 自動刪除，避免儲存額度被吃光 |
| Default region | Southeast Asia | 台灣連線延遲較低 |

#### 確認沒有付款方式

前往 `github.com/settings/billing`，確認未綁定任何付款方式。這是「額度用完即停、不會扣款」的前提。

✅ **驗收**：能在 billing 頁面看到 Codespaces 的本月用量，且無付款方式。



### 5.3 建立 Repo

右上角「+」→ **New repository** → 名稱 `codespaces-lab` → **Public** → 勾選 **Add a README file** → Create。

### 5.4 開啟第一台 Codespace

repo 頁面 → 綠色 **Code** 按鈕 → **Codespaces** 分頁 → **Create codespace on main**。等 1～2 分鐘。

#### 確認你開的是 Codespaces，而不是 github.dev

GitHub 有兩種外觀幾乎相同、行為完全不同的網頁編輯器。認錯會導致難以理解的版控行為。

| | **github.dev**（在 repo 頁面按 `.` 鍵） | **Codespaces** |
|---|---|---|
| 終端機 | **沒有** | 有 |
| 執行 Docker | **不能** | 能 |
| 本地 clone | **沒有** | 有完整的 Git 儲存庫 |
| 按下 Commit 的結果 | **直接寫入遠端分支**（沒有本地 commit 的概念） | 建立本地 commit，之後還要 push |
| 用途 | 快速改一兩個檔案 | 完整開發環境 |

本手冊全程使用 **Codespaces**。確認方法：下方有終端機、`docker version` 可執行。

### 5.5 以 Codespaces Secrets 保存 API 金鑰

1. `github.com/settings/codespaces` → **Codespaces secrets** → **New secret**
2. Name：`LLM_API_KEY`，Value 貼上 **Google AI Studio** 的金鑰（命名規則：不可用 `GITHUB_` 開頭、不可用數字開頭）
3. Repository access：勾選 `codespaces-lab`
4. 再建立一個 `GROQ_API_KEY`，存 **Groq** 的金鑰（附錄 A 的供應商切換會用到；兩家的金鑰不能混用）

#### 重新啟動 Codespace（最常漏掉的一步）

Secret 只在**建立或重新啟動**時注入。按 `F1` → `Codespaces: Stop Current Codespace`，再從 repo 重新開啟。

### 5.6 停止與刪除：額度管理

| 動作 | 效果 | 消耗 |
|---|---|---|
| **Stop** | 關機保留磁碟 | 停止計運算，**儲存持續計費** |
| **Delete** | 整台移除 | 全部停止 |

收工三步驟：確認 `git push` 完成 → `github.com/codespaces` 刪除用不到的 codespace → 到 billing 檢查用量。

**核心觀念**：Codespace 是可拋棄的，repo 才是本體（cattle, not pets）。



✅ **驗收**：能建立、停止、刪除 Codespace，並在帳單頁面看到用量；`LLM_API_KEY` 已存成 Secret。

---

## 第 6 章　Codespace 的雲端版 VS Code 及 Ubuntu CLI 入門

> **對應課綱**：教學大綱第 6 項｜**形式**：實作｜**建議時數**：4 小時

### 學習目標

- 熟悉雲端版 VS Code 的介面，以及與桌面版的差異
- 掌握 Ubuntu 命令列的基本操作
- 以命令列驗證 API 金鑰
- 用 Git 指令與 Source Control 面板完成版本控制的基本循環
- 避開 Source Control 面板的常見陷阱

### 6.1 雲端版 VS Code 介面

- **Explorer**（左側）：檔案樹
- **Terminal**（下方）：容器內的 bash
- **Ports**（下方分頁）：對外開放的 port
- **Source Control**（左側分支圖示）：圖形化 Git

#### 與桌面版 VS Code 的差異

| 項目 | 桌面版 VS Code | Codespaces（瀏覽器） | 影響 |
|---|---|---|---|
| **Git 身分** | 要自己設 `user.name` / `user.email`、SSH key | **已自動帶入**，直接可 push | 少一道設定，但也看不到憑證在哪 |
| **擴充套件** | 你裝過的 GitLens 等全都在 | **預設沒有**（除非開啟 Settings Sync），每台新 codespace 都是乾淨的 | 熟悉的圖形化輔助工具不見了 |
| **快捷鍵** | 作業系統原生 | 部分被**瀏覽器攔截**（例如 `Ctrl+W` 關分頁、`Ctrl+N` 開新視窗；Firefox 的 `Ctrl+Shift+P` 會開私密視窗） | 操作手感差異最大的來源 |
| **Command Palette** | `Ctrl/Cmd+Shift+P` | 建議改用 **`F1`**，不會被瀏覽器搶走 | 記住 F1 就解決大半問題 |
| **檔案總管拖放** | 與本機檔案互通 | 要用上傳 / 下載，或 `gh` CLI | 檔案進出較不直覺 |
| **終端機** | 本機 shell | 容器內的 bash | 其實更乾淨，適合教學 |

**最重要的一個認知**：你覺得難用的多半不是 Git 本身，而是「**Source Control 面板的操作邏輯**」。它把 Git 的三個區域（工作目錄、暫存區、儲存庫）用圖示表達，但沒有寫出名字，所以第一次用會不知道按鈕在做什麼。下一節把它拆開。

### 6.2 Ubuntu CLI 入門

Codespace 下方的終端機（Terminal）是一個 Ubuntu 的 bash。本課程的實作大多在這裡完成。

#### 先確認你所在的環境

```bash
nproc && free -h        # 底層 VM 分配到的規格
cat /etc/os-release     # Dev Container 的 OS
docker version          # 同時看到 Client 與 Server 才代表可用
echo $CODESPACE_NAME
```

#### 常用指令

| 類別 | 指令 | 用途 |
|---|---|---|
| **位置** | `pwd` | 顯示目前所在的資料夾 |
| | `cd <資料夾>`、`cd ..`、`cd ~` | 切換資料夾、回上一層、回家目錄 |
| | `ls`、`ls -l`、`ls -A` | 列出檔案、顯示詳細資訊、包含隱藏檔 |
| **檔案** | `mkdir -p <資料夾>` | 建立資料夾（含中間層） |
| | `cat <檔案>`、`head -n 5 <檔案>` | 顯示檔案內容、只看前 5 行 |
| | `cp`、`mv`、`rm` | 複製、移動或改名、刪除（**沒有資源回收筒**） |
| | `code <檔案>` | 在編輯器中開啟檔案 |
| **搜尋** | `grep <文字> <檔案>` | 找出含有某段文字的行 |
| **權限** | `ls -l`、`chmod 600 <檔案>` | 查看權限、設為只有自己能讀寫 |
| **環境變數** | `echo $變數`、`export 變數=值` | 顯示、設定環境變數 |
| | `env`、`env \| grep LLM` | 列出所有環境變數、篩選 |
| **網路** | `curl <網址>` | 從命令列發出 HTTP 請求 |
| **說明** | `<指令> --help`、`man <指令>` | 查看用法 |

#### 三個必懂的觀念

**① 絕對路徑 vs 相對路徑**

```bash
cat /workspaces/codespaces-lab/README.md   # 絕對路徑：從 / 開始，在哪裡執行結果都一樣
cat README.md                              # 相對路徑：取決於你「現在在哪個資料夾」
```

> **P5：指令的結果不應取決於隱藏狀態。** 本手冊中凡是牽涉設定檔的指令，一律使用絕對路徑。

**② 環境變數只活在當下的終端機**

```bash
export MY_NAME=Arthur
echo $MY_NAME          # 顯示 Arthur
# 開一個新的終端機分頁再執行 echo $MY_NAME → 空的
```

需要長期保存的設定要寫進檔案，或存成 Codespaces Secret（第 5 章 5.5）。

**③ 整段貼上指令的陷阱**

需要你輸入內容的指令（例如 `read`），**不可以和其他指令一起整段貼上**。貼上的下一行文字會被當成你的輸入：

```bash
read -rp "請輸入名字：" NAME    # 若與下一行一起貼上……
echo "hello $NAME"              # ……這一行會被當成 NAME 的值，而不會被執行
```

> 原則：**互動輸入的指令一律單獨貼上執行。**

#### 練習

```bash
cd ~ && pwd                          # 回到家目錄
mkdir -p ~/practice && cd ~/practice
echo "hello codespace" > note.txt    # > 寫入檔案（覆蓋）
echo "second line" >> note.txt       # >> 附加到檔案結尾
cat note.txt
grep second note.txt
chmod 600 note.txt && ls -l note.txt # 權限應為 -rw-------
cd ~ && rm -r ~/practice             # 清除練習資料夾
```

✅ **驗收**：能說出目前所在位置、建立與刪除檔案、設定並讀取環境變數。

### 6.3 CLI 實作：驗證 API 金鑰

第 5 章已經把 Google AI Studio 的金鑰存成 Codespaces Secret。現在用 `curl` 從命令列實際呼叫一次 API，確認金鑰可用。

```bash
echo "${LLM_API_KEY:0:4}****"   # 只印前 4 碼，永遠不要輸出完整金鑰

# 步驟 1：先問 API「這把金鑰能用哪些模型」——排錯的第一步永遠是這個
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key=$LLM_API_KEY" \
  | python3 -c "import sys,json; [print(m['name'], m.get('supportedGenerationMethods')) for m in json.load(sys.stdin).get('models', [])]"

# 步驟 2：從清單中複製確切的 model ID（去掉前綴 models/），存成變數
export MODEL="gemini-3.x-flash-lite"   # ← 換成清單上 Gemini 3.x Flash-Lite 的實際 ID

# 步驟 3：用 OpenAI 相容端點呼叫
curl -sS https://generativelanguage.googleapis.com/v1beta/openai/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"用一句話解釋容器\"}]}"

# 步驟 4：確認是否支援 system 角色
curl -sS https://generativelanguage.googleapis.com/v1beta/openai/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"system\",\"content\":\"用繁體中文回答\"},{\"role\":\"user\",\"content\":\"hi\"}]}"
# Gemini 支援 system 角色；若改用 Gemma 而回 400 並提到 system，程式的 auto 模式會自動處理
```

> `MODEL` 是終端機變數，**開新的終端機分頁就會消失**。第 7 章起會改用設定檔（`dev.env`）保存，避免這個問題。

**看錯誤碼判斷成因**（呼叫失敗時保留完整 JSON）：

| HTTP / status | 意思 | 處理 |
|---|---|---|
| 400 `INVALID_ARGUMENT` | **看訊息內容區分**：`API key not valid` → 金鑰錯誤（**Google 對無效金鑰回 400，不是 401**）；提到 model → 名稱格式錯誤；提到 system / developer instruction → 不支援 system 角色 | 依訊息處理：重新複製金鑰 / 對照步驟 1 的清單 / 確認 `LLM_MERGE_SYSTEM` 為 `auto` 或 `1` |
| 401 / 403 | 認證或權限問題（例如專案未啟用 Generative Language API） | 確認 API 已啟用；重建金鑰 |
| 404 `NOT_FOUND` | 模型名稱不存在 | **model ID 寫錯或已停用**，從步驟 1 的清單重新複製 |
| 429 `RESOURCE_EXHAUSTED` | 額度用盡；若額度為 0，代表該模型不在免費層 | 等待重置，或改用其他模型 |

✅ **驗收**：步驟 1 列得出模型清單，步驟 3 回傳含有模型回答的 JSON。

### 6.4 Git 版本控制：Source Control 面板解剖

點左側邊欄的**分支圖示**（第三個，有數字徽章的那個）打開面板。由上而下：

```
┌─ Source Control ─────────────────────────────┐
│ [訊息輸入框]            ← commit message       │
│ [✓ Commit] 按鈕         ← 等同 git commit      │
├──────────────────────────────────────────────┤
│ Staged Changes (2)      ← 暫存區 git add 過的  │
│   app/main.py      [−]  ← 按「−」退回工作目錄  │
├──────────────────────────────────────────────┤
│ Changes (3)             ← 工作目錄，尚未 add   │
│   README.md        [+]  ← 按「+」加入暫存區    │
│                    [↩]  ← 捨棄變更（不可復原！）│
└──────────────────────────────────────────────┘
最下方狀態列：  main*  ⟳ 2↓ 1↑   ← 分支名 / 同步按鈕 / 待拉取·待推送
（分支尚未 publish 時沒有數字，只顯示雲朵圖示）
```

對應關係一次記清楚：

| 面板動作 | Git 指令 | 說明 |
|---|---|---|
| 檔案右側 **+**（Stage Changes） | `git add <file>` | 放進暫存區 |
| 檔案右側 **−**（Unstage Changes） | `git restore --staged <file>` | 退出暫存區 |
| 檔案右側 **↩**（Discard Changes） | `git restore <file>` | **變更直接消失，無法復原** |
| 輸入訊息後按 **Commit** | `git commit -m "..."` | 若暫存區是空的，VS Code 會問要不要 stage 全部；**不要點 Always**（見 6.5） |
| 狀態列 **⟳ Sync Changes** | `git pull` + `git push` | 一個按鈕做兩件事，是最常誤解的地方 |
| 狀態列 **Publish Branch** | `git push -u origin <branch>` | 新分支第一次推送時才出現 |
| 狀態列 **分支名稱** | `git switch` | 點下去可切換或建立分支 |

⚠️ **三個最常踩的坑**：
1. **↩ 與 − 長得像但後果天差地遠**。− 只是退出暫存區，↩ 會直接丟掉你寫的程式碼。教學時務必強調。
2. **Sync Changes 會先 pull**。如果遠端有別人的 commit，可能觸發 merge，初學者會突然被丟進一個看不懂的畫面。
3. **面板的數字徽章是「變更檔案數」**，不是待推送的 commit 數。待推送數要看狀態列的 `↑N`（需有 upstream）。

### 6.5 Commit 按鈕的陷阱與工作區設定

即使在 Codespaces，Source Control 面板的預設行為仍會讓人以為「commit 自己跑到遠端去了」。

| 陷阱 | 現象 | 根因 |
|---|---|---|
| **① 藍色主按鈕會「變身」** | 第一次按是 **Commit**；commit 完，同一個位置變成 **Sync Changes ↑1**，順手再按一次就推上遠端了 | 面板的主按鈕（action button）會依狀態自動換成「下一步」動作 |
| **② 下拉選單的 Commit & Push** | 用過一次 **Commit & Push / Commit & Sync** 之後，主按鈕沿用該動作 | `git.rememberPostCommitCommand` 開啟時會記住上次的後續動作 |
| **③ 點了「Always」** | 暫存區為空時按 Commit，跳出「要不要直接 stage 全部並提交」；點了 **Always** 之後再也不問，每次都提交全部檔案 | 點 Always 會把 `git.enableSmartCommit` 改成 true（**預設是 false**） |
| **④ Sync 隱含 pull** | 按下 Sync 後突然進入合併畫面 | **Sync Changes = `git pull` + `git push`** |

> 本手冊作者實測遇到的「commit 跑到遠端」，最可能是 ① 或 ②。

**解法：在 repo 的 `.vscode/` 資料夾把行為釘死**，讓 GUI 的每個動作與一條指令一對一，學員才學得到 Git 的狀態模型。

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

**套用步驟**：

```bash
mkdir -p .vscode                 # 接著用編輯器建立上面兩個檔案
git add .vscode/settings.json .vscode/extensions.json
git commit -m "chore: pin git behavior and recommend extensions"
git push
```

`settings.json` 存檔後立即生效。`extensions.json` 則不同：**VS Code 只在開啟工作區時檢查建議清單**，存檔不會跳出提示。

**在目前的 codespace 立即安裝建議的擴充套件**：
1. `F1` → **Extensions: Show Recommended Extensions**（或在擴充套件面板搜尋 `@recommended`）
2. 在 **Workspace Recommendations** 區塊標題右側，點雲朵圖示 **Install Workspace Recommended Extensions**，一次全部安裝

**清單是空的時候**：確認檔案位於 repo 根目錄（`ls /workspaces/<repo>/.vscode/extensions.json`）、JSON 沒有格式錯誤（註解是允許的）；已安裝的擴充套件不會出現在建議清單中。

> 學員從 template 開新 codespace 時就是「開啟工作區」，提示會正常出現（右下角；若鈴鐺為勿擾模式，則收在通知中心）。只有在既有 codespace 中新增這個檔案時，才需要上面的手動步驟。

> **若你先前依 v5 建立過 `.devcontainer/`**：移除它，讓 codespace 回到預設映像的快速路徑。移除後需要再重建**最後一次**（之後就不再需要）：
> ```bash
> git rm -r .devcontainer
> git commit -m "chore: drop custom dev container in favor of workspace settings"
> git push
> ```
> 然後 `F1` → **Codespaces: Rebuild Container**；或直接刪除這台 codespace、重新開一台，通常更快。

**為什麼不用 `.devcontainer/devcontainer.json`**：

| | `devcontainer.json` | `.vscode/` 設定（**本課程採用**） |
|---|---|---|
| 生效方式 | 必須 **Rebuild Container**，耗時數分鐘 | **存檔即生效** |
| 開新 codespace 的速度 | 走「建置」路徑，較慢 | 走預設映像的快取路徑，快 |
| 儲存額度 | 客製化容器可能計入 | 預設映像不計入 |
| 擴充套件 | 自動安裝 | 跳出提示，需點一下 |
| 適用時機 | 需要客製化環境（特定語言版本、系統套件、服務） | 只需要調整編輯器行為 |

本課程只需要調整 Git 面板的行為，不需要客製化容器，所以選擇代價最低的做法。`devcontainer.json` 留作「環境即程式碼」的進階主題，並明講其代價：開機變慢、修改後必須重建。

> **Rebuild 為什麼慢**：沒有 `devcontainer.json` 時，codespace 使用事先快取在主機上的預設映像，開機只需掛載 repo、啟動容器。Rebuild 則要停止容器、依設定重新建置、重新安裝擴充套件、再重新連線。可用 `F1` → **Codespaces: View Creation Log** 查看每一步的耗時。

**驗證設定生效**：
1. 開啟 `F1` → **Preferences: Open Workspace Settings**，確認上述 `git.*` 設定出現在工作區層級。
2. 在**已有 upstream 的分支**（例如 main）上改檔並用面板 commit：狀態列出現 `↑1`，且藍色按鈕**沒有**變成 Sync Changes。
3. 若某項設定未生效（VS Code 少數設定只接受使用者層級），改到 `F1` → **Preferences: Open User Settings** 設定，並記錄在開課前檢查清單。

### 6.6 Git 操作：指令與面板兩種做法

#### 做法 A：用終端機（建議的主線）

```bash
# 基本循環
echo "## Hello" >> README.md
git status                               # 隨時確認現在在哪個狀態
git add README.md
git commit -m "docs: add greeting"
git push

# 分支與 PR
git switch -c feature/hello              # 建立並切換分支
echo "print('hi')" > hello.py
git add hello.py && git commit -m "feat: add hello.py"
git push -u origin feature/hello         # -u：記住對應的遠端分支
```

Codespace 內還預裝了 GitHub CLI，可以不離開終端機就開 PR：

```bash
gh pr create --fill          # 用 commit 訊息自動填標題與內容
gh pr view --web             # 在瀏覽器開啟這個 PR
gh pr checks                 # 看 CI 跑完了沒（本章尚未設定 CI，顯示 no checks reported 屬正常）
```

**合併 PR**：在 `gh pr view --web` 開啟的 PR 頁面按 **Merge pull request** → **Confirm merge**。合併後 main 分支上就看得到 `hello.py`（PR 的審查與合併建議用網頁操作，見 6.9）。

**為什麼建議以 CLI 為主線**：指令在桌面版、Codespaces、未來任何公司環境都一樣；網路上的文件與教學也都用指令。GUI 一改版，學員學到的就過期了。

#### 做法 B：用 Source Control 面板（GUI）

同樣一輪流程的 GUI 版本：

1. 修改檔案後，面板的 **Changes** 區會出現該檔案。
2. 點檔名可開啟**差異比對**（左舊右新），確認改動內容。
3. 在檔名右側按 **+** 移進 **Staged Changes**。
4. 在上方輸入 commit message（例如 `docs: add greeting`），按 **Commit**。
5. 狀態列按 **Sync Changes**（或新分支時的 **Publish Branch**）。

**建立分支**：點狀態列左下角的分支名稱 → **Create new branch** → 輸入 `feature/hello` → Enter。

**開 PR 的 GUI 做法**：左側的 **GitHub Pull Requests** 面板（6.5 的 `.vscode/extensions.json` 會提示安裝）可以直接建立 PR、查看審查意見、逐行回覆。

> 本課程的 GUI 操作一律在 6.5 的設定下進行：commit 不會自動 push，暫存區不會自動填入。這樣面板的每個動作都對得上一條指令。

### 6.7 衝突（conflict）怎麼處理

分組協作時一定會遇到。VS Code 預設會在檔案內直接標出衝突區塊，並提供 **Accept Current Change / Accept Incoming Change / Accept Both Changes** 的行內按鈕；需要時再按 **Resolve in Merge Editor** 開啟三欄式編輯器（上方左為 **Incoming**＝對方的版本、上方右為 **Current**＝你的版本、下方為合併結果）。初學者容易在這裡迷路，建議這樣教：

```bash
git pull                      # 出現 CONFLICT 訊息
git status                    # 看哪些檔案衝突
# 開啟檔案，直接刪掉 <<<<<<< ======= >>>>>>> 標記，保留想要的內容
git add <file>
git commit --no-edit          # 用預設的合併訊息完成合併（不會跳出編輯器）
```

先讓學員看懂**衝突標記的文字**，再介紹行內按鈕與 Merge Editor。順序反過來的話，學員會以為衝突是一種需要特殊工具才能解決的錯誤。

### 6.8 救命指令

| 情境 | 指令 |
|---|---|
| 不知道現在是什麼狀態 | `git status` |
| commit 訊息打錯（尚未 push） | `git commit --amend -m "新訊息"` |
| 想退回上一個 commit 但保留檔案 | `git reset --soft HEAD~1` |
| 檔案改爛了想還原（**未 commit 的變更會消失**） | `git restore <file>` |
| 想看歷史 | `git log --oneline --graph --all` |
| 誤刪 commit 想找回 | `git reflog` 找到 hash，再 `git switch -c rescue <hash>` |

### 6.9 💡 講師提示：版控的教學建議

| 對象 | 建議做法 |
|---|---|
| **全班主線** | 用**終端機指令**教。畫出「工作目錄 → 暫存區 → 儲存庫 → 遠端」四格圖，每個指令標在箭頭上 |
| **視覺型學員** | 補充 Source Control 面板，強調它只是同一組指令的按鈕 |
| **分組協作（PR 與 code review 練習）** | PR 的建立與審查用 **GitHub 網頁**，介面最清楚，也最接近面試官會看到的樣子 |
| **一定要示範** | `↩ Discard Changes` 的後果、`Sync Changes` 隱含 pull、以及衝突的解法 |

✅ **驗收**：用 CLI 與 GUI 各做一次「改檔 → commit → push」，並開出一個 PR 後合併；main 分支上看得到 `hello.py`。



---

## 第 7 章　VS Code 程式碼直接以容器技術執行

> **對應課綱**：教學大綱第 7 項｜**形式**：實作｜**建議時數**：3 小時

### 學習目標

- 分清楚開發迴圈（Inner Loop）與交付迴圈（Outer Loop）
- 建立聊天機器人程式，並養成「每完成一個檔案就 commit」的紀律
- 以環境變數注入組態與金鑰
- **不寫 Dockerfile**，直接用官方 Python 容器執行程式碼
- 以連接埠轉送公開服務

### 7.1 先分清楚兩個迴圈

```
【Inner Loop｜開發迴圈】第 7 章 ─────────────────────────────
  改程式 → 存檔 → 服務自動重啟（--reload）→ 看結果 → 再改
  • 容器讀的是「磁碟上的工作目錄」，與 Git 無關，所以技術上不需要 commit
  • 目標是「快」，循環以秒計
  • 風險：成果只存在這台 codespace，保留期限到了就會被刪除

           ↓ 一個段落完成了，才進入下一個迴圈 ↓

【Outer Loop｜交付迴圈】第 8～9 章 ─────────────────────────────
  commit → push → PR → CI 在「乾淨的機器」上重新 build → image 進 GHCR → 任何機器都能 run
  • 目標是「可重現、可追溯、可交付」
  • 這才是商業級流程
```

**為什麼開發時不需要先 commit**：本章掛載進容器的、以及第 8 章 `docker build` 讀取的，都是磁碟上的檔案系統；Git 管的是歷史紀錄，兩者沒有強制關係。這和你在本機開發時的行為完全相同。

**為什麼商業流程一定要 commit**：

| 理由 | 沒有 commit 的後果 |
|---|---|
| 可追溯 | 線上映像出事，查不到對應的原始碼 |
| 可重現 | 「我電腦上可以跑」—— 因為你的工作目錄有別人沒有的檔案 |
| 乾淨建置 | CI 從零 clone 再 build，才能揪出忘了加入版控的檔案 |
| 協作 | 同事看不到你的程式碼 |
| 不會消失 | **Codespace 到了保留期限會自動刪除，未 push 的內容（包含已 commit 但未 push 的 commit）一併消失** |

**本章的紀律**：每完成一個檔案就 commit 一次，在第 8 章 8.4 一次推送並開 PR，讓你親眼看到「本地有幾個 commit 等著出去」。這是**教學設計**；實務上每天收工前一定要 push——在自己的分支上 push 不會影響 main。

### 7.2 開一條工作分支

```bash
git switch main && git pull     # 先回到最新的 main（第 6 章的 PR 已合併）
git switch -c feat/ai-demo      # 所有開發都在分支上進行，main 保持可用
git status                      # 確認分支已切換、工作目錄乾淨
```

⚠️ **一定要先切回 main**：做完第 6 章後，學員通常還停在 `feature/hello`。若直接開分支，`feat/ai-demo` 會疊在 `feature/hello` 上，7.8 驗收的 commit 數就會對不上。

⚠️ **重貼的風險**：學員不確定是否成功而再貼一次時，`git switch -c` 會因分支已存在而失敗（`fatal: a branch named 'feat/ai-demo' already exists`），**人卻停在 main 上**，接下來的 commit 都會進到 main。此時改用 `git switch feat/ai-demo` 切過去即可。8.7 的 `ci/pipeline`、9.9 的 `feat/new-prompt` 同理。

**為什麼不直接在 main 上開發**：main 代表「隨時可交付的狀態」。在分支上開發，才能用 PR 做審查，也才能在改壞時直接丟掉分支。

### 7.3 先寫 `.gitignore` 與 `.dockerignore`

**這是第一個 commit，順序很重要**：先擋住不該進版控的東西，再開始寫程式。

```bash
cat > .gitignore <<'IGN'
# 機密：任何 *.env 都不准進版控
.env
*.env
*.key
# 建置雜物
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

**兩者的差別**（學員最常混淆）：

| 檔案 | 擋住誰 | 目的 |
|---|---|---|
| `.gitignore` | Git | 機密與雜物**不要進版控** |
| `.dockerignore` | `docker build` | 不需要的檔案**不要進映像**：縮小體積、避免把 `.git` 與機密打包進去 |

接著在 repo 根目錄建立以下檔案。

### 7.4 `app/llm.py` — provider 抽象層與降級策略

先建立資料夾，再用編輯器建立 `app/llm.py`：

```bash
mkdir -p app
```

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

```bash
git add app/llm.py && git commit -m "feat: add provider-agnostic LLM client with explicit fallback"
```

### 7.5 `app/main.py` — 最小 Web 服務

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

```bash
git add app/main.py && git commit -m "feat: add FastAPI endpoints with health and version checks"
```

### 7.6 相依套件與開發組態

`requirements.txt`：

```
fastapi==0.115.*
uvicorn[standard]==0.32.*
openai==1.*
```

> 商業專案會再用 lock 檔（例如 `pip-compile` 產生的完整版本清單）固定所有間接依賴，確保每次建置完全一致。教學版用 `==x.y.*` 鎖定主次版本，作為折衷。

```bash
git add requirements.txt && git commit -m "build: declare python dependencies"
```

**先把執行期設定寫成設定檔**，不再每次手打一長串 `-e`（手打正是漏設參數、默默掉進 mock 的主因）：

```bash
export MODEL="gemini-3.x-flash-lite"   # 第 4 章取得的確切 model ID；開新終端機需重設
REPO="$(git rev-parse --show-toplevel)"   # repo 根目錄的絕對路徑：不論目前在哪個子資料夾都正確
printf 'LLM_PROVIDER=google\nLLM_MODEL=%s\nLLM_MERGE_SYSTEM=auto\nLLM_API_KEY=%s\n' "$MODEL" "$LLM_API_KEY" > "$REPO/dev.env"
chmod 600 "$REPO/dev.env"

git status --short               # dev.env 不應出現 → 證明 .gitignore 的 *.env 生效
```

### 7.7 以容器直接執行程式碼（不打包映像）

到目前為止，你寫好了程式，但還沒有任何容器。這一節**不寫 Dockerfile**，直接借用官方的 Python 容器，把原始碼「掛進去」執行。

```
 【Codespace 的工作目錄】                 【官方 python:3.12-slim 容器】
  app/llm.py  app/main.py   ──掛載（-v）──▶  /srv/app/...
  requirements.txt                          pip install → uvicorn 啟動
  dev.env  ──────────────────注入（--env-file）──▶  環境變數
```

```bash
REPO="$(git rev-parse --show-toplevel)"   # repo 根目錄的絕對路徑

docker run --rm -it --name ai-dev -p 8000:8000 \
  --env-file "$REPO/dev.env" \
  -e PYTHONDONTWRITEBYTECODE=1 \
  -v "$REPO":/srv -w /srv \
  python:3.12-slim \
  sh -c "pip install --no-cache-dir -q -r requirements.txt && uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
```

> 若出現 `The container name "/ai-dev" is already in use`：代表之前啟動的容器還在（例如不小心關掉了終端機）。先執行 `docker rm -f ai-dev` 移除，再重新執行上面的指令。

逐一說明每個參數：

| 參數 | 意義 |
|---|---|
| `--rm` | 容器結束後自動刪除，不留垃圾 |
| `-it` | 互動模式：log 直接顯示在終端機，按 `Ctrl+C` 即可停止 |
| `-p 8000:8000` | 把容器的 8000 埠對應到 Codespace 的 8000 埠 |
| `--env-file` | 以絕對路徑注入組態（含金鑰） |
| `-e PYTHONDONTWRITEBYTECODE=1` | 不在你的工作目錄產生 `__pycache__` |
| `-v "$REPO":/srv` | **把原始碼掛載進容器**：容器內的 `/srv` 就是你的 repo |
| `-w /srv` | 容器內的工作目錄 |
| `python:3.12-slim` | Docker Hub 上的官方 Python 映像 |
| `sh -c "..."` | 容器啟動後：安裝套件，再啟動服務 |
| `--reload` | 偵測到程式碼變更時自動重啟服務 |

**另開一個終端機分頁**驗證：

```bash
curl -s localhost:8000/healthz       # {"status":"ok"}
curl -s localhost:8000/version       # git_sha 為 dev：這不是一個有版本身分的成品
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"用一句話解釋容器"}'
```

#### 這種做法的意義與限制

| | 本章：掛載原始碼執行 | 第 8 章：打包成映像 |
|---|---|---|
| 程式碼在哪 | **容器外**，掛載進去 | **封裝在映像裡** |
| 改程式碼 | 存檔即生效 | 要重新建置映像 |
| 每次啟動 | 都要重新 `pip install` | 套件已裝在映像裡 |
| 能不能交給別人 | **不能**：對方沒有你的原始碼 | **能**：映像就是完整成品 |
| 適合 | 開發階段（Inner Loop） | 交付階段（Outer Loop） |

已經得到的好處：**執行環境被容器統一了**。不論 Codespace 本身裝的是哪一版 Python，程式都跑在 `python:3.12-slim` 裡。但它還不是可交付的成品——這正是第 8 章要解決的事。

### 7.8 對外公開（臨時展示用）

切到 **Ports** 分頁 → 8000 會自動出現 → 點地球圖示開啟。
在 port 上按右鍵 → **Port Visibility**：
- **Private**（預設）：需登入 GitHub（類似 GCP 的 IAP）
- **Public**：任何人可存取（類似開放 `0.0.0.0/0`）

⚠️ 這個網址**綁在這台 codespace 上**，停機即失效，只適合臨時展示，**不是部署**。真正的交付在第 8～9 章。

完成後在第一個終端機按 `Ctrl+C` 停止容器（對外公開需要容器仍在執行，所以停止的步驟放在本節最後）。

✅ **驗收**：
1. 以官方 Python 容器掛載原始碼，服務正常回應，且 `/version` 的 mode 為 live
2. `git log --oneline main..HEAD` 看得到四個 commit，`dev.env` 不在其中

---

## 第 8 章　程式碼打包成容器映像

> **對應課綱**：教學大綱第 8 項｜**形式**：實作｜**建議時數**：4 小時

### 學習目標

- 撰寫多階段建置、非 root 執行的 Dockerfile
- 建置、標記並執行映像
- 以 PR 流程提交程式碼
- 以 GitHub Actions 建立 CI/CD：自動建置、測試、推送映像至 GHCR
- 以版本號發布正式版本

### 8.1 從「掛進去」到「封進去」

第 7 章的容器只提供執行環境，程式碼仍在容器外。本章把**程式碼、套件、執行環境**全部封裝成一個映像，讓它成為可以交給任何人、任何機器的成品（P1）。

### 8.2 撰寫 Dockerfile

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

```bash
git add Dockerfile && git commit -m "build: containerize app with multi-stage build"
```

### 8.3 建置並執行映像

與第 7 章最大的不同：這次**程式碼與套件都封裝在映像裡**，容器啟動時不再需要掛載原始碼，也不必重新 `pip install`。

```bash
REPO="$(git rev-parse --show-toplevel)"   # 開新終端機時需重設
docker build -t ai-demo:dev "$REPO"
docker run -d --name ai-demo -p 8000:8000 --env-file "$REPO/dev.env" ai-demo:dev
docker logs ai-demo              # 啟動訊息應顯示 mode=live
curl -s localhost:8000/healthz
curl -s localhost:8000/version   # git_sha=dev：本地建置沒有版本身分
```

> 容器啟動需要 2～3 秒，緊接在 `docker run -d` 之後的 `docker logs` 常常還是空的、`curl` 也可能回 `Connection refused`。稍等再執行一次即可，不代表失敗。
>
> 若 `docker run` 出現 `The container name "/ai-demo" is already in use`：代表之前已經啟動過，改用下方「改完程式後重跑的循環」（會先 `docker rm -f ai-demo` 再重建）。

**注意 `git_sha=dev` 這個事實**：映像雖然已經是完整的成品形式，但它是在你的開發機上、用工作目錄建置的，**包含尚未 commit 的修改**，所以它沒有、也不可能有版本身分。它仍屬於 Inner Loop；要成為可交付、可追溯的成品，必須交給 8.5 之後的 CI/CD 在乾淨環境中建置。

改完程式後重跑的循環（需要時才執行；第一次照順序練習可以跳過）：

```bash
REPO="$(git rev-parse --show-toplevel)"   # 開新終端機時需重設
docker rm -f ai-demo && docker build -t ai-demo:dev "$REPO" && \
docker run -d --name ai-demo -p 8000:8000 --env-file "$REPO/dev.env" ai-demo:dev
```

### 8.4 把開發成果推上 GitHub

```bash
git log --oneline main..HEAD     # 只列出這條分支新增的 commit：應為 5 個
git status                       # working tree clean（dev.env 被忽略，不算變更）
git push -u origin feat/ai-demo  # -u：建立 upstream，之後狀態列才會顯示 ↑N / ↓N
gh pr create --fill              # 或在 GitHub 網頁按 Compare & pull request
```

**關於狀態列的數字**：這條分支在 push 前**還沒有 upstream**，所以狀態列顯示的是雲朵圖示（Publish Branch），而不是 `↑5`——這是正常的。`main..HEAD` 這個指令才是確認「有幾個 commit 等著出去」的可靠方法；push 並建立 upstream 之後，狀態列才會開始顯示差距。

✅ **驗收**：
1. `git log --oneline main..HEAD` 看得到五個語意清楚的 commit（chore / feat / feat / build / build）
2. `git status` 顯示 working tree clean，且 `dev.env` 從未出現在任何 commit 中
3. PR 頁面看得到完整的檔案異動
4. 用手機行動網路（未登入 GitHub）能打開 Public 網址並得到回覆

> 若手機打開時要求登入 GitHub，代表 8000 埠的公開設定沒有保留（容器由 7.7 的 `ai-dev` 換成 8.3 的 `ai-demo`）：到 **Ports** 分頁把 8000 重新設為 **Public**（做法同 7.8），再試一次。

**收尾**：驗收完成後，移除在背景執行的 `ai-demo` 容器。它是用 `-d` 啟動的，不會自己停止，之後的章節也用不到：

```bash
docker rm -f ai-demo             # 強制停止並刪除容器；映像 ai-demo:dev 仍保留
```

> **commit message 慣例（Conventional Commits）**：`feat:` 新功能、`fix:` 修錯、`chore:` 雜務、`build:` 建置相關、`ci:` 流水線、`docs:` 文件。面試官看 commit 歷史就能判斷這個人有沒有受過工程訓練，**這是作品集的一部分**。

### 8.5 Outer Loop：交付流程概觀

第 7 章與 8.3 的成果只存在於你的 codespace。接下來要把它變成**任何人、任何機器都能執行的交付物**。

```
 PR 合併進 main ──▶ CI 從零 clone ──▶ 乾淨環境 build（寫入 GIT_SHA）──▶ smoke test ──▶ image 推上 GHCR
                                                                                      │
                     ┌────────────────────────────────────────────────────────────────┘
                     ▼
  任何有 Docker 的機器（空白 codespace / 同事的電腦 / PaaS）
      docker run ghcr.io/<帳號>/<repo>:<SHA>  +  部署時注入的組態（金鑰、模型）
```

四個關鍵觀念：

| 觀念 | 具體體現 | 商業意義 |
|---|---|---|
| **乾淨建置** | CI 從零 clone，不帶你的工作目錄 | 揪出「忘了 commit 的檔案」 |
| **可追溯** | 映像 tag = git SHA，且 SHA 寫進映像（`/version` 可查） | 任何線上版本都能回推到那一行程式碼 |
| **不可變** | 同一個 SHA tag 永遠是同一份內容 | 出事時能重現當時的狀況；回溯不需重新建置 |
| **組態外置** | 程式碼進映像，金鑰與模型由部署者注入 | 同一個映像跑在不同環境（12-Factor） |

### 8.6 先把開發分支的 PR 合併

交付流程的起點是 **main**，不是你的功能分支。

```bash
gh pr merge --merge --delete-branch   # 或在網頁按 Merge pull request
git switch main && git pull           # 讓本地 main 跟上遠端
```

> **合併方式的影響**：`--merge` 會產生一個 merge commit，並保留分支上原本的 commit。若改用 `--squash`，分支上的 commit 會被壓成**一個新的 commit（新的 SHA）**。無論哪種方式，**只有推進 main 的那個 commit 才會有對應的映像**——功能分支上的 SHA 不會。

### 8.7 撰寫 CI/CD：`.github/workflows/ci.yml`

```bash
git switch -c ci/pipeline             # 從最新的 main 開新分支
mkdir -p .github/workflows            # 接著建立下方的 ci.yml
```

> 若出現 `fatal: a branch named 'ci/pipeline' already exists`（重貼所致），改用 `git switch ci/pipeline`，不要在 main 上繼續（見 7.2）。

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

提交並走完整的 PR 流程：

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build, smoke test and publish image to GHCR"
git push -u origin ci/pipeline
gh pr create --fill
gh pr checks --watch && gh pr merge --merge --delete-branch   # CI 全綠才合併；PR 階段只建置與測試，不會推 image
git switch main && git pull     # 合併觸發 push 事件 → 這一次才會推 image
```

⚠️ **用 `&&` 串接 checks 與 merge**：CI 失敗時不會合併，守住品質閘門。剛建立 PR 時 GitHub Actions 可能還沒排入佇列，畫面會顯示 `no checks reported`，此時也不會合併；等 10～20 秒再執行一次即可。若分成兩行貼上，學員容易把 `no checks reported` 誤認為已完成，直接合併而跳過 CI。

**教學點：兩種身分**。CI 用的 `GITHUB_TOKEN` 是「**工作流程的身分**」（對應 GCP service account），由 GitHub 在每次執行時自動發放、執行結束即失效；第 4 章的 API 金鑰是「**人的身分**」，由你建立、由你保管。兩者的權限範圍與生命週期都不同，這是雲端權限設計的基礎。

**教學點：建置時寫入版本**。`--build-arg GIT_SHA=...` 把 commit SHA 寫進映像，smoke test 再驗證 `/version` 回報的就是這個 SHA。從此「這個容器跑的是哪一版」不需要猜。

### 8.8 讓映像可被取用

⚠️ package 要等 8.7 合併後 **main 的 CI 跑完**（約 1～3 分鐘）才會出現。可在 repo 的 **Actions** 分頁確認最新一筆已顯示綠色勾勾，再進行下面的操作。

到 GitHub 個人頁面的 **Packages** 分頁 → 選這個 package → **Package settings** → **Change visibility** → **Public**。

驗證「真的不需要登入」——先登出，避免用到 codespace 既有的憑證而得到錯誤的結論：

```bash
docker logout ghcr.io
docker pull ghcr.io/<帳號>/<repo>:latest     # 成功 → 已公開（路徑一律小寫）
```

### 8.9 發版（release）與版本語意

SHA 適合機器，人需要看得懂的版本號。workflow 已在 8.7 設定好 tag 觸發，這裡只需要打 tag：

```bash
git switch main && git pull
git tag -a v0.1.0 -m "first working demo"     # annotated tag：帶有作者、時間與說明
git push origin v0.1.0                        # 觸發 CI → GHCR 多出 v0.1.0
```

⚠️ 推送 tag 會觸發一次 CI，跑完（約 1～3 分鐘，可在 repo 的 **Actions** 分頁確認綠色勾勾）之後，Packages 頁面才會出現 `v0.1.0` 這個 tag。

**三種 tag 的分工**（商業專案的標準做法）：

| Tag | 誰在用 | 特性 |
|---|---|---|
| `<SHA>` | CI/CD、部署、回溯 | 唯一、不可變，**正式環境應該指定這個** |
| `v0.1.0` | 人、發版紀錄、對外溝通 | 有語意，對應 Git tag |
| `latest` | 開發與臨時展示 | 會移動，**正式環境不該依賴** |

✅ **驗收**：
1. PR 階段的 CI 全綠且**沒有**推送映像；合併後 GHCR 出現 `<SHA>` 與 `latest`
2. package 已設為 Public，登出後仍可 `docker pull`
3. 打 `v0.1.0` tag 後，GHCR 出現同名 tag

---

## 第 9 章　以 Codespace 空容器執行映像

> **對應課綱**：教學大綱第 9 項｜**形式**：實作｜**建議時數**：3 小時

### 學習目標

- 在一台沒有原始碼的機器上，部署指定版本的映像
- 在部署時注入組態，並驗證容器實際收到的組態
- 完成一次版本回溯（rollback）
- 對照雲端 PaaS 平台，理解它替你自動化了哪些工作

### 本章流程

把 8.7 推上 GHCR 的映像，部署到一台**完全沒有原始碼**的機器上執行。本章使用兩台 codespace：

| 代號 | 是哪一台 | 用途 | 有沒有原始碼 |
|---|---|---|---|
| **【開發機】** | 你一直在用的 repo codespace | 查詢要部署的版本號 | 有 |
| **【部署機】** | 本章新開的**空白** codespace | 拉映像、注入組態、執行 | **沒有** |

【開發機】的指令請在 repo 根目錄執行（9.9 的 `app/main.py` 是相對路徑；若不在，先 `cd /workspaces/<repo>`）。【部署機】的指令一律使用絕對路徑，在哪個資料夾執行都可以。

整體流程：

```
【開發機】取得 SHA ──────────────────────────────┐
                                                 ▼
【部署機】安裝工具（一次）→ 設定（一次）→ bash deploy.sh <SHA> → ✅
                                                 ▲
                         之後每次部署或回溯，都只需要這一行
```

> **為什麼用腳本，而不是一行一行手打？** 部署牽涉映像路徑、版本號、組態檔位置、容器何時建立等多個狀態，手動操作時任何一個沒對上都會出錯，而且錯誤往往不明顯（例如服務照常回應，其實是示範模式）。把這些步驟寫成**有驗證的腳本**，是業界的標準做法——你等一下寫的 `deploy.sh`，本質上就是一個最陽春的 PaaS。

### 9.1 【開發機】取得要部署的版本號

映像的 tag 就是 main 上的 commit SHA。先確認 CI 已經把它推上 GHCR：

```bash
git switch main && git pull
git rev-parse HEAD                     # 完整的 40 碼 SHA → 複製下來
gh run list --branch main --limit 1    # 必須是 completed / success，否則 image 還沒推上去
```

同時到 GitHub 個人頁面的 **Packages** 分頁點開這個 package，確認兩件事：
- 頁面上的 tag 清單中有這個 SHA
- 頁面上方的 `docker pull ghcr.io/...` 那一行——等一下設定時直接整行貼上即可

### 9.2 【瀏覽器】建立部署機

1. 在瀏覽器**新分頁**開啟 `https://github.com/codespaces`（「Your codespaces」頁面）。
2. 頁面上方有一排 GitHub 維護的範本。找到 **Blank**；若沒有直接顯示，點 **See all** 或 **Explore templates** 展開完整清單。
3. 在 **Blank** 下方按 **Use this template**。
4. 瀏覽器會開新分頁，1～2 分鐘後出現 VS Code 介面。

> ⚠️ **不要**在 repo 頁面再按一次 Create codespace：那樣會 clone 整個 repo，部署機上就有原始碼，實驗前提不成立。
>
> 用範本建立的 codespace 是「**unpublished**」狀態：沒有連結任何 repo。這正是我們要的。

### 9.3 【部署機】確認這是一台乾淨的機器

現在瀏覽器裡有兩個外觀幾乎相同的 VS Code 分頁。以下指令都在**部署機**執行：

```bash
pwd                             # 應為 /workspaces/ 下的空白範本資料夾，而不是你的 repo
ls -A                           # 沒有 Dockerfile、沒有 app/
docker version                  # Client 與 Server 都要出現
docker images                   # 空的：這台機器沒有任何你建置過的 image
echo "${LLM_API_KEY:-（空）}"   # 顯示「（空）」：Secret 綁定 repo，這台沒有 repo 所以拿不到
```

建議把終端機分頁改名以免混淆：在分頁標題按右鍵 → **Rename** → 輸入 `DEPLOY`。

最後一行是關鍵觀察：**部署機不會自動擁有開發機的機密，組態必須由部署者提供。**

### 9.4 【部署機】安裝部署工具（只需一次）

在 `~/deploy/` 建立兩支腳本。每一支都是**先用終端機建立並開啟檔案，再把腳本內容貼到編輯器並存檔**。

> ⚠️ 不要用 here-document（`cat > file <<'EOF'`）把整支腳本貼進終端機：實測在 Codespaces 的終端機無法順利完成，腳本又長又含特殊字元，容易貼壞。

**① 建立 `make-env.sh`**：

```bash
mkdir -p ~/deploy
code ~/deploy/make-env.sh
```

> ℹ️ 左側 Explorer 看不到 `~/deploy` 是正常的：Explorer 只顯示工作區（`/workspaces/…`），`~/deploy` 在家目錄。部署工具與金鑰刻意放在工作區之外；想在 Explorer 看到，可執行 `code -a ~/deploy` 把它加進工作區（選用）。

把下面的內容貼到剛開啟的 `make-env.sh` 分頁並存檔（`Ctrl+S`）：

```bash
#!/usr/bin/env bash
# 部署機設定：產生 deploy.conf（要部署哪個 image）與 prod.env（執行期組態）。
# 用法：bash ~/deploy/make-env.sh        ← 必須單獨執行，不可與其他指令一起貼上
# 重跑時直接按 Enter 可保留原本的值。
set -euo pipefail
DIR="$(cd "$(dirname "$0")" && pwd)"
CONF="$DIR/deploy.conf"
ENV_FILE="$DIR/prod.env"

# 清理常見的輸入錯誤：前後空白、Windows 換行、誤加的引號
clean() {
  local v="${1//$'\r'/}"
  v="${v#"${v%%[![:space:]]*}"}"
  v="${v%"${v##*[![:space:]]}"}"
  v="${v#\"}"; v="${v%\"}"; v="${v#\'}"; v="${v%\'}"
  printf '%s' "$v"
}
# 讀取既有值（若有），作為預設值
old_image=""; old_model=""; old_key=""
[[ -f "$CONF" ]]     && old_image="$(sed -n 's/^IMAGE=//p' "$CONF")"
[[ -f "$ENV_FILE" ]] && old_model="$(sed -n 's/^LLM_MODEL=//p' "$ENV_FILE")"
[[ -f "$ENV_FILE" ]] && old_key="$(sed -n 's/^LLM_API_KEY=//p' "$ENV_FILE")"

# 1. image 路徑
read -rp "image 路徑，例如 ghcr.io/帳號/repo${old_image:+ [Enter 保留：$old_image]}: " image
image="$(clean "${image:-$old_image}")"
image="${image#docker pull }"                  # 容許直接貼上 Packages 頁面的 docker pull 指令
image="$(printf '%s' "$image" | tr '[:upper:]' '[:lower:]')"
if [[ "$image" == ghcr.io/*/*:* ]]; then        # 誤帶 tag → 自動移除並提醒
  echo "ℹ️  已移除 image 路徑中的 tag「:${image##*:}」，部署時再指定版本"
  image="${image%:*}"
fi
if [[ ! "$image" =~ ^ghcr\.io/[a-z0-9._-]+/[a-z0-9._-]+$ ]]; then
  echo "❌ image 路徑格式錯誤：「$image」（應為 ghcr.io/<帳號>/<repo>）" >&2; exit 1
fi

# 2. API key
read -rsp "Google AI Studio API key（輸入時不顯示）${old_key:+ [Enter 保留原金鑰]}: " key; echo
key="$(clean "${key:-$old_key}")"
[[ -n "$key" ]] || { echo "❌ 金鑰是空的，未寫入任何檔案" >&2; exit 1; }
if [[ "$key" != AIza* || ${#key} -ne 39 ]]; then
  echo "⚠️  金鑰格式與常見格式不同（開頭 ${key:0:4}、長度 ${#key}），請確認是否貼對" >&2
fi

# 3. model ID
read -rp "model ID${old_model:+ [Enter 保留：$old_model]}: " model
model="$(clean "${model:-$old_model}")"; model="${model#models/}"
[[ -n "$model" ]] || { echo "❌ model ID 是空的，未寫入任何檔案" >&2; exit 1; }

# 原子性寫入：先寫暫存檔再改名，不會留下寫到一半的檔案
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
```

**② 建立 `deploy.sh`**：

```bash
code ~/deploy/deploy.sh
```

把下面的內容貼到剛開啟的 `deploy.sh` 分頁並存檔（`Ctrl+S`）：

```bash
#!/usr/bin/env bash
# 部署指定版本，並自動驗證。這支腳本就是一個最陽春的 PaaS：拉 image → 注入組態 → 啟動 → 健康檢查。
# 用法：bash ~/deploy/deploy.sh <SHA 或版本號>
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
# 容器實際收到的組態（而非檔案內容）
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
```

**③ 設定權限並確認**：

```bash
chmod 700 ~/deploy/make-env.sh ~/deploy/deploy.sh
ls -l ~/deploy                  # 應看到兩支腳本
```

| 腳本 | 做什麼 | 對應 PaaS 的哪個畫面 |
|---|---|---|
| `make-env.sh` | 設定要部署哪個映像、金鑰與模型，寫入 `deploy.conf` 與 `prod.env` | 服務設定頁、環境變數 / Secrets 設定 |
| `deploy.sh` | 拉取指定版本 → 重建容器 → 健康檢查 → 驗證組態與版本 | 「Deploy」按鈕與部署紀錄 |

### 9.5 【部署機】設定（只需一次）

> ⚠️ **這一行必須單獨貼上執行。** 需要互動輸入的指令若和其他指令一起貼上，貼上的下一行文字會被當成你的輸入。

```bash
bash ~/deploy/make-env.sh
```

依提示輸入三項：

| 提示 | 輸入什麼 | 說明 |
|---|---|---|
| `image 路徑` | 9.1 在 Packages 頁面看到的 `docker pull ghcr.io/...` **整行貼上** | 腳本會自動去掉 `docker pull`、tag，並轉成小寫 |
| `API key` | Google AI Studio 的金鑰：到 Google AI Studio 的 **API Keys** 頁面複製 | 部署機沒有 Codespaces Secret，必須手動貼上；輸入時畫面不顯示任何字元，這是正常的 |
| `model ID` | 第 4 章取得的 model ID | 不用加引號；多餘的引號與 `models/` 前綴會被自動清除 |

看到 `✅ 設定完成` 並列出三項摘要即可。若出現 `⚠️ 金鑰格式與常見格式不同`，請確認是否貼對，再執行一次。

> 之後要換金鑰或模型，重新執行這一行即可；**不想改的項目直接按 Enter**，會保留原值。不要用編輯器手動修改 `prod.env`——Docker 的 env 檔格式很嚴格，一個多餘的空格就會讓變數失效。

### 9.6 【部署機】部署

```bash
bash ~/deploy/deploy.sh <9.1 複製的 SHA>
```

正常的輸出：

```
▶ [1/5] 拉取 image：ghcr.io/<帳號>/<repo>:7ae9e9b1...
▶ [2/5] 移除舊容器（若有）
▶ [3/5] 以 /home/codespace/deploy/prod.env 注入組態並啟動
▶ [4/5] 等待健康檢查
▶ [5/5] 驗證版本與組態
   /version：{"git_sha":"7ae9e9b1...","provider":"google","mode":"live"}
✅ 部署完成：ghcr.io/<帳號>/<repo>:7ae9e9b1...（live）
```

五個階段各自的意義：

| 階段 | 做了什麼 | 為什麼重要 |
|---|---|---|
| 1 拉取 | 從 GHCR 下載指定版本 | **指定 SHA，而非 `latest`**，才知道部署的是什麼 |
| 2 移除 | 刪除舊容器 | env 檔只在建立容器時讀取，**必須重建**才會套用新組態 |
| 3 啟動 | 以絕對路徑注入 `prod.env` | 結果不取決於「你剛好在哪個資料夾」 |
| 4 健康檢查 | 輪詢 `/healthz` 直到服務就緒 | 容器在跑 ≠ 服務可用 |
| 5 驗證 | 檢查容器**實際收到**的組態、比對版本、實際呼叫一次模型 | 抓出「看起來正常其實是示範模式」的情況 |

想用瀏覽器看畫面：下方 **Ports** 分頁 → 8000 → 點地球圖示。

除了 SHA，也可以用 8.9 打的版本號部署：`bash ~/deploy/deploy.sh v0.1.0`。

**請讓學員停下來想三秒**：這台機器上沒有 `Dockerfile`、沒有 `app/`、沒有 `pip install`，服務卻跑起來了，而且 `/version` 明確說出它是哪一版。

- **映像是成品，repo 是原料**。交付的是成品。
- **程式碼在映像裡，組態在部署時注入**。同一個映像，換一份組態就跑在不同供應商上。
- **部署指定 SHA，而非 `latest`**。這樣才知道線上跑的是什麼。
- **這就是 PaaS 做的事**。Cloud Run、ECS、Render 的本質都是「拉映像、注入組態、跑起來、幫你管重啟與擴縮」——你剛才用 60 行腳本做了其中最核心的部分。

### 9.7 部署失敗時

`deploy.sh` 在任何一個階段失敗都會停下，並以 `❌` 說明原因：

| 錯誤訊息 | 原因 | 處理 |
|---|---|---|
| `找不到設定檔` | 尚未執行設定 | 執行 9.5 |
| `版本號格式錯誤` | SHA 複製不完整，或夾帶了其他字元 | 從 9.1 重新複製 |
| `拉取失敗` | 此版本不在 GHCR，或 package 仍是 private | 確認 Packages 頁面的 tag 清單；確認 8.8 已設為 Public |
| `服務未能啟動` | 容器啟動即退出（log 會一併顯示） | 多半是 `ConfigError`：依 log 訊息重新執行 9.5 |
| `容器收到的下列變數為空值` | 組態檔的值是空的 | 重新執行 9.5 |
| `執行中的版本與指定的 SHA 不一致` | 該 tag 對應的映像內容與預期不同 | 確認 CI 是否以正確的 SHA 建置 |
| `服務以示範模式運作（原因：google 回應 HTTP 400）` | 供應商拒絕請求；腳本會一併顯示供應商的錯誤訊息 | `API key not valid` → 金鑰錯誤；提到 model → model ID 錯誤；修正後重新執行 9.5 與 9.6 |

> 想自己檢查容器實際收到的組態（金鑰會被遮蔽）：
> ```bash
> docker inspect prod --format '{{range .Config.Env}}{{println .}}{{end}}' \
>   | grep '^LLM' | sed -E 's/^(LLM_API_KEY=.{0,4}).*/\1****/'
> ```
> **原則：檢查「容器實際收到什麼」，而不是「檔案裡寫了什麼」。**

### 9.8 保留部署機，供 9.9 使用

**先不要刪除部署機**。若中途關掉分頁：`https://github.com/codespaces` → 找到標示為 unpublished 的那一台 → 右側「**…**」→ **Open in browser**。

重新打開後**不需要重設任何變數**——所有設定都存在 `~/deploy/` 的檔案裡，直接執行 `bash ~/deploy/deploy.sh <SHA>` 即可。

> ℹ️ **（選用）手動關閉部署的容器**：`docker rm -f prod`。課程流程不需要這一步：每次 `deploy.sh` 都會先移除舊的 `prod` 再建立新的，9.9 步驟 4 刪除部署機時容器也會一起消失；codespace 停止時容器同樣會停止。

### 9.9 版本回溯（rollback）

#### 步驟 1【開發機】製造一個新版本

做一個看得出差異的小改動，走完整的 PR 流程：

```bash
git switch main && git pull
git switch -c feat/new-prompt
# 用編輯器修改 app/main.py 中 SYSTEM_PROMPT 的預設值，例如加上「回答時以『您好』開頭」
git add app/main.py && git commit -m "feat: refine default system prompt"
git push -u origin feat/new-prompt
gh pr create --fill
gh pr checks --watch && gh pr merge --merge --delete-branch   # CI 全綠才合併 → 觸發 main 的 CI → 推出新 image

git switch main && git pull
git rev-parse HEAD                     # 新版的 SHA → 複製下來
gh run list --branch main --limit 1    # 等到 completed / success 才繼續
```

> 若 `git switch -c feat/new-prompt` 出現 `already exists`（重貼所致），改用 `git switch feat/new-prompt` 後再修改 `app/main.py`（見 7.2）。

⚠️ 剛合併完時，新的 CI 可能還沒排入佇列，列出的會是**上一次**已完成的紀錄。請確認這一筆的標題含有 `feat/new-prompt`（也就是剛才的合併），而且狀態是 `completed` / `success`；不是的話，等 10～20 秒再查一次。

#### 步驟 2【部署機】部署新版

```bash
bash ~/deploy/deploy.sh <新版 SHA>
```

用瀏覽器問一個問題，回答應反映新的提示詞。

#### 步驟 3【部署機】假設新版出了問題：回溯

```bash
bash ~/deploy/deploy.sh <9.6 部署的舊版 SHA>
```

`/version` 顯示舊版的 SHA，舊版被原封不動地叫回來。

注意回溯時**沒有**重新建置、沒有 `git revert`、也沒有等待 CI——只是用另一個版本號再部署一次。

**這就是 rollback 的全部原理**：線上出事時，換回上一個 SHA 重新部署即可，不需要重新建置，也不會夾帶任何新的變更。**而這一切的前提，就是第 7 章開始養成的 commit 紀律**：沒有 commit 就沒有 SHA，沒有 SHA 就沒有可回溯的版本。

> - 只有推進 main 的 commit 才有映像。回溯的候選版本應從 **GHCR 的 tag 清單**挑選，而不是從 `git log` 隨便挑一個 SHA。
> - 回溯只是讓服務先恢復正常的緊急措施。之後仍要在開發機上修正程式碼（或 `git revert`），走完整流程發出新版，main 才會回到正確狀態。

#### 步驟 4 收工：刪除部署機

部署機是一次性的，用完就刪，避免持續占用儲存額度：

1. 開啟 `https://github.com/codespaces`
2. 找到那台 unpublished 的部署機 → 右側「**…**」→ **Delete** → 確認

刪除時會提示「unpublished 的變更將會遺失」。部署機上只有部署工具與組態，而組態（含金鑰）本來就不該被保存，直接確認即可。

### 9.10 從這裡到真實的 PaaS

| 這門課手動做的事 | PaaS 幫你做的事 |
|---|---|
| `deploy.sh`：`docker pull` + `docker run` + 健康檢查 | 自動拉取並啟動、健康檢查，掛掉自動重啟 |
| `make-env.sh` 產生 `prod.env` | 控制台或 secret 管理介面 |
| 自己開 port | 自動配網域與 TLS |
| `deploy.sh <舊 SHA>` | 一鍵 rollback |
| 一台機器 | 自動擴縮與負載平衡 |

學員在面試時能說的是：「我理解 PaaS 幫我自動化了哪些步驟，因為那些步驟我都手動做過一遍。」

✅ **驗收**：
1. CI 的 PR 階段全綠且**沒有**推映像；合併後 GHCR 出現 `<SHA>` 與 `latest`
2. 在**部署機**（Blank 範本建立、無原始碼、無 Secret）執行 `deploy.sh <SHA>`，輸出 `✅ 部署完成（live）`
3. 完成一次「新版 → 回溯舊版」，`/version` 分別顯示兩個 SHA；完成後已刪除部署機
4. 打 `v0.1.0` tag 後，GHCR 出現同名 tag

---

## 附錄 A　專案實作：大語言模型聊天機器人網站

> **對應課綱**：教學大綱第 10 項｜**形式**：專題｜**建議時數**：2 小時

### A.1 選定主題：前職領域 × AI 助手

所有學員使用相同的程式骨架，但各自選定主題，避免作品集千篇一律。建議公式是「**過去的工作領域 × AI 助手**」：

| 背景 | 主題範例 |
|---|---|
| 餐飲業 | 菜單過敏原查詢助手 |
| 會計 | 發票分類說明助手 |
| 行政 | 公文用語改寫助手 |
| 本科系 | commit message 產生器 |

只需修改三處：`SYSTEM_PROMPT`、前端文案、範例資料。

> ⚠️ 作品集 repo 是公開的，範例資料一律使用虛構或去識別化的內容。

### A.2 供應商切換與 Ollama 保底

這一節的目的是建立**完全不依賴外部供應商**的最後防線，並證明「同一個映像，只換組態就能切換供應商」。在**開發用的 codespace** 進行。

> **前提**：本機要有第 8 章 8.3 建置的 `ai-demo:dev` 映像（`docker images ai-demo` 可確認）。若中途換過 codespace，先依 8.3 重新建置。

```bash
docker rm -f ai-demo 2>/dev/null                # 釋放 8000 port

# 安裝並啟動 Ollama（模型檔約數百 MB，會佔用 Codespace 儲存額度）
curl -fsSL https://ollama.com/install.sh | sh
(ollama serve > /tmp/ollama.log 2>&1 &)   # Codespace 沒有 systemd，安裝時的 "systemd is not running" 警告屬正常
sleep 3 && ollama pull qwen2.5:0.5b

# 情境 1：本地模型。--network host 讓容器能連到 codespace 的 localhost:11434
docker run -d --name ai-local --network host -e LLM_PROVIDER=ollama ai-demo:dev
curl -s localhost:8000/version
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"用一句話解釋容器"}'
docker rm -f ai-local

# 情境 2：Groq（model ID 請先到 Groq console 確認）
docker run -d --name ai-groq -p 8000:8000 \
  -e LLM_PROVIDER=groq -e LLM_MODEL="llama-3.3-70b-versatile" -e LLM_API_KEY="$GROQ_API_KEY" ai-demo:dev
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"hi"}'
docker rm -f ai-groq

# 情境 3：沒有金鑰 → mock，且 reason 說明原因
docker run -d --name ai-mock -p 8000:8000 -e LLM_PROVIDER=google ai-demo:dev
curl -s -X POST localhost:8000/api/ask -H 'Content-Type: application/json' -d '{"prompt":"hi"}'
docker rm -f ai-mock

# 情境 4：設定錯誤 → 服務拒絕啟動（fail fast）
docker run --rm -e LLM_PROVIDER=googel ai-demo:dev   # 故意拼錯：容器應立即退出並印出 ConfigError
```

**Ollama 安裝與執行的注意事項**：

- **缺少 `zstd`**：新版 Ollama 以 `.tar.zst` 格式發布，安裝腳本在系統沒有 `zstd` 時會中止，並顯示 `ERROR: This version requires zstd for extraction`。此時先執行 `sudo apt-get update && sudo apt-get install -y zstd`，再重跑安裝指令。開課前請確認 Codespaces 預設映像是否已內建 `zstd`。
- **codespace 重新啟動後**：`ollama serve` 是背景程式，停機後即結束（模型檔會保留）。回來繼續時先重跑 `(ollama serve > /tmp/ollama.log 2>&1 &)`，否則情境 1 會回到 mock。
- **第一次回應逾時**：2-core 機器第一次回應最慢（要先載入模型）。程式的逾時為 30 秒，超過會降級為 mock，reason 顯示 `連線失敗（APITimeoutError）`；模型載入後再送一次通常即可成功。
- **`docker run -d` 後立即 `curl`**：服務啟動需要 2～3 秒，若出現 `Connection refused`，稍等再試。

**情境 3 與 4 的對比是這一節的重點**：「沒有金鑰」是可預期的執行狀態，所以降級；「provider 拼錯」是部署錯誤，所以拒絕啟動。學員要能說出為什麼兩者的處理方式不同。

⚠️ 需實測並記錄：2-core CPU 的回應時間、模型檔佔用的儲存空間。用完後 `ollama rm qwen2.5:0.5b` 釋放空間。

✅ **驗收**：同一個映像，在 `google`（第 7 章已驗證）、`groq`、`ollama`、無金鑰（mock，含 reason）四種組態下都能回應；拼錯 provider 時容器拒絕啟動。這就是「供應商鎖定風險」的現場示範。



### A.3 作品集元素

- **README**：專案簡介、架構圖、Demo GIF、環境變數說明、如何在本機執行
- **CI badge**：`![ci](https://github.com/<帳號>/<repo>/actions/workflows/ci.yml/badge.svg)`
- **Open in Codespaces 按鈕**：讓面試官一鍵開啟（用面試官自己的額度；面試官沒有你的金鑰，服務會以 mock 模式運作，畫面會標示原因）
- **GitHub Pages**：Settings → Pages，建立常駐的作品集頁
- **Profile README**：建立與帳號同名的 repo，釘選作品

---

## 附錄 B　常見卡關速查

| 症狀 | 原因 | 解法 |
|---|---|---|
| `LLM_API_KEY` 是空的 | Secret 未授權該 repo，或 codespace 未重啟 | 檢查 Repository access，Stop 後重新啟動 |
| 容器啟動後立刻退出 | `ConfigError`：provider 拼錯，或有金鑰卻沒設 `LLM_MODEL` | `docker logs <容器名>` 查看訊息 |
| 回應是「示範模式」 | 畫面上的 reason 會說明：未設金鑰 / HTTP 4xx / 連線失敗 | 依 reason 處理；HTTP 碼對照 6.3 |
| API 回 404 | model ID 寫錯或已停用 | 用 6.3 的 ListModels 重新複製 |
| API 回 429 | 觸發限流，或該模型不在免費層 | 等待；檢查是否有無窮重試；改用 Groq 或 ollama |
| API 回 400 且提到 system | 模型不支援 system 角色（Gemma 常見） | 確認 `LLM_MERGE_SYSTEM` 為 `auto`（預設）或 `1` |
| 某個 Gemini 模型呼叫失敗，但它出現在費率表上 | 費率表列出不代表可用（例如 Gemini 2.x） | 改用 Gemini 3.x Flash-Lite；以 6.3 的 ListModels 與實測為準 |
| 很快就用完當日額度 | 選到每日不到 100 次的模型 | 改用 Gemini 3.x Flash-Lite（約 500 次），或暫時改用 Gemma 4 |
| API 回 400 `API key not valid` | 金鑰錯誤（Google 對無效金鑰回 400） | 重新複製金鑰；檢查長度與開頭 |
| 改了 env 檔，結果沒變 | `docker restart` 不會重讀 env 檔 | `docker rm -f` 後重新 `docker run`（部署機請直接重跑 `deploy.sh`，它每次都會重建容器） |
| 部署機上 `deploy.sh` 報錯 | 見錯誤訊息 | 對照 **9.7** 的錯誤訊息表 |
| 互動輸入的值不對、或後面的指令沒執行 | 含 `read` 的指令和其他指令整段貼上 | 互動輸入的指令一律**單獨**貼上執行 |
| 按了 ↩ 之後程式碼不見了 | Discard Changes 會丟棄未 commit 的變更 | 無法復原；養成先 commit 的習慣 |
| Commit 之後直接出現在遠端 | 主按鈕變身為 Sync Changes、記住了 Commit & Push，或用的是 github.dev | 套用 6.5 的 `.vscode/settings.json`；確認是 Codespaces |
| `.vscode/settings.json` 的某項設定沒生效 | 少數設定只接受使用者層級 | 改到 User Settings 設定 |
| 建立 `extensions.json` 後沒有安裝提示 | 提示只在開啟工作區時出現，存檔不會觸發 | `F1` → Extensions: Show Recommended Extensions；或 Developer: Reload Window |
| 改了 `devcontainer.json` 卻沒生效（進階主題） | 設定只在建立或重建容器時套用 | `F1` → Codespaces: Rebuild Container（耗時數分鐘） |
| 開新 codespace 變慢 | repo 中有 `devcontainer.json`，走建置路徑 | 若不需要客製化容器，移除它並改用 `.vscode/` 設定 |
| 新分支 commit 後狀態列沒有 `↑N` | 分支尚未 publish，沒有 upstream | 正常現象；用 `git log --oneline main..HEAD` 確認，或先 `git push -u` |
| CI 建置失敗但本機可以跑 | 檔案忘了 commit（乾淨建置揪出來了） | `git status` 檢查未追蹤檔案，補 commit |
| CI 出現 Node.js 20 相關警告或錯誤 | 使用了綁定 Node 20 的舊版 action（如 `actions/checkout@v4`） | 升級到支援 Node 24 的版本（`actions/checkout@v5` 以上） |
| CI 推映像失敗（403 / denied） | 缺 `packages: write`，或路徑含大寫 | 補 permissions；路徑轉小寫 |
| Port 開了但打不開 | Visibility 是 Private | 右鍵改 Public |
| 無法建立 codespace | 額度用盡 | 刪除舊 codespace，或等次月重置 |
| 新帳號被要求驗證 | 防濫用機制 | 課前在家用個人網路完成註冊，**嚴禁課堂現場集體註冊** |



---

## 附錄 C　開課前檢查清單

- [ ] 開課前 2 週查看 GitHub Changelog 的 **Retired** 分類，以及 Codespaces / Actions / Packages / Pages 的計費文件
- [ ] 用**全新、未綁卡**的 GitHub 帳號跑完第 3～9 章與附錄 A
- [ ] 記錄整趟流程消耗的 Codespaces 運算時數與儲存量
- [ ] 實測 Gemini 3.x Flash-Lite、Gemma 4、Groq 的每日額度與確切 model ID，更新 4.2 的模型選擇表與附錄 D
- [ ] 確認示範帳號的 AI Studio 專案 Plan 欄為 **Free**，且未連結任何帳單帳戶
- [ ] 以 `LLM_MERGE_SYSTEM=auto` 分別用 Gemini 3.x Flash-Lite 與 Gemma 4 執行，確認兩者都能正常回應
- [ ] 確認 Gemini 2.x 是否仍無法呼叫、其他 Gemini 3.x 的每日額度是否有變動
- [ ] 實測 6.5：`.vscode/settings.json` 中每一項 `git.*` 設定在工作區層級都生效（主按鈕不會變身、commit 不會自動 push）
- [ ] 實測 template repo 開新 codespace 的時間（應維持預設快取速度），並確認擴充套件安裝提示有出現
- [ ] （進階主題備課）實測加上 `devcontainer.json` 後的開機時間與儲存用量差異
- [ ] 實測 7.7：在 Codespace 中以 `-v` 掛載 `/workspaces/<repo>` 至容器可正常運作（取決於 Codespace 的 Docker 執行方式），並確認 `--reload` 會偵測到檔案變更
- [ ] 實測 Blank 範本 codespace：`docker` 可用、工作目錄名稱、在「Your codespaces」頁面的顯示方式與刪除流程
- [ ] 實測 Ollama 在 2-core 的回應時間
- [ ] 驗證重啟 codespace 後 Public port 是否被還原
- [ ] 實測第 9 章：在全新的 Blank codespace 從 9.3 跑到 9.6，記錄所需時間；故意輸入錯誤（帶 `:latest` 的路徑、錯誤金鑰）確認腳本的提示正確
- [ ] 實測 9.9：`deploy.sh <新版>` → `deploy.sh <舊版>`，`/version` 正確
- [ ] 用第二個帳號扮演面試官，測試 Open in Codespaces + mock 模式
- [ ] 準備「當天供應商故障」的應變 runbook（切換 `LLM_PROVIDER` 與 env 檔）
- [ ] 課前 3 天寄出學員前置作業：註冊 GitHub（含 2FA）、註冊 Google AI Studio 或 Groq
- [ ] 準備投影片：Git 四格圖、Source Control 面板對照表、Inner / Outer Loop 對照圖



---

## 附錄 D　平台現況與驗證紀錄

開課前請重新驗證這張表，並更新日期。**最近一次查核：2026-10-02。****官方說明文件不等於服務現況**：GitHub Models 退役後，docs.github.com 上的相關文件仍然存在，這是本次踩到的坑。

| 服務 | 現況 | 免信用卡 | 驗證等級 | 來源優先序 |
|---|---|---|---|---|
| GitHub Free 帳號 | 可用 | ✅ | 官方文件 | pricing → docs |
| Codespaces | 可用：每月 120 core-hours（2-core 倍率 2，實際 60 小時）、15 GB-month | ✅ 無付款方式時額度用盡即封鎖，不會扣款 | 官方文件 | docs billing |
| GitHub Actions | public repo 標準 runner 免費（2026 年的計價調整未影響 public repo）。**Node 20 已於 2026-09 自 runner 移除**，JavaScript actions 需使用支援 Node 24 的版本 | ✅ | 官方 changelog | changelog → docs billing |
| GHCR（Container Registry） | public 映像儲存與流量**目前**免費；官方承諾若改變政策會至少提前一個月通知；public packages 本身即免費 | ✅ | 官方文件 | docs billing |
| GitHub Pages | Free 方案的 public repo 可用 | ✅ | 官方文件 | docs pages limits |
| **GitHub Models** | ❌ **2026-07-30 退役**（playground、目錄、API、BYOK 全部關閉） | — | 官方 changelog | **changelog 優先** |
| Google AI Studio | 免費層可用。**實測（2026-10-02）**：主力 **Gemini 3.x Flash-Lite**（約 500 次／日，速度快）；備援 Gemma 4（約 1,600 次／日，很慢）；其他 Gemini 3.x 不到 100 次／日；Gemini 2.x 雖列於費率表，實際無法呼叫 | ✅ | 講師實測 | AI Studio 控制台 > 費率表 |
| Groq | Free 方案可用 | ✅ 只需 email | ⚠️ 額度為官方樣本值，實際以 console 為準 | console.groq.com/docs/rate-limits |
| Ollama（Codespace 內） | 可用，不依賴外部政策 | ✅ | 需實測效能 | — |
| Codespaces 連接埠 Public 設定 | ⚠️ **待實測**：重新啟動 codespace 後，Public 是否會被還原為 Private（影響 7.8 的臨時展示與 8.4 驗收第 4 項） | ✅ | 待講師實測 | docs codespaces |

**驗證原則**：
1. 查「這個服務還在嗎」→ **先看 changelog 與定價頁**，再看說明文件。
2. 查「額度多少」→ 以該服務的**控制台實際顯示值**為準，文件為輔。
3. 每期開課前 2 週用**全新帳號**重跑一次本手冊的第 3～6 章。



---

## 附錄 E　版本沿革

| 版本 | 日期 | 說明 |
|---|---|---|
| 教學手冊 v1.2 | 2026-10-11 | 修正實作銜接：6.6 補上 PR 合併步驟；7.2 開分支前先切回 main；移除 7.7「體驗 `--reload`」練習（非必要，且容易留下未提交的修改，導致 8.6 切換分支失敗），停止容器的步驟移到 7.8 對外公開之後；7.4 `mkdir -p app` 移到建檔前；8.9 對第 9 章部署腳本的提前引用移至 9.6；7.1 Inner Loop 敘述改為與第 7 章「掛載原始碼、不 build」的做法一致；9.10 驗收去重；7.8 待實測項目移至附錄 D；8.7 與 9.9 改以 `gh pr checks --watch && gh pr merge` 串接，CI 未通過或尚未開始時不會合併；8.8 補上「等 main 的 CI 完成後 package 才會出現」、9.5 補上金鑰取得位置、9.9 補上確認 CI 紀錄是新合併的提示；8.3 重跑循環標示為「需要時才執行」、8.4 補上 Public 設定未保留時的處理；8.9 補上「等 tag 的 CI 跑完才看得到版本號」；第 9 章開頭補上開發機須在 repo 根目錄執行；7.2、8.7、9.9 補上重貼 `git switch -c` 時會停在 main 的說明與補救；8.3 補上 `docker run -d` 後立即 `docker logs` 可能為空的說明；7.7、8.3 補上容器名稱已被使用時的處理；9.4 兩支腳本改為「用 `code` 開啟檔案，貼到編輯器存檔」，不再用 here-document 貼進終端機，並說明 Explorer 看不到 `~/deploy` 的原因；8.4 驗收後補上移除背景容器 `ai-demo` 的收尾步驟；9.8 補上手動關閉 `prod` 容器的選用說明；附錄 A.2 補上 `ai-demo:dev` 前提、Ollama 安裝缺少 `zstd` 的處理、systemd 警告說明、重啟後重跑 `ollama serve`、首次回應逾時與啟動等待 |
| 教學手冊 v1.1 | 2026-10-02 | 依講師實測，主力模型由 Gemma 4 改為 Gemini 3.x Flash-Lite，Gemma 4 降為備援；新增 4.2 模型選擇表；`LLM_MERGE_SYSTEM` 預設改為 `auto`，依模型自動判斷 |
| 教學手冊 v1.0 | 2026-10-02 | 依課綱教學大綱 1～9 項重新編排《教學平台上手手冊 v6.0》；新增第 1、2 章觀念內容、第 6 章 Ubuntu CLI 入門、第 7 章「掛載原始碼以容器直接執行」 |
| 上手手冊 v6.0 | 2026-10-02 | 部署段落改以 `make-env.sh`、`deploy.sh` 兩支經測試的腳本承擔 |
| 上手手冊 v5.x | 2026-10-01～02 | 技術稽核版；Git 行為設定改為工作區設定；部署段落多次修正 |
| 上手手冊 v2～v4 | 2026-09-30～10-01 | GitHub Models 退役後改用 Google AI Studio；區分 Inner／Outer Loop |
