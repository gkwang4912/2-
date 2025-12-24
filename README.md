# 多模態影片分析與 RAG 檢索系統

這是一個整合電腦視覺 (CV) 與自然語言處理 (NLP) 的多模態 RAG (Retrieval-Augmented Generation) 系統。
專案將影片處理流程模組化，從影片中自動提取關鍵影格 (Keyframes)、場景資訊、語音逐字稿 (Transcript) 以及對話截圖，並將這些多模態資訊建立成向量資料庫。
最終，使用者可以透過 Web 介面以自然語言提問，系統會檢索相關的對話與畫面，並結合大型語言模型 (LLM) 生成回答。

---

## 🛠️ 環境建置 (Prerequisites)

在開始之前，請確保您的系統已安裝以下工具：

1.  **Python 3.8+**: 建議使用 Anaconda 或 Miniconda 管理環境。
2.  **FFmpeg**: 必須安裝並加入系統環境變數 (PATH)，用於影片處理。
3.  **LM Studio**: 本專案的問答功能依賴本地運行的 LLM (如 `Qwen2.5-VL` 或其他支援 OpenAI API 格式的模型)。
    *   請下載並安裝 LM Studio。
    *   啟動 Local Server，設定 Port 為 `1234` (預設值)。
    *   確認 API URL 為 `http://127.0.0.1:1234/v1`。

### 安裝專案依賴

建議為此專案建立獨立的虛擬環境。請在專案根目錄下執行：

```bash
# 安裝所有模組需要的套件 (建議依序安裝各資料夾內的 requirements.txt，或參考以下整合指令)
pip install openai-whisper ultralytics opencv-python pandas numpy tqdm flask faiss-cpu transformers torch pillow ffmpeg-python
```

*注意：若您有 NVIDIA 顯卡，請安裝對應 CUDA 版本的 `torch` 和 `faiss-gpu` 以獲得更佳效能。*

---

## 🚀 詳細使用流程 (Step-by-Step Guide)

請依照資料夾編號順序執行。為了避免路徑問題，建議將您的影片檔案命名為 `test.mp4` 並依序複製到各工作目錄中。

### 步驟 1: 關鍵影格與物件偵測
**位置**: `1_關鍵偵擷取/`

此步驟分析影片的場景切換與物件，並抓取關鍵影格。

1.  將您的影片 (例如 `test.mp4`) 放入此資料夾。
2.  執行分析程式：
    ```bash
    cd 1_關鍵偵擷取
    python analyze.py test.mp4
    ```
3.  **產出結果**:
    *   `keyframes/` 資料夾：包含所有場景的關鍵影格圖片。
    *   `test_analysis.csv`：包含場景與物件偵測的時間點紀錄。

### 步驟 2: 語音逐字稿擷取
**位置**: `2_逐字稿擷取/`

此步驟使用 OpenAI Whisper 將影片語音轉為文字。

1.  確保影片 `test.mp4` 位於此資料夾 (或從上一步複製過來)。
2.  執行轉錄程式：
    ```bash
    cd ../2_逐字稿擷取
    python transcribe.py
    ```
    *(備註: 程式預設讀取 `test.mp4`，若為其他檔名請修改程式碼)*
3.  **產出結果**:
    *   `transcript.csv`：包含 `開始時間`, `結束時間`, `內容` 的初步逐字稿。

### 步驟 3: 圖文對齊與截圖
**位置**: `3_逐字稿圖片擷取/`

此步驟根據逐字稿的時間戳記，精確擷取每句對話的開始與結束畫面。

1.  確保以下檔案位於此資料夾：
    *   影片 `test.mp4`
    *   上一步驟產生的 `transcript.csv`
2.  執行截圖程式：
    ```bash
    cd ../3_逐字稿圖片擷取
    python extract_screenshots.py
    ```
3.  **產出結果**:
    *   `screenshots/` 資料夾：包含對話對應的截圖。
    *   `transcript.csv` (已更新)：新增了 `開始照片檔名` 與 `結束照片檔名` 欄位。

### 步驟 4: 準備 RAG 資料庫素材
**位置**: `5_RAG_database/`

在建立資料庫前，我們需要將前面步驟產生的所有素材彙整到 `input` 資料夾。

1.  進入 `5_RAG_database` 資料夾。
2.  建立一個名為 `input` 的資料夾 (如果不存在)。
3.  **將以下檔案複製到 `input/` 中**：
    *   步驟 1 的 `keyframes/` 資料夾內所有圖片 (建議分類或直接放入)。
    *   步驟 3 的 `screenshots/` 資料夾內所有圖片。
    *   步驟 3 更新後的 `transcript.csv`。
    *   (選用) 步驟 1 的 `test_analysis.csv`。

資料夾結構示意：
```
5_RAG_database/
└── input/
    ├── transcript.csv
    ├── img_1_start.jpg ... (來自 screenshots)
    └── frame_0001_Scene_Cut.jpg ... (來自 keyframes)
```

### 步驟 5: 建立 RAG 資料庫 (Ingestion)
**位置**: `5_RAG_database/`

此步驟將 `input` 資料夾內的圖文資料轉換為向量並存入資料庫。

1.  執行資料庫建置程式：
    ```bash
    python rag_ingest.py
    ```
2.  程式會自動讀取 `input` 內的 CSV 與圖片，使用 CLIP 模型進行 Embedding。
3.  **產出結果** (勿刪除)：
    *   `rag_mm.db`: SQLite Metadata 資料庫。
    *   `text.index`, `image.index`: FAISS 向量索引檔。

---

## 💻 啟動查詢系統 (Web UI)

完成資料庫建置後，即可啟動網頁介面進行對話。

1.  **確保 LM Studio 已啟動** 並正在運行 Server (預設 Port 1234)。
2.  在 `5_RAG_database/` 資料夾下執行：
    ```bash
    python server.py
    ```
3.  打開瀏覽器，前往 `http://localhost:5000`。
4.  在對話框輸入您的問題 (例如：「影片中提到關於...的內容是什麼？」或「幫我找出現太空人的畫面」)。系統將會檢索最相關的圖片與文字並回答您。

---

## 📂 專案結構說明

*   `1_關鍵偵擷取`: 視覺分析核心 (TransNetV2 + YOLOv8)。
*   `2_逐字稿擷取`: 聽覺分析核心 (Whisper)。
*   `3_逐字稿圖片擷取`: 多模態對齊核心 (Time-based Screenshot)。
*   `5_RAG_database`: 系統整合核心 (CLIP Embedding + FAISS + LLM Query)。

## ❓ 常見問題 (Troubleshooting)

*   **FileNotFoundError (WinError 2)**: 通常是電腦未安裝 FFmpeg 或未加入 PATH。
*   **CUDA / GPU Errors**: 若無 NVIDIA 顯卡，請確保 `transcribe.py` 與 `requirements.txt` 中使用的是 CPU 版本套件 (如 `faiss-cpu`)，且程式碼中 `fp16=False` (Whisper 設定)。
*   **Connection Error (LM Studio)**: 確保 LM Studio 的 Local Server 功能已開啟，且 Port 為 1234。
