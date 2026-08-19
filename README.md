# Google Sheets API (Vercel Serverless)

以 Express 5 寫成、部署在 Vercel 的 Serverless API，讓程式與 AI Agent 可以透過 HTTP 讀寫多個 Google Sheets。

- 正式站：<https://sheets-api-xi.vercel.app>
- GitHub：<https://github.com/WenGlen/SheetsAPI>（push 到 `main` 後 Vercel 自動部署）

## 專案結構

```
SheetsAPI/
├── api/
│   ├── index.js                # 所有路由與 Google Sheets 存取邏輯（Express app）
│   └── docs.js                 # 自我說明文件（HowToUseForAgent / 入口 endpoints 的內容來源）
├── setup-shopee-templates.js   # 一次性腳本：建立蝦皮相關分頁範本
├── vercel.json                 # Vercel 部署設定（所有路徑 rewrite 到 /api）
├── SETUP.md                    # Google Cloud 憑證與 Sheet 權限的詳細設定步驟
├── .env                        # 環境變數（不上傳）
├── package.json
└── README.md
```

> `api/index.js` 匯出 Express `app`；本地會 `app.listen`，在 Vercel 上則作為 Serverless Function 匯出。

## 核心概念

- **一個 sheet = 一組環境變數**：URL 的 `:sheet` 會對應到 `SHEET_ID_<大寫>`。
  例：`/api/glen` → `SHEET_ID_GLEN`、`/api/test` → `SHEET_ID_TEST`。
- **分頁用 `tab=` 指定**：分頁名稱含中文或特殊字元時，**必須先 `encodeURIComponent` 編碼**再放進 URL。
  例：`tab=商品毛利計算` → `tab=%E5%95%86%E5%93%81%E6%AF%9B%E5%88%A9%E8%A8%88%E7%AE%97`。
  可先呼叫 `GET /api/:sheet/tabsName` 取得原始分頁名稱再自行編碼。
- **`row` 編號不含標題列**：`row=1` 是第一筆「資料」，也就是試算表的第 2 行。標題列位移由後端處理，呼叫端不必自己 +1。
- **單格更新**：`PUT .../row=N/col=欄位名` 一次只改一格，避免用整列更新覆寫到其他欄造成的同時複寫衝突。詳見下方〈儲存格定位規則〉。
- **給 AI Agent 的自我說明**：`GET /api/:sheet/HowToUseForAgent` 會回傳一份完整、機器可讀的用法說明（含每個分頁的操作與已編碼好的 URL），讓 Agent 不必猜路徑。

## 環境變數

在 `.env`（本地）或 Vercel 專案設定中配置。憑證擇一即可：

### 憑證方式 A：分開的欄位（適合本地開發）

```env
GOOGLE_CLIENT_EMAIL=your-service-account@your-project.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
```

### 憑證方式 B：完整 JSON 字串（適合 Vercel）

```env
GOOGLE_CREDENTIALS_JSON={"type":"service_account","client_email":"...","private_key":"..."}
```

### Sheet ID 與其他設定

```env
# 每個要串接的 sheet 各一行，key 為 SHEET_ID_<路徑大寫>
SHEET_ID_GLEN=1zdD...B4v8
SHEET_ID_TEST=1vQ5...R4U4

# 本地開發用（Vercel 上會忽略）
PORT=3000

# 選填：設定後，每次 /api/:sheet 的請求會把稽核紀錄 append 到該試算表的 SheetsAPI!A:J
# 留空 / false / 0 代表關閉
ACCESS_LOG_SPREADSHEET_ID=
```

> Sheet ID 是 Google Sheets 網址 `/d/` 與 `/edit` 之間那一段。
> 取得憑證、啟用 Google Sheets API、把 Service Account 加入 Sheet 編輯者的完整步驟見 [SETUP.md](SETUP.md)。

## 本地開發

```bash
npm install

# 方式 1：Vercel 環境（最接近正式）
npm run dev        # = vercel dev

# 方式 2：純 Express
npm start          # = node api/index.js，預設 http://localhost:3000
```

## 部署到 Vercel

已與 GitHub 連動：**push 到 `main` 後 Vercel 會自動重新部署**。

首次設定或改環境變數時，在 Vercel 專案的 **Settings → Environment Variables** 填入上述 `GOOGLE_CREDENTIALS_JSON`（或分開的 `GOOGLE_CLIENT_EMAIL` + `GOOGLE_PRIVATE_KEY`）與各 `SHEET_ID_*`，再重新部署即可。

## API 端點

以下 `:sheet` 代換為 sheet 名稱（如 `glen`）、`:tab` 為 **encodeURIComponent 編碼後**的分頁名稱、`N` 為資料列編號（1 起算，不含標題列）。

### 探索與說明

| Method | Path | 說明 |
|---|---|---|
| GET | `/` | API 入口導覽 |
| GET | `/api/health` | 健康檢查 |
| GET | `/api/:sheet` | 該 sheet 的說明與端點列表 |
| GET | `/api/:sheet/HowToUseForAgent` | 完整用法說明（供 AI Agent 讀取） |
| GET | `/api/:sheet/HowToUseForAgent/分頁1/分頁2/...` | 同上，但把操作範圍鎖定在指定分頁 |
| GET | `/api/:sheet/tabsName` | 取得所有分頁名稱（原始、未編碼） |
| GET | `/api/debug/auth` | 測試 Google 憑證是否正常 |

