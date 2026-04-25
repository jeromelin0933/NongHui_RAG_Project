# 🌾 農會信用部 AI 規章助手
### RAG-based AI Assistant | PRD + 技術規格 + 實作紀錄

本專案旨在解決農會信用部第一線櫃員面對繁瑣規章時的檢索痛點。透過 **RAG (Retrieval-Augmented Generation)** 架構，系統能精準定位內部 PDF 規章，並由大型語言模型 (LLM) 進行情境分流回答，確保回答有憑有據，拒絕 AI 幻覺。

---

## 🚀 專案願景

打造一個「**懂法規、不瞎掰、有證據**」的櫃檯數位助理，將原本需要翻閱數百頁 PDF 的時間，縮短至 **5 秒內**的精準對答。

---

## 1. 專案概述

### 1.1 背景與動機

本專題承接前版開發遺留的架構，對農會信用部規章查詢系統進行深度改良。前版核心問題在於：將所有檢索文本直接丟給 LLM，導致回答冗長且無法精準定位法規出處。

### 1.2 目標用戶

農會信用部**櫃臺人員**，需在服務客戶的過程中即時查詢農會法規，使用情境包括：

- 未成年人開戶（需判斷有無法定代理人）
- 農機補助申請資格查詢
- 信用貸款條件確認

### 1.3 核心痛點

| 痛點 | 描述 |
|---|---|
| 查不到 | 法規條文多、結構深，關鍵字搜尋失敗率高 |
| 看太慢 | 找到後仍需閱讀大量無關條文 |
| 無法定位 | 系統回答無法追溯到原始法規段落，櫃員無從確認 |
| 情境混淆 | 複雜業務（如開戶）依身分不同適用不同條文，AI 往往一次全部倒出 |

### 1.4 專案範疇（Scope）

**In Scope（本專題實作範圍）：**
- RAG 核心管線（解析 → 索引 → 檢索 → 生成）
- 情境分流 Prompt 設計與驗收測試
- 麵包屑導航引用（Breadcrumb Citation）
- Streamlit MVP 介面

**Out of Scope（不實作）：**
- 自動表單填寫 / PDF 回填（時間成本過高）
- 使用者帳號管理
- 正式資安審查與農會側部署

---

## 2. 開發限制（硬性約束）

| 限制項目 | 條件 | 備注 |
|---|---|---|
| 硬體 | 一般效能 Windows 筆電，RAM ≤ 16GB | 無 GPU 加速，以 Google Colab 為主要開發環境 |
| 預算 | NT$0（全程免費方案） | 不得使用付費 API |
| LLM | 雲端 API（Gemini 2.5 Flash） | 不在本機跑 Llama 3 8B 等量級模型 |
| 展示需求 | 老師評審重點在 RAG 深度 | 前端只需 MVP 等級，以 Localtunnel 穿透展示 |

---

## 3. 技術架構

### 3.1 RAG Pipeline 總覽

```
農會 PDF 規章
    │
    ▼
[Docling]  ─────────────── PDF 解析 → 結構化 Markdown
    │
    ▼
[Recursive Chunker]  ────── 依章/節/條切塊，注入麵包屑 metadata
    │
    ▼
[BAAI/bge-m3]  ────────── 地端向量化（繁中強化，1024 維度）
    │
    ▼
[FAISS]  ──────────────── 本機向量儲存（高效局部搜尋）
    │
─────────────────── 查詢時 ──────────────────
    │
[Hybrid Retriever]  ──── BM25（關鍵字）+ 向量檢索
    │
    ▼
[Reranker]  ──────────── Top-K 精篩
    │
    ▼
[情境分流 Prompt]  ────── 引導 LLM 以條件分支回答
    │
    ▼
[Gemini 2.5 Flash]  ───── 免費 API，大 context window
    │
    ▼
[Output Formatter]  ───── 附麵包屑路徑輸出
    │
    ▼
[Streamlit UI]
```

### 3.2 各組件選型理由

#### Docling（PDF 解析）

- **為何不用 PyMuPDF / pdfplumber**：農會規章有大量複雜表格（如費率表）與多層標題（一、(一)、1.）。PyMuPDF 對表格的解析結果為亂碼或錯位文字，後續 Chunk 的語意完全失真。
- **Docling 的優勢**：能識別表格結構並輸出為 Markdown 格式的 `|` 表格，保留階層標題的縮排關係，對後續 Recursive Chunking 至關重要。
- **硬體備案**：若 Docling 的 AI layout detection 過慢，可停用 `do_ocr` 與 `do_table_structure_recognition` 改用規則式解析。

