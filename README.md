🌾 農會信用部 AI 規章助手 (RAG-based AI Assistant)

本專案旨在解決農會信用部第一線櫃員面對繁瑣規章時的檢索痛點。透過 RAG (Retrieval-Augmented Generation) 架構，系統能精準定位內部 PDF 規章，並由大型語言模型 (LLM) 進行情境分流回答，確保回答有憑有據，拒絕 AI 幻覺。

🚀 專案願景

打造一個「懂法規、不瞎掰、有證據」的櫃檯數位助理，將原本需要翻閱數百頁 PDF 的時間，縮短至 5 秒內的精準對答。

🛠️ 技術架構

資料前處理 (ETL)：利用 Docling 將複雜格式 PDF 轉換為結構化 Markdown。

向量化檢索 (Retrieval)：

Embedding: 使用 BAAI/bge-m3 地端模型進行文本向量化（1024 維度）。

Vector Store: 使用 FAISS 進行高效局部向量搜尋，避免雲端資料庫版本衝突。

大語言模型 (Generation)：串接 Gemini 2.5 Flash API，透過精心設計的 System Prompt 進行「情境分流」與「來源標註」。

前端展示 (UI/UX)：使用 Streamlit 打造對話式介面，並透過 Localtunnel 進行內網穿透展示。

📋 開發修正紀錄 (Dev Log)

在開發過程中，針對環境限制與系統穩定性進行了以下優化（Troubleshooting & Workarounds）：

| 修改項目 | 修正動機 (Why) | 解決方案 (How) |
| 資料庫換手 | Colab 預設環境與 ChromaDB 產生嚴重的 opentelemetry 版本依賴衝突 (Dependency Hell)。 | 將向量庫由 ChromaDB 替換為輕量、無衝突且穩定的 FAISS-CPU。 |
| 模型版本升級 | 原 Gemini 1.5 端點回傳 404 報錯，因 Google 伺服器端點迭代更新。 | 升級程式碼至最新且反應更快的 gemini-2.5-flash 模型。 |
| 語法相容優化 | Streamlit chat_input 在判斷式中直接 = 賦值會產生 Syntax Error。 | 使用 Python 3.8+ 的 海象運算子 (:=) 實現在判斷式中同步賦值。 |
| 前端穩定性降級 | Localtunnel 傳輸動態 JS 套件不穩，導致 Markdown 程式碼區塊 (Code Block) 渲染時因抓不到語法高亮套件 (StreamlitSyntaxHighlighter.js) 而讓網頁崩潰。 | 優雅降級 (Graceful Degradation)：將法規原文展示由「程式碼區塊 (```)」改為原生的「引言格式 (> )」，成功避開外部 JS 載入需求，確保 Demo 穩定度。 |
| 防禦性編程 | API 在全球高峰期偶爾會回傳 503 (Service Unavailable) 錯誤，導致系統當機。 | 加入 Retry Mechanism (自動重試機制)，搭配 try-except 捕捉異常，提升系統在免費版 API 限制下的可靠性。 |

✨ 核心亮點功能

精準防幻覺 (Anti-Hallucination)：嚴格限制 AI 僅能依據檢索到的法規回答。若提問觸及規章未提及的邊界條件（如：阿公代簽權限），系統會踩死底線，誠實回答「未找到相關資訊」。

情境分流回答：針對如「未成年人開戶」之複雜情境，AI 能自動將條文拆解為「應備文件」、「簽署表單」等多種狀況，進行條列式呈現，大幅降低櫃員閱讀認知負擔。

透明化來源檢核 (Explainability)：每則回答均附帶法規出處路徑（麵包屑），並提供「摺疊面板」顯示原始法規 Markdown 片段，確保檢索透明化，讓第一線人員隨時可核對原文。

🖥️ 快速啟動指南 (Colab 環境)

環境初始化與還原：

pip install streamlit langchain-google-genai langchain-community sentence-transformers faiss-cpu
unzip faiss_index.zip



啟動 Web 服務與建立外網隧道：

streamlit run app.py & npx localtunnel --port 8501



存取說明：

取得 Colab 虛擬機的 Endpoint IP。

點擊 Localtunnel 生成的 URL (https://xxxx.loca.lt)。

輸入 Endpoint IP 進行身分驗證。

於網頁左側邊欄輸入你的 Google API Key，即可開始對話測試。
