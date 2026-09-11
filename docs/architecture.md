# 系統架構文件 (System Architecture)

本文件描述咖啡團購系統的技術架構與設計理念。

## 🛠 技術堆疊 (Tech Stack)

### 後端 (Backend)
- **語言**: Python 3
- **框架**: Flask (Web Framework)
### 資料庫 (Database)
- **本地**: SQLite (用於開發與本地離線運行)
- **雲端**: PostgreSQL (Neon) - 用於 Vercel 等伺服器端環境
- **自動偵測**: 透過 `DATABASE_URL` 環境變數自動切換資料庫引擎。

### 前端 (Frontend)
- **模板引擎**: Jinja2 (Flask 內建)
- **樣式**: Vanilla CSS (CSS3, Flexbox, Grid)
- **互動**: Vanilla JavaScript (ES6+, Fetch API)
- **圖表**: Chart.js (用於銷售儀表板)
- **UI 元件**:
  - **手機版導覽**: 底部固定導覽列，支援快速切換。
  - **面板轉換**: 篩選器與評論視窗在手機版會自動轉為 Bottom-Sheet 式樣。

### 爬蟲 (Scraper)
- **工具**: Requests + BeautifulSoup4
- **圖片策略 (Remote-First)**: 直接儲存原始遠端 URL 到資料庫，免除本地存縮需求，確保 Vercel 雲端環境能正確顯示圖片。

### 手機版優化 (Mobile UX Strategy)
- **物理消滅 (Physical Removal)**: 為解決 Safari 渲染跳動 (Flicker)，側邊欄預設為 `display: none`。
- **二階段切換**: JavaScript 開啟側邊欄時，先設為 `block` 再觸發 CSS `.open` 動態，確保過渡平滑且無閃爍。

---

## 📂 系統模組架構

```
coffee/
├── 1_update.py / 2_server.py / 3_export.py   # [入口] 更新 / 開發伺服器 / 收單
├── run_prod.sh / run_prod.bat                # [入口] 生產伺服器 (Gunicorn / Waitress)
├── config.json          # 網站文案與 port
├── vercel.json          # Vercel 部署設定
├── src/
│   ├── app.py           # Flask 應用核心、路由與 API
│   ├── models.py        # 資料庫存取 (SQLite / Postgres 自動切換)、統計、Excel 匯出
│   ├── scraper.py       # 爬蟲邏輯 (Requests + BeautifulSoup)
│   └── google_integration.py  # Google Apps Script 送單 (選用)
├── api/index.py         # Vercel serverless 進入點，載入 src.app
├── templates/           # Jinja2 HTML 視圖
├── static/              # style.css、app.js、images/products/ (圖片快取)
├── scripts/             # 維護腳本：migrate_to_neon、reprocess_db、sync_db
├── docs/                # 本文件所在
└── data/                # 本地資料 (不進版控)
    ├── products.db      # SQLite 主資料庫
    ├── feedback.json    # 匿名回饋存檔
    ├── statistics.json  # 上期結算後的歷史統計
    └── order_*.xlsx     # 收單匯出的商家 / 內部訂購單
```

## 🔄 核心流程 (Core Flows)

### 1. 產品同步 (Update Flow)
1. 執行 `1_update.py`
2. `scraper.py` 請求官網分類頁面
3. 解析產品名稱、價格、連結
4. **圖片下載**: 檢查本地 `static/images/products/` 是否存在，若無則下載
5. **資料去重**: 透過 SKU 或名稱檢查資料庫，更新價格
6. 存入 SQLite `products` 表格

### 2. 訂購流程 (Ordering Flow)
1. 使用者訪問首頁 (`/`) 或分類頁 (`/products/<cat>`)
2. `app.py` 查詢 SQLite，渲染頁面
3. 使用者加入購物車 -> 存入 Session (`session['cart']`)
4. 結帳頁面 (`/cart`) 輸入姓名
5. 提交訂單 (`POST /api/orders`)
   - 建立 `Order` 紀錄
   - 建立多筆 `OrderItem` 紀錄
   - 清空購物車
   - **更新即時統計**: `get_current_sales_statistics` 隨之變動

### 3. 商品評論流程 (Review Flow)
1. 使用者在商品卡片點擊「撰寫評論」。
2. 前端彈出 Modal 或 Bottom-Sheet 輸入 1-5 星評分與留言。
3. 提交至 `POST /api/reviews/<product_id>`。
4. 儲存於資料庫，前端即時更新該商品的平均星數與評論數量。

### 3. 即時儀表板 (Dashboard Flow)
1. 訪問 `/orders`
2. 後端 `get_current_sales_statistics()`:
   - 撈取所有 `orders` 與 `order_items`
   - 計算 Top 10 熱銷商品
   - 統計「烘焙度」分佈 (Pie Chart)
   - 統計「價格帶」銷量 (Bar Chart)
3. 前端透過 Chart.js 渲染圖表，若無數據則顯示空狀態動畫

### 4. 結算匯出 (Export Flow)
1. 執行 `3_export.py`
2. `export_orders_to_xlsx`:
   - **商家訂購單**: 按產品匯總數量 (Group by Product)
   - **內部分發單**: 按人員分組 (Group by Customer)，列出每人明細與總額
3. 使用 `openpyxl` 產生 Excel 檔案並套用樣式
4. 更新 `data/statistics.json` 做為歷史紀錄
5. 清空 `orders` 資料表，並在 `settings` 表寫入 `last_export_at`
6. 之後 `1_update.py` 新爬進來的商品 (`created_at` > `last_export_at`) 就會出現在「本次新品」