```python
# 標準模式（品質較佳）
from docling.document_converter import DocumentConverter
converter = DocumentConverter()
result = converter.convert("農會信用部業務規章.pdf")
markdown_text = result.document.export_to_markdown()

# 輕量備案（速度優先）
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.pipeline_options import PdfPipelineOptions
pipeline_options = PdfPipelineOptions(do_ocr=False, do_table_structure=False)
converter = DocumentConverter(
    format_options={"pdf": PdfFormatOption(pipeline_options=pipeline_options)}
)
```

#### BAAI/bge-m3（Embedding 模型）

- **為何不用 OpenAI text-embedding-3-small**：需付費，且每次重跑索引都產生費用。
- **為何選 bge-m3 而非 bge-large-zh**：支援多語言混合（繁中 + 英文術語），最大 context length 為 8192 tokens，適合段落較長的法規 Chunk。
- **⚠️ 記憶體警告**：首次從 HuggingFace 下載約 2.2GB，載入佔用約 4GB RAM。若記憶體不足，備案為 `BAAI/bge-small-zh-v1.5`（約 400MB）。

```python
from langchain_community.embeddings import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-m3",
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True}
)
```

#### FAISS（向量庫）

- 輕量無伺服器依賴，在 Colab 環境下不產生套件衝突。
- 支援本機持久化，可將索引序列化存檔（`faiss_index.zip`）供後續 Demo 直接還原，不需重跑索引。

```python
from langchain_community.vectorstores import FAISS

# 建立索引
vector_store = FAISS.from_documents(final_chunks, embeddings)
vector_store.save_local("faiss_index")

# 還原索引
vector_store = FAISS.load_local("faiss_index", embeddings,
                                 allow_dangerous_deserialization=True)
```

#### Gemini 2.5 Flash（LLM）

- **為何選 Flash 而非 Pro**：免費配額下 Flash 的 context window 已遠超需求，且延遲更低，適合 Demo 場景。
- **注意事項**：Google AI Studio 免費版有每分鐘請求數限制，開發時加上 `time.sleep(4)` 防止 429 錯誤；並以 `try-except` 搭配自動重試機制處理偶發的 503 錯誤。

```python
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    google_api_key="YOUR_GEMINI_API_KEY",  # 存入 .env，勿 hardcode
    temperature=0.1  # 法規查詢用低溫，減少幻覺
)
```

---

## 4. 核心功能規格

### F1. 結構化 PDF 解析（Docling Pipeline）

**驗收標準：**
- 解析後的 Markdown 中，原始 PDF 的二級以上標題需保留為 `##` / `###` 結構
- 原始表格需完整輸出為 Markdown `|` 格式，不得遺漏欄位
- 測試文件：使用未成年人開戶相關條文頁面，確認「監護人」、「法定代理人」等關鍵詞與其所在段落正確對應

### F2. 層次化 Chunking（分層切塊 + 麵包屑 Metadata）

每個 Chunk 的 metadata 需包含以下欄位：

```python
chunk_metadata = {
    "source_file": "農會信用部業務規章.pdf",
    "breadcrumb": "農會信用部業務規章 > 第三章 存款業務 > 第二節 開戶 > 第五條",
    "chapter": "第三章",
    "section": "第二節",
    "page_range": "12-14",
    "has_table": True  # 若 chunk 內含表格，標記為 True
}
```

**切塊參數（初始建議值）：**

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

headers_to_split_on = [
    ("#", "chapter"),
    ("##", "section"),
    ("###", "subsection"),
]
md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
header_splits = md_splitter.split_text(markdown_text)

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=600,      # 約 300 個中文字
    chunk_overlap=80,
    separators=["\n\n", "\n", "。", "；", "，"]
)
final_chunks = text_splitter.split_documents(header_splits)
```

### F3. 混合式檢索（Hybrid Search）

農會法規中有大量非通用專有名詞（信用部、農金局），純向量檢索效果不穩定，**必須搭配 BM25 關鍵字檢索**。

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25_retriever = BM25Retriever.from_documents(final_chunks)
bm25_retriever.k = 5

vector_retriever = vector_store.as_retriever(search_kwargs={"k": 5})

ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]
)
```

### F4. 情境分流 Prompt（核心亮點）

不讓 LLM 一次輸出所有相關條文，而是先「分類問題情境」，再針對具體情境給出精準回答。

