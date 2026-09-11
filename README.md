# 咖啡團購系統 (Coffee Group Buy System) ☕

> [!CAUTION]
> **免責聲明 (Disclaimer)**
>
> 本專案僅供程式開發學習與內部教育用途，**非官方系統**。
> 本系統之爬蟲功能僅用於示範資料抓取技術，所有抓取之產品圖片、文案與商標權利均歸屬 **原品牌商 (Oklao 歐客佬)** 所有。
> 請勿將此系統部署於公開網路或用於商業營利行為。使用者應自行承擔使用本軟體之所有風險。

一個用於自動化部門同事之間咖啡團購流程的 Flask Web 應用程式：爬取商品 → 同事線上下單 → 匯出商家訂購單與內部分發單。

## ✨ 主要功能

| 功能 | 說明 |
|------|------|
| 📝 訂單與購物車 | 姓名查詢、修改數量、刪除訂單；已加入購物車的商品會標示，避免重複購買 |
| 📊 即時銷售看板 | 首頁顯示本期熱銷 Top 5 與烘焙度、價格帶分佈圖表 |
| ⭐ 本次新品 | 自動標示最近一次爬蟲更新的新商品，支援價格與烘焙度篩選 |
| 💬 商品評論 | 1–5 星評分與文字留言，手機版為底部滑出面板 |
| 📮 匿名回饋 | 頁尾「匿名回饋」可直接把建議送給開發者 |
| ☁️ 混合資料庫 | 本地用 SQLite；設定 `DATABASE_URL` 後自動切換為 Postgres (Neon)，可直接部署到 Vercel |
| 📱 手機優化 | 底部導覽列、44px 觸控目標、消除 Safari 重新整理閃爍 |

---

## 🚀 快速開始

### 0️⃣ 安裝

```bash
git clone https://github.com/hans0803/CoffeeGroupBuy.git
cd CoffeeGroupBuy
pip install -r requirements.txt
cp .env.example .env   # 本地 SQLite 可留空；雲端請填 DATABASE_URL
```

之後若有更新，執行 `git pull` 拉取最新程式碼。

### 1️⃣ 更新商品 — 執行爬蟲

```bash
python 1_update.py
```

從來源網站爬取最新產品並寫入資料庫。圖片會快取到 `static/images/products/`，資料庫則保存遠端圖片網址，雲端部署不需另外存圖。

### 2️⃣ 啟動服務

| 模式 | 指令 | 適用 |
|------|------|------|
| 開發 | `python 2_server.py` | 單人測試，port 5001 |
| 生產 (macOS / Linux) | `sh run_prod.sh` | Gunicorn 4 workers，多人連線 |
| 生產 (Windows) | `run_prod.bat` | Waitress，多人連線 |

生產模式的 port 由 `config.json` 的 `server.port` 決定。同事可透過 `http://<你的IP>:<port>` 連線：

- macOS 查 IP：`ipconfig getifaddr en0`
- Windows 查 IP：`ipconfig`

### 3️⃣ 收單 — 匯出並清空

```bash
python 3_export.py
```

產生商家訂購單與內部分發單 (`data/order_*.xlsx`)，累積歷史銷量，並可選擇清空當期訂單。

### 4️⃣ 雲端部署 (Vercel + Neon)

1. 在 [Neon.tech](https://neon.tech) 建立免費 PostgreSQL 並取得 `DATABASE_URL`。
2. 將連結放入 `.env` (本地) 與 Vercel 環境變數。
3. 本地執行 `python 1_update.py` 將商品同步到雲端資料庫。
4. 推送到 GitHub，Vercel 會依 `vercel.json` 自動部署。

詳細步驟見 [docs/deployment.md](docs/deployment.md)。

---

## ⚙️ 設定

- **`config.json`**：首頁文案 (標題、收單時間)、購買說明、特別活動、生產模式 port。每期團購通常只需改 `hero.deadline_info`。
- **`.env`**：環境變數，範本見 [.env.example](.env.example)。

| 變數 | 必填 | 說明 |
|------|------|------|
| `DATABASE_URL` | 否 | 留空用 SQLite；填 Neon 連線字串切換為 Postgres |
| `SECRET_KEY` | 建議 | Flask session 金鑰，正式環境請自訂 |
| `GOOGLE_SCRIPT_URL` | 否 | 送單時同步到 Google 表單的 Apps Script 網址 |

---

## 📂 專案結構

```
coffee/
├── 1_update.py          # [入口] 爬蟲更新商品
├── 2_server.py          # [入口] 開發伺服器
├── 3_export.py          # [入口] 收單匯出 Excel
├── run_prod.sh / .bat   # [入口] 生產伺服器 (Gunicorn / Waitress)
├── config.json          # 網站文案與 port 設定
├── vercel.json          # Vercel 部署設定
├── requirements.txt
├── .env.example         # 環境變數範本
│
├── src/                 # 核心程式
│   ├── app.py           #   Flask 路由與 API
│   ├── models.py        #   資料庫存取 (SQLite / Postgres)、Excel 匯出
│   ├── scraper.py       #   爬蟲
│   └── google_integration.py  # Google 表單整合 (選用)
├── api/index.py         # Vercel serverless 進入點
├── templates/           # Jinja2 HTML 模板
├── static/              # CSS、JS、商品圖片快取 (products/ 不進版控)
├── scripts/             # 維護用一次性腳本 (見下)
├── docs/                # 技術文件 (見下)
└── data/                # 本地資料，不進版控：products.db、匯出的 xlsx、feedback.json
```

### 維護腳本 (`scripts/`)

| 腳本 | 用途 |
|------|------|
| `migrate_to_neon.py` | 把本地 SQLite 的訂單、歷史銷量、評論一次搬到 Neon |
| `reprocess_db.py` | 用最新的解析規則重新判定既有商品的烘焙度與處理法 |
| `sync_db.py` | 從 `data/products.json` 匯入商品到資料庫 (離線還原用) |

皆需在專案根目錄執行，例如 `python scripts/reprocess_db.py`。

### 技術文件 (`docs/`)

- [architecture.md](docs/architecture.md) — 技術堆疊、模組架構與核心流程
- [database.md](docs/database.md) — 資料表結構與 JSON 檔案
- [api.md](docs/api.md) — AJAX API 介面
- [deployment.md](docs/deployment.md) — 本地與雲端部署指南
- [CHANGELOG.md](CHANGELOG.md) — 版本紀錄
