# OwlPay Harbor 產品與技術全貌

> 本文件同時面向產品、業務與技術人員，提供 Harbor 系統的完整脈絡。
> 技術細節以摺疊區塊或標註呈現，非技術讀者可跳過不影響理解。

---

## 目錄

- [1. Harbor 是什麼？](#1-harbor-是什麼)
- [2. 系統架構](#2-系統架構)
- [3. On-Ramp（入金）：法幣 → 穩定幣](#3-on-ramp入金法幣--穩定幣)
- [4. Off-Ramp（出金）：穩定幣 → 法幣](#4-off-ramp出金穩定幣--法幣)
- [5. 報價機制（Quote）](#5-報價機制quote)
- [6. 動態表單機制（Requirements）](#6-動態表單機制requirements)
- [7. 支付方式詳解](#7-支付方式詳解)
- [8. 手續費結構](#8-手續費結構)
- [9. 合規與 AML](#9-合規與-aml)
- [10. 技術參考索引](#10-技術參考索引)

---

## 1. Harbor 是什麼？

Harbor 是 OwlPay 的穩定幣出入金平台，讓用戶能夠：

- **入金（On-Ramp）**：把銀行裡的美元等法幣，轉換成鏈上的 USDC 穩定幣
- **出金（Off-Ramp）**：把鏈上的 USDC 穩定幣，兌換回法幣並匯入銀行帳戶

簡單來說，Harbor 是 **法幣世界與區塊鏈世界之間的橋樑**。

### 支援的區塊鏈

Ethereum、Polygon、Arbitrum、Optimism、Avalanche、Stellar、Solana

### 支援的穩定幣

USDC（主要）

---

## 2. 系統架構

Harbor 由四個獨立服務組成，透過 HTTP API 與 Webhook 串接：

```
┌──────────────────────┐  ┌──────────────────────┐
│   Harbor Portal      │  │   Harbor Admin        │
│   （合作夥伴管理介面）  │  │   （OwlPay 內部管理）   │
│   給 application 管理  │  │   給內部人員審核、       │
│   客戶、轉帳、帳務     │  │   監控、管理系統         │
└──────────┬───────────┘  └──────────┬───────────┘
           │ HTTP API                │ HTTP API
           └──────────┬─────────────┘
                      │
           ┌──────────▼───────────┐
           │   Harbor API         │  Laravel 後端
           │   （核心業務邏輯）      │  客戶管理、轉帳、報價、AML、Webhook
           └──────────┬───────────┘
                      │ HTTP API + Webhook
           ┌──────────▼───────────┐
           │   Bank Module        │  Laravel 後端
           │   （金融操作引擎）      │  銀行對接、Circle 整合、區塊鏈操作
           └──────────┬───────────┘
                      │
                ┌─────┼──────┬──────────┐
                ▼     ▼      ▼          ▼
             FV Bank  Circle  MX.com   區塊鏈
             (銀行)   (CPN)  (帳戶連結) (Ledger/Stellar)
```

| 服務 | 職責 | 使用者 | 技術棧 |
|------|------|--------|--------|
| **Harbor Portal** | 合作夥伴管理介面（客戶、轉帳、帳務） | Application 擁有者（商戶/合作夥伴） | Vue 3, TypeScript, PrimeVue, Pinia |
| **Harbor Admin** | 內部管理介面（審核、監控、系統管理） | OwlPay 內部人員 | Vue 3, TypeScript, PrimeVue |
| **Harbor API** | 業務邏輯核心（轉帳流程、費率、AML） | — | Laravel 12, PHP 8.2, MySQL, Redis |
| **Bank Module** | 金融操作（銀行、Circle、區塊鏈對接） | — | Laravel 11, PHP 8.2, MySQL, Redis, AWS KMS |

> **重要**：Portal 和 Admin 是完全獨立的專案，有各自的 repository、dependencies 和 coding conventions。它們透過 Harbor API 溝通，彼此不共享程式碼。

### 外部服務依賴

| 服務 | 用途 |
|------|------|
| **FV Bank（Lead Bank）** | 美國銀行合作夥伴，處理法幣收付 |
| **Circle Payment Network (CPN)** | USDC 發行方的支付網路，處理跨境出金與穩定幣轉換 |
| **Circle Mint** | USDC 鑄造與銷毀，跨鏈橋接 |
| **MX.com** | 銀行帳戶連結服務（ACH 扣款授權用） |
| **Ledger Service / Stellar Service** | 區塊鏈上的 USDC 轉帳操作 |
| **AML Service** | 反洗錢合規檢查 |

### 雙層 API 架構

Harbor API 對外提供兩層 API：

| API 層 | 路徑前綴 | 消費者 | 認證方式 |
|--------|---------|--------|---------|
| **Application API** | `/api/v2/applications/{id}/...` | 外部應用程式（API Key） | API Key + Signature |
| **Portal API** | `/portal/v2/applications/{appUuid}/...` | Harbor Portal 前端 | Cookie + Session（Sanctum） |

Portal API 大多數端點會委派（delegate）給 Application API 處理，但會在回傳結果中附加額外欄位（如 `beneficiary_info`、`destination_type`），提供 Portal 前端所需的完整資料。

---

## 3. On-Ramp（入金）：法幣 → 穩定幣

### 一句話理解

> 用戶把美元匯到 Harbor 的銀行帳戶，Harbor 在區塊鏈上發送等值 USDC 給用戶。

### 流程圖

```
用戶                    Harbor                     銀行/區塊鏈
 │                        │                           │
 │  ① 請求報價             │                           │
 │ ──────────────────────>│                           │
 │  ② 回傳匯率+手續費      │                           │
 │ <──────────────────────│                           │
 │                        │                           │
 │  ③ 確認轉帳             │                           │
 │ ──────────────────────>│                           │
 │  ④ 回傳銀行帳戶資訊      │                           │
 │ <──────────────────────│                           │
 │                        │                           │
 │  ⑤ 匯款（WIRE/ACH）    │                           │
 │ ───────────────────────────────────────────────── >│
 │                        │   ⑥ 銀行確認收款            │
 │                        │ <─────────────────────────│
 │                        │                           │
 │                        │   ⑦ AML 驗證（條件性）      │
 │                        │                           │
 │                        │   ⑧ 鏈上發送 USDC          │
 │                        │ ─────────────────────────>│
 │  ⑨ 收到 USDC           │                           │
 │ <──────────────────────│                           │
```

### 狀態流轉

```
PENDING_CUSTOMER_TRANSFER_START   用戶尚未匯款，等待中
        ↓
PENDING_HARBOR                    Harbor 已收到法幣，處理中
        ↓
   [ON_HOLD]                      （條件性）AML 審查中
        ↓
   [REQUEST_FOR_INFORMATION]      （條件性）需要用戶補充更多資訊
        ↓
PENDING_EXTERNAL                  USDC 已發送，等待鏈上確認
        ↓
COMPLETED                         完成，用戶已收到 USDC
```

**異常狀態**：

| 狀態 | 說明 |
|------|------|
| `expired` | 用戶未在期限內匯款，轉帳過期 |
| `refunded` | 轉帳已全額退款 |
| `too_small` / `too_large` | 金額低於下限或超過上限 |
| `error` | 系統錯誤 |
| `reject` | AML 審查未通過 |
| `cancelled` | 用戶取消轉帳 |

### `source_received` 欄位

API 回傳中的 `source_received` 是根據轉帳狀態衍生的布林值，而非獨立儲存的欄位。當狀態進入以下任一狀態時為 `true`：

`pending_customer_transfer_complete`、`pending_external`、`pending_harbor`、`on_hold`、`pending_customer`、`completed`

### 入金支付方式

| 方式 | 說明 | 結算時間 | 手續費 |
|------|------|----------|--------|
| **WIRE（電匯）** | 用戶自行從銀行匯款到 Harbor 帳戶 | 1-3 天 | $40~$150 |
| **ACH_PULL（自動扣款）** | Harbor 從用戶連結的銀行帳戶自動扣款 | 2-5 天 | $0（含在匯率利差中） |

### 入金 Transfer Instructions

入金確認後，Harbor 會回傳以下銀行帳戶資訊供用戶匯款：

| 欄位 | 說明 | 必要 |
|------|------|------|
| `account_number` | 帳號 | 是 |
| `routing_number` | ABA routing number（美國國內） | 條件性 |
| `swift_code` | SWIFT code（國際匯款） | 條件性 |
| `intermediary_bank_name` | 中轉銀行名稱 | 條件性 |
| `intermediary_bank_swift_code` | 中轉銀行 SWIFT code | 條件性 |
| `bank_name` | 銀行名稱 | 是 |
| `bank_address` | 銀行地址 | 是 |
| `account_holder_name` | 帳戶持有人 | 是 |
| `narrative` | 匯款附言（用於識別匯款人） | 是 |

### 入金 Provider

**只有一個：FV Bank（owlting）**

Circle CPN 目前未實作入金報價，入金完全由 FV Bank 處理。

---

## 4. Off-Ramp（出金）：穩定幣 → 法幣

### 一句話理解

> 用戶把 USDC 發到 Harbor 提供的區塊鏈地址，Harbor 將等值法幣匯入用戶的銀行帳戶。

### 流程圖

```
用戶                    Harbor                     銀行/區塊鏈
 │                        │                           │
 │  ① 請求報價             │                           │
 │ ──────────────────────>│  （同時取 FV Bank + CPN）   │
 │  ② 回傳多組報價          │                           │
 │ <──────────────────────│                           │
 │                        │                           │
 │  ③ 選擇報價+確認轉帳    │                           │
 │ ──────────────────────>│                           │
 │  ④ 回傳 USDC 接收地址   │                           │
 │ <──────────────────────│                           │
 │                        │                           │
 │  ⑤ 發送 USDC 到該地址   │                           │
 │ ───────────────────────────────────────────────── >│
 │                        │   ⑥ 區塊鏈確認收到 USDC     │
 │                        │ <─────────────────────────│
 │                        │                           │
 │                        │   ⑦ AML 驗證（條件性）      │
 │                        │                           │
 │                        │   ⑧ 法幣匯款到用戶銀行      │
 │                        │ ─────────────────────────>│
 │  ⑨ 收到法幣             │                           │
 │ <──────────────────────│                           │
```

### 狀態流轉

```
PENDING_CUSTOMER_TRANSFER_START   用戶尚未發送 USDC，等待中
        ↓
PENDING_CUSTOMER_TRANSFER_COMPLETE  用戶已發送 USDC，等待確認（出金專用）
        ↓
PENDING_HARBOR                    Harbor 已收到 USDC，處理中
        ↓
   [ON_HOLD]                      （條件性）AML 審查中
        ↓
   [REQUEST_FOR_INFORMATION]      （條件性）需要用戶補充更多資訊
        ↓
PENDING_EXTERNAL                  法幣已匯出，等待銀行確認
        ↓
COMPLETED                         完成，用戶已收到法幣
```

> 注意：`pending_customer_transfer_complete` 狀態僅用於出金流程，表示鏈上資金已到帳但尚未進入 Harbor 處理。

### 出金 Transfer Instructions

出金確認後，Harbor 回傳區塊鏈地址供用戶發送 USDC：

| 欄位 | 說明 |
|------|------|
| `instruction_chain` | 接收 USDC 的區塊鏈 |
| `instruction_address` | 接收 USDC 的錢包地址 |
| `instruction_memo` | 地址附言（部分鏈需要，如 Stellar） |

### 出金 Provider：雙軌並行

| Provider | 匯率來源 | 覆蓋範圍 | 支付方式 |
|----------|----------|----------|----------|
| **FV Bank（owlting）** | 內部資料庫（靜態 + markup） | 主要美國境內 | DOMESTIC_WIRE, WIRE |
| **Circle CPN** | Circle 外部 API（即時報價） | 多國跨境 | ACH, WIRE, SEPA, PIX, CLABE, FPS 等 |

兩個 provider 的報價 **同時回傳**，由前端/應用端選擇最適合的方案。

### 出金 Destination Type

出金轉帳包含 `destination_type` 欄位，標示收款方類型：`individual`（個人）或 `business`（公司）。

### 出金獨有機制：跨鏈橋接

當用戶的 USDC 所在的鏈不是 Circle CPN 支援的出金鏈時，Bank Module 會自動處理跨鏈：

```
用戶 USDC 在 Arbitrum → CCTP/Circle Mint 橋接到 Ethereum → CPN 從 Ethereum 出金
```

---

## 5. 報價機制（Quote）

### 報價是什麼？

用戶在入金或出金前，系統會先提供一份「報價」，包含：匯率、手續費、結算時間、報價有效期。用戶確認報價後才正式發起轉帳。

### 報價的產生鏈

#### 入金報價（單軌）

```
Harbor API
  → Bank Module
    → FV Bank Service（唯一 provider）
      → 查詢內部 exchange_rates 資料表
      → 套用 markup（基點利差）
      → 計算銀行手續費（硬編碼費率表）
      → 產出報價
    ← 回傳報價
  ← 疊加 Harbor Fee + Commission Fee
← 回傳給前端
```

#### 出金報價（雙軌）

```
Harbor API
  → Bank Module
    ├→ Circle CPN Service
    │   → 呼叫 Circle 外部 API: POST /v1/cpn/quotes（即時）
    │   ← 回傳報價（匯率、手續費由 Circle 決定）
    │
    └→ FV Bank Service
        → 本地計算（同入金邏輯）
        ← 回傳報價
    ← 合併兩組報價
  ← 各自疊加對應的 Harbor Fee + Commission Fee
← 所有報價一起回傳給前端
```

### 報價與轉帳建立的二階段流程

轉帳建立是一個二階段過程：

```
① Quote 階段（報價）
   Harbor API 呼叫 TransferCalculator，使用 FeeSet 計算 commission_fee 和 harbor_fee
   同時從 Bank Module 取得匯率和報價
   結果存入 transfer_quotes 表

② Store 階段（建立轉帳）
   Harbor API 再次執行 TransferCalculator
   寫入 transfers 表時，quote 階段的值有優先權：
     commission_fee = transfer_quote.commission_fee ?? 計算值
     harbor_fee     = transfer_quote.harbor_fee ?? 計算值
     exchange_rate  = transfer_quote.exchange_rate ?? 計算值
     source_amount  = transfer_quote.source_amount ?? 計算值
     destination_amount = transfer_quote.destination_amount ?? 計算值
```

> **為什麼 quote 優先**：FeeSet 的費率可能在 quote 和 store 之間發生變化（例如月初 tier 重算），使用 quote 階段的值確保用戶看到的報價與最終收費一致。

### 入金 vs 出金報價差異

| 面向 | 入金 | 出金 |
|------|------|------|
| Provider 數量 | 1 個（FV Bank） | 2 個（FV Bank + Circle CPN） |
| 外部 API 呼叫 | 無 | 有（Circle 即時報價） |
| 匯率性質 | 靜態（DB + markup） | FV Bank: 靜態 / CPN: 即時 |
| 報價有效期 | 1 小時 | 1 小時 |

---

## 6. 動態表單機制（Requirements）

### 背景

不同的轉帳方式（WIRE、PIX、SEPA...）需要用戶填寫不同的資訊。Harbor 透過 `/requirements` API 回傳 **JSON Schema**，前端根據 schema 動態渲染表單。

### 表單是怎麼產生的？

```
Bank Module                    Harbor API                     前端
  │                              │                              │
  │  PHP Config 定義              │                              │
  │  ├ recipient_methods/*.php   │                              │
  │  │  （每種支付方式的欄位）      │                              │
  │  └ travel_rule/*.php         │                              │
  │    （每種客戶類型的 KYC 欄位） │                              │
  │                              │                              │
  │  合併產出 JSON Schema ──────>│                              │
  │                              │  ① 建立報價時觸發              │
  │                              │  GenerateHarborSchema        │
  │                              │    1. 拿 Bank Module schema  │
  │                              │    2. 用 mapping 轉換欄位名   │
  │                              │    3. 存成靜態 JSON 檔案      │
  │                              │                              │
  │                              │  ② 前端呼叫 /requirements    │
  │                              │    → 讀取靜態 JSON 回傳 ────>│
  │                              │                              │  動態渲染表單
```

**關鍵點：產生時動態，使用時靜態。** Schema 在報價建立時產生並存檔，前端請求時直接讀取。

### 表單的變化維度

Schema 檔名格式：`{支付方式}.{來源類型}.{目標類型}.v1.json`

| 維度 | 說明 | 範例 |
|------|------|------|
| **支付方式** | 決定要填哪些金融帳戶欄位 | `US_WIRE`, `BR_PIX`, `EU_SEPA`, `HK_FPS` |
| **來源客戶類型** | 個人 or 公司 | `individual`, `business` |
| **目標客戶類型** | 個人 or 公司 | `individual`, `business` |

#### 入金表單：固定

- 支付方式永遠是 `US_BLOCKCHAIN`
- 用戶需填：收款錢包地址 + 收款人 KYC 資料

#### 出金表單：依國家和支付方式變化

| 支付方式 | 用戶需填寫 |
|----------|-----------|
| **US_WIRE** | 帳號、routing number、銀行名稱 |
| **BR_PIX** | 巴西 CPF 稅號 + 電話/Email/PIX EVP（三擇一） |
| **EU_SEPA** | IBAN、BIC/SWIFT |
| **HK_FPS** | FPS ID、銀行帳號 |
| **MX_CLABE** | 18 位 CLABE 帳號 |

---

## 7. 支付方式詳解

### 入金方式

#### WIRE（電匯）

- **運作方式**：用戶自行從銀行匯款到 Harbor 的 FV Bank 帳戶
- **Harbor 提供**：帳號、routing number、SWIFT code、bank name、narrative（匯款附言）
- **結算時間**：1-3 個工作天
- **手續費**：$40~$150（依金額和路徑）
- **適用範圍**：美國國內 + 國際（SWIFT）

#### ACH_PULL（自動扣款）

- **運作方式**：用戶先連結銀行帳戶（透過 MX.com），Harbor 自動從帳戶扣款
- **結算時間**：2-5 個工作天
- **手續費**：$0（費用包含在匯率利差中）
- **適用範圍**：僅限美國境內（USD）
- **前提條件**：需先完成銀行帳戶連結 + 即時餘額檢查通過
- **Provider**：FV Bank
- **Feature Flag**：可透過 `ENABLE_ACH_PULL_TRANSFER` 開關控制

**ACH 銀行連結流程（MX.com）：**

```
用戶 → 開啟 MX Connect Widget → 登入自己的網路銀行 → MX 驗證並取得帳戶資訊
     → Bank Module 建立 BankLinkedAccount → 之後轉帳時可直接選擇此帳戶
```

### 出金方式

| 方式 | Provider | 適用地區 | 說明 |
|------|----------|----------|------|
| DOMESTIC_WIRE | FV Bank | 美國 → 美國 | 美國國內電匯 |
| WIRE | FV Bank / CPN | 國際 | 國際電匯 |
| ACH | CPN | 美國 | 美國 ACH 出金 |
| SEPA | CPN | 歐盟 | 歐洲單一支付區 |
| PIX | CPN | 巴西 | 巴西即時支付 |
| CLABE/SPEI | CPN | 墨西哥 | 墨西哥銀行轉帳 |
| FPS | CPN | 香港 | 快速支付系統 |

---

## 8. 手續費結構

### 費用組成

Harbor 的轉帳涉及多層費用，但**並非所有費用都從用戶的轉帳金額中扣除**：

```
用戶匯入的金額（source_amount）
│
├── ① Commission Fee（佣金）──── 從 source_amount 中扣除
│   由 application 自訂的加價
│   公式：固定金額 + (金額 × 百分比)
│   這是唯一從用戶轉帳金額中實際扣除的 Harbor 層費用
│
├── ② Exchange Rate（匯率換算）
│   扣除 commission_fee 後的金額 × exchange_rate = destination_amount
│   匯率已包含 OwlPay Markup 和銀行/Provider 處理費
│
│   ─────────────────────────
│   = 用戶實際收到的金額（destination_amount）
│
│
│ 【以下費用不影響此筆轉帳的金額進出】
│
├── ③ Harbor Fee（平台費）──── 月底與 application 結算
│   依 FeeSet + FeeSetTier 分層計費
│   公式：固定費用 + (金額 × 費率基點)
│   費率依該 application 當月累計交易量決定 tier
│   FV Bank 和 CPN 各有獨立的費率表
│   ※ 記錄在 transfer 上，但不從轉帳金額中扣除
│
└── ④ OwlPay Markup（匯率利差）──── 包含在匯率中
    Bank Module 層在匯率上加的利差（basis points）
    用戶不會直接看到，已反映在 exchange_rate 中
```

### 金流計算公式

**入金（On-Ramp）**：

```
destination_amount ≈ (source_amount - commission_fee) × exchange_rate
```

**出金（Off-Ramp）**：

```
destination_amount ≈ (source_amount - commission_fee) × exchange_rate
```

> **注意**：`destination_amount` 和 `exchange_rate` 的實際值來自 Bank Module 的報價（FV Bank 或 Circle CPN），上述公式僅為近似表示。詳見下方「Receipt 的雙來源架構」。

### Receipt 的雙來源架構

Transfer 的 receipt 資料來自**兩個獨立系統**，它們之間沒有嚴格的數學對應關係：

| 欄位 | 來源 | 說明 |
|------|------|------|
| `initial_amount`（= source_amount） | Bank Module 報價 | 用戶匯入金額 |
| `final_amount`（= destination_amount） | Bank Module 報價 | 用戶收到金額 |
| `exchange_rate` | Bank Module 報價 | 匯率（含 markup） |
| `commission_fee` | TransferCalculator（本地計算） | 從 FeeSet 計算 |
| `harbor_fee` | TransferCalculator（本地計算） | 從 FeeSet 計算 |

因此，如果你嘗試用 `(initial_amount - commission_fee) × exchange_rate` 驗算 `final_amount`，結果可能不會完全吻合。這是因為 Bank Module 報價時使用的內部計算邏輯（含銀行手續費、markup 等）與 Harbor 層的費用計算是分開進行的。

### 入金手續費速覽

| 支付方式 | 銀行處理費 | Harbor Fee | 匯率利差 |
|----------|-----------|------------|----------|
| WIRE（美國→美元） | $40~$150 | 依 tier（月結） | 有 |
| WIRE（非美國→非美元） | $35 | 依 tier（月結） | 有 |
| ACH_PULL | $0 | 依 tier（月結） | 有（較高，補貼零手續費） |

### 出金手續費速覽

| Provider | 銀行處理費 | Harbor Fee |
|----------|-----------|------------|
| FV Bank（美國國內） | $12 | 依 FV Bank tier（月結） |
| FV Bank（國際） | $35~$150 | 依 FV Bank tier（月結） |
| Circle CPN | Circle API 回傳 | 依 CPN tier（月結） |

---

## 9. 合規與 AML

### 觸發條件

- 單筆金額 **> $3,000 USD**
- 或轉帳對象 **非本人**（非 self-transfer）

### $3,000 門檻的額外要求

當入金金額超過 $3,000 USD 時，需要收集額外資訊：

**錢包資訊**（所有 > $3,000 的入金）：
- `beneficiary_receiving_wallet_type`：錢包類型（Personal Wallet / Third Party Custodian / Other）
- `beneficiary_institution_name`：託管機構名稱（如適用）

**受益人 KYC 資訊**（> $3,000 且非 self-transfer）：
- 出生日期、Email、電話
- 居住地址（街道、城市、州/省、郵遞區號、國家）
- 身分證件（類型、簽發國、號碼）

### 流程

```
轉帳進行中
    ↓
觸發 AML 檢查 → 狀態變為 ON_HOLD
    ↓
Harbor → AML Service 提交：
  ├ Travel Rule 資料（姓名、生日、地址、證件號碼）
  ├ 來源/目標帳戶資訊
  └ 交易金額與目的
    ↓
AML 審核通過 → 狀態恢復，繼續處理
AML 審核拒絕 → 狀態變為 REJECT
```

### Travel Rule 資料

| 資料項 | 個人客戶 | 公司客戶 |
|--------|----------|----------|
| 姓名 | 全名 | 公司名稱 |
| 日期 | 出生日期 | 公司成立日期 |
| 地址 | 居住地址 | 註冊地址 |
| 證件 | 身分證/護照號碼 | 統一編號/註冊編號 |
| 國家 | 國籍 | 註冊國家 |

---

## 10. 技術參考索引

### 核心檔案速查

#### Harbor API（業務邏輯層）

| 檔案 | 說明 |
|------|------|
| `app/Http/Controllers/Application/V2/TransferController.php` | 轉帳主控制器（報價、建立、查詢） |
| `app/Http/Controllers/Portal/v2/TransferController.php` | Portal 專用轉帳控制器（委派 + 附加資料） |
| `app/Services/TransferService.php` | 轉帳業務邏輯 |
| `app/Repositories/TransferRepository.php` | 轉帳 DB 寫入（含 quote 優先邏輯） |
| `app/Services/HttpService/BankModuleService.php` | 呼叫 Bank Module 的 HTTP 客戶端 |
| `app/Services/HttpService/AMLService.php` | 呼叫 AML 服務 |
| `app/Services/Tools/TransferCalculator.php` | 費用計算器（commission_fee、harbor_fee） |
| `app/Services/Tools/HarborCustomerSchemaBuilder.php` | 表單 Schema 轉換器 |
| `app/Console/Commands/GenerateHarborSchema.php` | Schema 產生 Artisan 指令 |
| `app/Http/Controllers/Callback/BankModuleController.php` | Bank Module Webhook 接收 |
| `app/Http/Resources/Application/V2/TransferResource.php` | Application API 轉帳回應格式 |
| `app/Http/Resources/Portal/v2/TransferResource.php` | Portal API 轉帳回應格式（擴充 beneficiary_info） |
| `app/Models/Transfer.php` | 轉帳主模型 |
| `app/Enums/TransferStatusEnum.php` | 轉帳狀態定義 |
| `app/Enums/TransferTypeEnum.php` | 轉帳類型定義（on-ramp / off-ramp / swap） |
| `config/schema.php` | Schema 映射設定 |
| `config/bank_module/schemas/*.php` | 各支付方式的欄位映射 |
| `storage/app/private/harbor/schemas/*.json` | 預產生的表單 JSON Schema |
| `routes/transfers/v2.php` | Application 轉帳 API 路由 |
| `routes/portal/v2.php` | Portal 轉帳 API 路由 |

#### Bank Module（金融操作層）

| 檔案 | 說明 |
|------|------|
| `app/Http/Controllers/Application/V1/OrderQuoteController.php` | 報價產生控制器 |
| `app/Services/External/FvBankService.php` | FV Bank 報價 + 費率計算 |
| `app/Services/External/CirclePaymentNetworkService.php` | Circle CPN 報價整合 |
| `app/Services/External/MXApiService.php` | MX.com API 整合 |
| `app/Services/BankAccountLinking/MXDriver.php` | MX 銀行帳戶連結驅動 |
| `app/Models/Order.php` | 訂單主模型 |
| `app/Models/BankLinkedAccount.php` | 已連結銀行帳戶 |
| `app/Enums/OrderStatusEnum.php` | 訂單狀態定義 |
| `app/Enums/OrderPaymentMethodEnum.php` | 支付方式列舉 |
| `app/Console/Commands/HandleCPNWithdrawOrder.php` | CPN 出金處理指令 |
| `config/requirements/recipient_methods/*.php` | 各支付方式的欄位定義（Schema 來源） |
| `config/requirements/travel_rule/*.php` | 各客戶類型的 KYC 欄位定義 |

#### Harbor Portal（前端）

| 檔案 | 說明 |
|------|------|
| `src/api/transfers.ts` | 轉帳 API 客戶端 |
| `src/types/transfer.ts` | 轉帳 TypeScript 型別定義 |
| `src/stores/transfers.ts` | 轉帳狀態管理（Pinia） |
| `src/views/Transfers/` | 轉帳頁面元件 |
| `src/components/DepositTransferDetailContent.vue` | 入金 Transfer 詳情內容 |
| `src/components/WithdrawTransferDetailContent.vue` | 出金 Transfer 詳情內容 |
| `src/components/transferDetailStyles.scss` | Transfer 詳情共用樣式 |
| `src/composables/useCopyTransferInstructions.ts` | 複製轉帳指示 composable |

### API 端點速查

#### Application API（外部應用程式使用）

| 方法 | 端點 | 說明 |
|------|------|------|
| POST | `/v2/applications/{id}/transfers/quotes` | 取得報價 |
| GET | `/v2/applications/{id}/transfers/quotes/{id}/requirements` | 取得動態表單 Schema |
| POST | `/v2/applications/{id}/transfers` | 建立轉帳 |
| GET | `/v2/applications/{id}/transfers/{uuid}` | 查詢轉帳狀態 |
| GET | `/v2/applications/{id}/transfers` | 列出所有轉帳 |
| GET | `/v2/applications/{id}/transfers/supported_chains` | 取得支援的區塊鏈 |
| GET | `/v2/applications/{id}/transfers/withdrawal_settings` | 取得出金設定 |
| POST | `/callback/bank_module` | Bank Module 狀態回調 |

#### Portal API（Portal 前端使用）

| 方法 | 端點 | 說明 |
|------|------|------|
| POST | `/portal/v2/applications/{appUuid}/transfers/quotes` | 取得報價 |
| GET | `/portal/v2/applications/{appUuid}/transfers/quotes/{quoteUuid}/requirements` | 取得動態表單 Schema |
| POST | `/portal/v2/applications/{appUuid}/transfers` | 建立轉帳 |
| GET | `/portal/v2/applications/{appUuid}/transfers` | 列出轉帳（含 beneficiary_info） |
| GET | `/portal/v2/applications/{appUuid}/transfers/{transferUuid}` | 轉帳詳情（含 beneficiary_info、destination_type） |
| GET | `/portal/v2/applications/{appUuid}/transfers/supported_chains` | 取得支援的區塊鏈 |
| GET | `/portal/v2/applications/{appUuid}/transfers/withdrawalSettings` | 取得出金設定 |
| GET | `/portal/v2/applications/{appUuid}/transfers/DepositSettings` | 取得入金設定 |
| GET | `/portal/v2/applications/{appUuid}/transfers/travel_rule` | 取得 Travel Rule Schema |

> Portal API 的 list 和 show 回傳的 Transfer 比 Application API 多了 `destination.beneficiary_info` 和 `destination.destination_type`（出金）欄位。Portal API 的 null 欄位會被自動過濾（`array_filter`），避免前端顯示空值。

### 環境配置

| 環境 | Harbor API | Bank Module |
|------|-----------|-------------|
| Production | `harbor.owlpay.com` | `bank-module.owlpay.com` |
| Staging | `harbor-stage.owlpay.com` | `bank-module-stage.owlpay.com` |
| Sandbox | `harbor-sandbox.owlpay.com` | — |