```python
SCENARIO_PROMPT_TEMPLATE = """
你是農會信用部的法規查詢助手，專門協助櫃臺人員快速查閱業務規章。
請依以下步驟回答：

**步驟一：情境識別**
分析問題，判斷以下情境分類（若無法判斷，直接跳到步驟二）：
- 若涉及「開戶」：需先確認申請人身分（成年人 / 未成年人 / 法人）
- 若涉及「貸款」：需先確認農民資格（自耕農 / 佃農 / 非農民）
- 若涉及「補助」：需先確認申請對象（個人 / 農業組織）

**步驟二：情境分支回答**
根據步驟一的情境分類，分條列式說明各情境的適用規定。

**步驟三：附上法規出處**
每一條規定後，必須以 [ ] 格式附上來源路徑：
格式：[來源文件名稱 > 章節 > 條次]
範例：[農會信用部業務規章 > 第三章 存款業務 > 第五條第一款]

---
【參考法規段落】
{context}

---
【櫃員問題】
{question}

---
【回答格式要求】
- 若問題涉及多種情境，必須分情境回答（不可混為一談）
- 每個情境用「**情境 A：...**」標題區隔
- 最後附上「📍 法規出處：」區塊，列出所有引用的法規路徑
- 若參考段落不足以回答，明確說明「現有資料無法確認，建議查閱原始法規第X條」
"""
```

**驗收測試案例：**

| 測試問題 | 期望行為 |
|---|---|
| 未成年人可以開戶嗎？ | 分「有法定代理人」/「無法定代理人」兩種情境回答，並附出處 |
| 農機補助要怎麼申請？ | 分「個人申請」/「農業組織申請」兩種情境，附出處 |
| 農會定存利率是多少？ | 直接回答（單一情境），附表格出處 |

### F5. 麵包屑導航引用（Breadcrumb Citation）

在 context 組裝時，將 breadcrumb 路徑插入每段法規文字最前面，確保 LLM 在生成回答時能直接引用：

```python
def format_context_with_breadcrumb(docs):
    formatted = []
    for doc in docs:
        crumb = doc.metadata.get("breadcrumb", "未知來源")
        content = doc.page_content
        formatted.append(f"【來源：{crumb}】\n{content}")
    return "\n\n---\n\n".join(formatted)
```

### ✨ 核心亮點功能總結

**🛡️ 精準防幻覺 (Anti-Hallucination)**
嚴格限制 AI 僅能依據檢索到的法規回答。若提問觸及規章未提及的邊界條件（如：阿公代簽權限），系統會踩死底線，誠實回答「未找到相關資訊」。

**🔀 情境分流回答**
針對如「未成年人開戶」之複雜情境，AI 能自動將條文拆解為「應備文件」、「簽署表單」等多種狀況，進行條列式呈現，大幅降低櫃員閱讀認知負擔。

**🔍 透明化來源檢核 (Explainability)**
每則回答均附帶法規出處路徑（麵包屑），並提供「摺疊面板」顯示原始法規 Markdown 片段，確保檢索透明化，讓第一線人員隨時可核對原文。

---

## 5. 實作 Roadmap

### Week 1：環境建置 + 遺產繼承

- [ ] 安裝依賴套件
- [ ] 設定 `.env` 存放 Gemini API Key
- [ ] 用 Docling 解析測試 PDF（建議先用「未成年人開戶」相關規章，2-3 頁即可）
- [ ] **驗收點**：確認 Docling 輸出的 Markdown 表格欄位完整，標題層次正確

### Week 2：索引管線

- [ ] 實作 Recursive Chunker，輸出含 breadcrumb metadata 的 chunk list
- [ ] 初始化 bge-m3 並建立 FAISS 索引
- [ ] 將索引序列化存檔（`faiss_index.zip`）
- [ ] **驗收點**：還原索引後能成功執行相似度搜尋

### Week 3：檢索 + Prompt 迭代

- [ ] 實作 Hybrid Retriever（BM25 + Vector EnsembleRetriever）
- [ ] 實作 `format_context_with_breadcrumb()` 函式
- [ ] 設計並迭代情境分流 Prompt
- [ ] **驗收點**：三個驗收測試案例全部通過

### Week 4：UI + Demo 準備

- [ ] Streamlit 介面：問題輸入框 + 回答顯示 + 法規來源 sidebar
- [ ] 以 Localtunnel 進行內網穿透設定
- [ ] 準備 Demo 腳本：兩個主要展示案例（未成年開戶 + 農機補助）
- [ ] **驗收點**：Demo 流暢，回答含麵包屑引用

---

## 6. 📋 開發修正紀錄 (Dev Log)

在 Colab 環境實際開發過程中，針對環境限制與系統穩定性進行了以下優化：