### 讀取

| Method | Path | 說明 |
|---|---|---|
| GET | `/api/:sheet/tabRaw=:tab` | 分頁原始二維陣列（不處理標題） |
| GET | `/api/:sheet/tab=:tab` | 分頁全部資料（物件陣列，第一行為欄位名） |
| GET | `/api/:sheet/tab=:tab/row=N` | 第 N 筆資料 |
| GET | `/api/:sheet/tab=:tab/row=X-Y` | 第 X～Y 筆資料（含頭尾） |
| GET | `/api/:sheet/tab=:tab/format=:range` | 讀取儲存格格式，`:range` 為 A1 notation（如 `B2:J7`） |

### 寫入資料列 / 儲存格

| Method | Path | 說明 |
|---|---|---|
| POST | `/api/:sheet/tab=:tab` | 新增資料到末尾。body：`{ values:[...] }`（單筆）、`{ values:[[...],[...]] }`（多筆）、或 `{ rows:[{欄位名:值},...] }`（依標題對齊） |
| PUT | `/api/:sheet/tab=:tab/row=N` | 更新第 N 筆**整列**。body：`{ values:[...] }` |
| PUT | `/api/:sheet/tab=:tab/row=N/col=:col` | 更新**單一儲存格**（推薦；避免整列覆寫）。body：`{ value: 值 }` |
| DELETE | `/api/:sheet/tab=:tab/row=N` | 清空第 N 筆資料（列保留、內容清除） |

### 欄位

| Method | Path | 說明 |
|---|---|---|
| POST | `/api/:sheet/tab=:tab/col=:col` | 在標題列末尾新增欄位，可選填 `{ values:[...] }` 一併填各列值 |
| PUT | `/api/:sheet/tab=:tab/col=:col/to=:newCol` | 修改欄位名稱 |

### 分頁

| Method | Path | 說明 |
|---|---|---|
| POST | `/api/:sheet/createTab=:tab` | 建立新分頁 |
| PUT | `/api/:sheet/renameTab=:tab/to=:newTab` | 改分頁名稱 |
| PUT | `/api/:sheet/moveTab=:tab/toIndex=:index` | 移動分頁位置（0 = 最前） |

### 格式

| Method | Path | 說明 |
|---|---|---|
| PUT | `/api/:sheet/tab=:tab/format` | 編輯格式（原生 `userEnteredFormat`）。body：`{ range:{startRowIndex,endRowIndex,startColumnIndex,endColumnIndex}, format:{...} }`（0-based、end exclusive） |
| PUT | `/api/:sheet/tab=:tab/formatSimple` | 編輯格式（簡化）。body：`{ range:{...}, backgroundColor, textColor, bold, italic, fontSize, fontFamily, horizontalAlignment, verticalAlignment, wrapStrategy }` |
| POST | `/api/:sheet/copyFormat=:sourceTab/to=:destTab` | 複製分頁格式。body：`{ source:{...}, destination:{...} }`（destination 省略則與 source 同位置） |

## 儲存格定位規則（重要）

讀寫時最容易出錯的是「定位」。本 API 刻意用兩個好懂、且從讀取結果就看得到的座標，換算全交給後端：

| 座標 | 用什麼 | 規則 |
|---|---|---|
| 列（row） | `row=N` | `N=1` 是第一筆資料，**不含標題列**（= 試算表第 2 行）。與 `getRow`／`getAllRows` 是同一套編號，不必自己 +1。 |
| 欄（col） | `col=欄位名稱` | 用**標題列的欄位名**（就是 `GET tab=:tab` 回傳物件裡的 key），**不是** A/B/C 欄位字母。後端會自動對應到正確欄位；找不到該欄位名時回 404 並列出現有欄位。 |

**單格更新範例**（把 `商品毛利計算` 第 3 筆的「狀態」改成「已上架」）：

```bash
curl -X PUT "https://sheets-api-xi.vercel.app/api/glen/tab=%E5%95%86%E5%93%81%E6%AF%9B%E5%88%A9%E8%A8%88%E7%AE%97/row=3/col=%E7%8B%80%E6%85%8B" \
  -H "Content-Type: application/json" \
  -d '{"value":"已上架"}'
```

## 錯誤處理

回傳格式：

```json
{ "success": false, "error": "錯誤訊息" }
```

常見錯誤：

- `Sheet ID not found for "xxx"` — `.env` 缺對應的 `SHEET_ID_XXX`。
- `找不到欄位「xxx」` — `col` 需為標題列的欄位名稱（非欄位字母）；回應會列出現有欄位。
- `The caller does not have permission` — Service Account 沒有該 Sheet 的編輯權限（見 [SETUP.md](SETUP.md)）。
- `exceeds grid limits` — 目標列／欄超出試算表現有網格範圍。

## License

ISC
