![Gemini LINE Bot Labs：對話、搜尋與文件問答](docs/readme-banner.svg)

# Gemini LINE Bot Labs

以 Google Colab、Flask 與 ngrok 串接 LINE Messaging API 和 Gemini 的筆記本實驗集。從 HTTP 隧道開始，逐步比較多輪對話、Google Search 工具與上傳文件後的問答流程。

> 這些是可閱讀與修改的教學筆記本，不是已完成安全加固的正式機器人。各實驗有不同的狀態管理與簽章處理方式，請先看下方限制。

[實驗入口](#labs) · [準備與操作](#usage) · [訊息流程](#flow) · [檔案地圖](#map) · [安全與已知限制](#safety)

<a id="labs"></a>
## 實驗入口

| 筆記本 | 實際內容 | 適合觀察 |
|---|---|---|
| [0404：Flask 與 ngrok](「0404_Colab_Pyngrok_Flask_ipynb」的副本.ipynb) | `GET /` 回傳 HTML，沒有 LINE 訊息處理器 | 公開隧道如何連到 Colab 的 `5051` |
| [0702：Stateful](「0702_Colab_LINE_Bot_with_GEMINI_Stateful_ipynb」的副本.ipynb) | `gemini-2.5-flash`、共用的 `chat`、文字回覆 | `chat.send_message()` 的多輪對話 |
| [0704：Tooluse](「0704_Colab_LINE_Bot_with_GEMINI_Tooluse_copy_ipynb」的副本.ipynb) | 加入 `Tool(google_search=GoogleSearch())` | 讓模型在對話中使用 Google Search |
| [0709：Rag](「0709_Colab_LINE_Bot_with_GEMINI_Rag_ipynb」的副本.ipynb) | 以使用者 ID 保存 session，下載 LINE 附件並上傳 Gemini Files API | TXT／PDF 文件問答與一般對話切換 |

檔名中的 `Rag` 是實驗命名；目前做法是將檔案 URI 直接交給 Gemini，沒有獨立的向量資料庫、切塊或檢索索引。檔案路徑與模型名稱均沿用原始程式，不表示這些 API 或模型已重新驗證。

<a id="usage"></a>
## 準備與操作

1. 在 Colab 開啟一份筆記本。現有徽章仍指向舊名稱 `sheng13/Ai--`，請改在「開啟筆記本 → GitHub」輸入 `sheng13/gemini-line-bot-labs`，或下載後上傳。
2. 準備專用測試用 LINE channel、ngrok 與 Gemini 存取權限。只授權測試資料，並先確認各服務的帳號限制與可能費用。
3. 在 Colab Secrets 建立下列名稱；不要將值貼入筆記本、Git 或截圖：

   | Secret 名稱 | 使用範圍 |
   |---|---|
   | `NGROK_AUTHTOKEN` | 全部筆記本 |
   | `LINE_CHANNEL_ACCESS_TOKEN` | 0702／0704／0709 的 Messaging API |
   | `LINE_CHANNEL_SECRET` | 0702／0704／0709 的 webhook 簽章處理 |
   | `GEMINI_API_KEY` | 0702／0704／0709 的 Gemini client |

4. 按順序執行儲存格。LINE 實驗原始套件安裝指令如下，沒有固定版本：

   ```python
   !pip install Flask pyngrok line-bot-sdk requests --quiet
   !pip install google-genai --quiet
   ```

5. 0702／0704／0709 的隧道儲存格會透過 `requests.put()` **修改 LINE channel 的 webhook URL**。只對專用測試 channel 執行，不要覆蓋正式機器人的端點。
6. 執行 Gemini 初始化與 Flask 儲存格，保持伺服器運作。這三份 LINE 實驗接收 `POST /`，公開網址後面不要自行加上 `/callback`。
7. 在 LINE 傳送 `AI 問題`，例如 `AI 請解釋 Docker`；`AI ` 包含一個半形空格。0709 另可先傳送 `.txt`／`.pdf` 附件，再問文件內容。
8. 結束時中斷 Flask、關閉 ngrok 隧道並停止 Colab 執行階段；依測試安排恢復或停用 LINE webhook。

0702 的非 `AI ` 訊息會原樣回覆一次；0704 的非 `AI ` 分支會回覆兩則相同文字。0709 則回傳操作提示，並支援 `清除文件` 切回一般對話。

<a id="flow"></a>
## 訊息流程

```text
LINE 文字／附件
    → ngrok 公開網址
    → Flask POST / :5051
    → LINE WebhookHandler
    → AI 前綴分流／檔案處理
    → Gemini 對話、Google Search 或 Files API
    → LINE reply message
```

0709 的 `get_user_session()` 在記憶體中維護 `user_sessions`；附件先寫入 `/content/uploaded_files`，再由 `upload_file_to_gemini()` 上傳。`query_with_rag()` 有文件時送出文件 URI 與問題，沒有文件時使用該使用者的 chat。

<a id="map"></a>
## 檔案地圖

```text
.
├── README.md
├── 「0404_Colab_Pyngrok_Flask_ipynb」的副本.ipynb
├── 「0702_Colab_LINE_Bot_with_GEMINI_Stateful_ipynb」的副本.ipynb
├── 「0704_Colab_LINE_Bot_with_GEMINI_Tooluse_copy_ipynb」的副本.ipynb
├── 「0709_Colab_LINE_Bot_with_GEMINI_Rag_ipynb」的副本.ipynb
└── docs/readme-banner.svg                 設計橫幅，非聊天截圖
```

<a id="safety"></a>
## 安全與已知限制

- 0702／0704 共用同一個 `chat`，不同使用者的上下文可能混在一起；只適合單人測試。0709 按使用者分流，但 session 只在記憶體，重啟後不會保留。
- 0702 在簽章無效時仍回 HTTP 200，不能把 webhook Verify 成功視為驗證已正確完成。0704／0709 對無效簽章回 400；上線前仍需完整檢查請求驗證與錯誤處理。
- LINE 實驗會列印原始 request body，可能包含使用者 ID、訊息與附件資訊。分享筆記本前清除輸出與日誌；不要使用公司、客戶或個人敏感資料。
- 0709 的 `清除文件` 只把 session 的 `uploaded_file` 設為 `None`，不會刪除 Colab 本機檔案或 Gemini 遠端檔案。請另按服務與組織政策處理資料保留。
- 附件以原始檔名寫入共用資料夾，目前只檢查 `.txt`／`.pdf` 副檔名，沒有完整檔名隔離、大小或內容驗證。只接受自己準備的測試文件，不要對不受信任的公眾開放。
- Colab 與 ngrok 可能中斷，套件版本也未鎖定。本次只整理原始碼與文件，未呼叫 LINE／Gemini API、變更 webhook 或執行筆記本。