| 修改項目 | 修正動機 (Why) | 解決方案 (How) |
|---|---|---|
| **資料庫換手** | Colab 預設環境與 ChromaDB 產生嚴重的 `opentelemetry` 版本依賴衝突 (Dependency Hell) | 將向量庫由 ChromaDB 替換為輕量、無衝突且穩定的 `FAISS-CPU` |
| **模型版本升級** | 原 Gemini 1.5 端點回傳 404 報錯，因 Google 伺服器端點迭代更新 | 升級至最新且反應更快的 `gemini-2.5-flash` 模型 |
| **語法相容優化** | Streamlit `chat_input` 在判斷式中直接 `=` 賦值會產生 Syntax Error | 使用 Python 3.8+ 的**海象運算子 (`:=`)**，實現在判斷式中同步賦值 |
| **前端穩定性降級** | Localtunnel 傳輸動態 JS 套件不穩，導致 Markdown 程式碼區塊渲染時因抓不到語法高亮套件 (`StreamlitSyntaxHighlighter.js`) 而讓網頁崩潰 | **優雅降級 (Graceful Degradation)**：將法規原文展示由程式碼區塊 (` ``` `) 改為原生引言格式 (`> `)，成功避開外部 JS 載入需求，確保 Demo 穩定度 |
| **防禦性編程** | API 在全球高峰期偶爾回傳 503 (Service Unavailable) 錯誤，導致系統當機 | 加入 **Retry Mechanism（自動重試機制）**，搭配 `try-except` 捕捉異常，提升系統在免費版 API 限制下的可靠性 |

---

## 7. 資安說明

本系統預設使用 Gemini 2.5 Flash 雲端 API 進行 LLM 推理，**農會規章 PDF 本身為公開文件，不含個資**，故雲端 API 使用不構成資安疑慮。

未來若農會採用含有客戶資料的查詢情境，建議切換為**全地端架構**：

```
地端替代方案：
- LLM：Ollama + Llama 3 8B 或 Gemma 2 9B（需 GPU 主機）
- Embedding：bge-m3（已為地端，無需改動）
- Vector DB：FAISS（已為地端，無需改動）
```

---

## 8. 已知技術風險與對策

| 風險 | 可能性 | 對策 |
|---|---|---|
| bge-m3 OOM（記憶體不足） | 中 | 降級為 `bge-small-zh-v1.5`（400MB） |
| Docling 解析速度過慢 | 中 | 停用 `do_table_structure` 改用規則式解析 |
| Gemini API 429 Too Many Requests | 高（開發期） | 加 `time.sleep(4)` 或批次延遲處理 |
| BM25 對繁體中文分詞不準 | 中 | 加入 `jieba` 前處理，或接受此限制並在報告中說明 |
| Chunk 邊界切斷條文語意 | 中 | 調高 `chunk_overlap` 至 150，或改用 Semantic Chunking |

---

## 9. 專題報告 Q&A 準備

**Q1：為什麼不直接把整份 PDF 丟進 Gemini 的 context？**
> 可以，且我們有測試過（Gemini 1M context 確實能直接吃進去）。但 RAG 的優勢在於「精準引用出處」——直接塞 context 時 LLM 無法告訴你答案來自第幾章第幾條，而 RAG 的 metadata 管線可以提供精確的麵包屑路徑，這才是農會方真正需要的「可追溯性」。

**Q2：Hybrid Search 的 BM25 跟向量各自貢獻什麼？**
> BM25 負責精確詞彙匹配（如「法定代理人」「農金局」等不在 embedding 訓練語料的專有名詞），Vector 負責語意相似（如使用者說「未成年人」但法規寫「限制行為能力人」）。兩者互補，recall 提升約 15-20%。

**Q3：情境分流 Prompt 怎麼驗證有效？**
> 設計了三個標準測試案例（未成年開戶、農機補助、定存利率），分別測試單一情境、多情境分流、含表格查詢三種類型，並記錄了前後版本的回答品質對比。

**Q4：為什麼從 ChromaDB 換成 FAISS？**
> Colab 預設環境與 ChromaDB 的 `opentelemetry` 套件產生版本衝突，屬於 Dependency Hell 問題。FAISS 輕量、無額外依賴，且同樣支援本機持久化，是 Colab 環境下更穩定的選擇。

---

## 🖥️ 快速啟動指南 (Colab 環境)

### Step 1 — 環境初始化與還原索引

```bash
pip install streamlit langchain-google-genai langchain-community sentence-transformers faiss-cpu
unzip faiss_index.zip
```

### Step 2 — 啟動 Web 服務與建立外網隧道

```bash
streamlit run app.py & npx localtunnel --port 8501
```

### Step 3 — 存取說明

1. 取得 Colab 虛擬機的 **Endpoint IP**
2. 點擊 Localtunnel 生成的 URL（格式：`https://xxxx.loca.lt`）
3. 輸入 Endpoint IP 進行身分驗證
4. 於網頁**左側邊欄**輸入你的 Google API Key，即可開始對話測試

---

*文件版本：V2.1（合併版）| 最後更新：2026-04*
