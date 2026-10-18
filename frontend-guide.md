# 前端串接導覽（依畫面流程）

給串接商城 API 的前端工程師。本頁按**使用者操作的順序**列出要呼叫哪些端點、何時呼叫、哪裡容易接錯；欄位與錯誤的完整定義在各角色文件，本頁只連結、不重複。

- 共通規則（信封、認證、錯誤碼、分頁、金額、時間）：[conventions.md](conventions.md)
- 各角色完整端點表：[storefront](storefront/README.md)、[buyer](buyer/README.md)、[seller](seller/README.md)、[union](union/README.md)、[platform](platform/README.md)
- 可直接執行的請求範例：`tests/Bruno/`（`02-buyer`、`03-seller`、`05-platform`）

> 2026-10-15 整理，2026-10-18 更新（見 [§8](#8-2026-10-18-之後的異動)）。目前金流只接 PAYUNi sandbox 信用卡一次付清，物流為 7-ELEVEN 店到店與黑貓宅配；正式環境尚未開放。

---

## 0. 開始之前必讀的六件事

1. **所有回應都是同一個信封** `{success, message, data, errors, meta}`。`message` 可直接顯示給使用者；422 驗證錯誤看 `errors`；分頁看 `meta.pagination`。不要用 HTTP 200 判斷業務成功，看 `success` 與 `data` 內的狀態欄位。
2. **兩種身分、兩套 token，不能混用。** 會員（買家、賣家、工會管理者）用 `POST auth/login`，打 `storefront/`、`buyer/`、`seller/`、`union/`；平台人員用 `POST platform/auth/login`，只打 `platform/`。拿錯 token 會在認證層就回 401。會員 token 代表「一筆會籍」：同一個人在兩個工會是兩個帳號，沒有切換身分的端點。
3. **登入後先看 `must_change_password`。** 為 true 表示仍是初始密碼，前端應強制導向改密碼（`POST auth/password`）；後端不會替你擋其他端點。
4. **金額一律整數新台幣，由後端計算。** 前端送出的金額只用來比對（例如結帳的 `expected_payable_amount`），不會被採用。不要自行加總成營收。
5. **時間格式不一致，請依欄位文件解讀。** 多數欄位為 `YYYY-MM-DD HH:MM:SS`；PAYUNi 相關欄位（付款狀態、能力查詢等）為 UTC 或 ISO 8601 含時區。顯示到期日時一律用回應裡的 `*_at` 欄位，不要自己加天數（工作天與日曆天規則不同）。

6. **token 會過期，收到 401 就導回登入。** 登入與註冊回應帶 `token_expires_at`（絕對到期）與 `token_idle_timeout_minutes`（閒置時限，每次請求重新起算）。會員預設 30 天／7 天，平台人員 12 小時／30 分鐘。可依 `token_expires_at` 提前提示；閒置失效無法預告，任何請求回 401 都應清掉 token 並導回登入頁。見 [storefront/01 token 時限](storefront/01-auth.md#token-時限)。

---

## 1. 訪客與登入

| 畫面 | 呼叫 | 說明 |
| --- | --- | --- |
| 首頁、工會商城 | `GET storefront/home`、`storefront/organizations/*` | 不需登入。見 [storefront/03](storefront/03-home-and-organizations.md) |
| 分類導覽 | `GET storefront/categories`（可帶 `organization_id`）→ `storefront/products?category_id=` | 只列有商品可看的分類，件數與點進去的搜尋總數一致；**主分類涵蓋其子分類**。不要拿它當賣家建商品的分類選單（那是 `seller/categories`）。見 [storefront/02 §5](storefront/02-catalog.md#5-分類導覽) |
| 商品列表、商品頁 | `GET storefront/products`、`storefront/products/{product}` | 每個 SKU 會帶「此刻買不買得到」；會員價須登入才判定。見 [storefront/02](storefront/02-catalog.md) |
| 註冊、啟用、登入、登出、改密碼 | `POST auth/register`、`auth/activate`、`auth/login`、`auth/logout`、`auth/password` | 見 [storefront/01](storefront/01-auth.md)。帳號多由組織核發，註冊不是唯一入口 |
| 忘記密碼 | `POST auth/password/forgot` → 信／簡訊連結開 `/reset-password#token=…` → `POST auth/password/reset` | 申請**一律回同一句**，不要推測寄了沒；重設成功會登出所有裝置、不回 token。重設連結與啟用連結互不通用。見 [storefront/01 §6–7](storefront/01-auth.md#6-忘記密碼申請重設連結) |

---

## 2. 買家：從結帳到收貨

### 2.1 購物車與試算

| 步驟 | 呼叫 |
| --- | --- |
| 看購物車 | `GET buyer/cart` |
| 加入、改數量、移除 | `POST buyer/cart`、`PATCH buyer/cart/items/{item}`、`DELETE buyer/cart/items/{item}` |
| 結帳試算 | `POST buyer/checkout/preview` |

- **一次結帳只能一家店**：`shipments` 恰好一組；混店、兩組 `shipments` 都是 422。購物車有多家店時，前端要讓買家逐店結帳。
- 購物車**不預留庫存**，試算也不預留；送出訂單那一刻才預留。
- **超商取貨不要送任何門市資料**（門市代碼、名稱、地址、`store_map_id`，連 null 都不要送），否則 422。門市由買家在 PAYUNi 付款頁選。只需送取件人姓名、手機。
- 黑貓宅配要送收件人、手機、郵遞區號、地址。

詳見 [buyer/03](buyer/03-cart-and-checkout.md)。

### 2.2 送出訂單

`POST buyer/checkout`，**一定要帶 `expected_payable_amount`**（試算拿到的應付金額）。金額不符會 422，請重新試算讓買家確認。成功 201，回傳主訂單與唯一一張子訂單，並開始計算付款期限（`payment_deadline_at`）。

### 2.3 付款（PAYUNi）—— 最容易接錯的一段

```
① POST buyer/orders/{orderNo}/payments            body: {"provider":"payuni","method":"credit_card"}
② POST buyer/orders/{orderNo}/payments/{paymentNo}/payuni-form
③ 瀏覽器以 Form POST 送出 ② 回傳的 form
④ 買家在 PAYUNi 付款頁付款（超商訂單在這裡選門市）
⑤ PAYUNi 導回商城，商城再以 303 導到前端返回頁 ?order_no={orderNo}
⑥ GET buyer/orders/{orderNo}/payment-status        輪詢，直到 payment.status 不是 created
```

**① 建立付款單**：只建立本地付款單，不會扣款。同訂單已有進行中的付款單時，回同一張（仍是 201）。

**② 領取付款表單**：**每張付款單只會給一次表單**。

- 第一次回 `delivery_status=prepared` 與 `form: {action, method: "POST", fields}`。
- 之後再呼叫回 `delivery_status=already_prepared`、`form: null`。不會重發，也沒有其他取回方式。
- 所以：拿到 `form` 後**立即**在同一個畫面建立表單並送出；不要存進 localStorage、快取、狀態管理持久層或錯誤回報。
- 每分鐘最多 10 次；回應帶 `no-store`。

**③ 送出表單**：依 `form.action` 建立 `<form method="post">`，`fields` 每一項放一個 hidden input，由使用者點擊或程式送出。不要改寫任何欄位值，不要改用 GET 或 fetch。

```js
const f = document.createElement('form');
f.method = 'POST'; f.action = form.action;
for (const [k, v] of Object.entries(form.fields)) {
  const i = document.createElement('input'); i.type = 'hidden'; i.name = k; i.value = v; f.appendChild(i);
}
document.body.appendChild(f); f.submit();
```

**④ PAYUNi 付款頁**：只會出現信用卡一次付清。超商訂單由買家在這頁選門市；黑貓訂單的收件資料會帶入且不可修改，買家可選希望配達時段（由 PAYUNi 處理，商城不保存）。

**⑤ 前端返回頁**：部署設定 `PAYUNI_FRONTEND_RETURN_URL` 指向前端的付款返回頁（例如 `https://shop.example/payment/return`），買家付完款或在付款頁按「返回商店」都會被導到 `…?order_no=M2026…`。頁面讀 `order_no`，帶買家 token 呼叫 ⑥。這個頁面不需要也不應該讀其他參數；未設定時瀏覽器會停在商城 API 的 JSON 提示。見 [前景返回](webhooks/payuni-upp-return.md)。

**⑥ 確認付款結果**：

- **返回頁不代表付款成功**，一律以 `payment-status` 為準。
- 付款成功的通知通常在一分鐘內被系統查核並採用，之後 `payment.status` 變成 `succeeded`、`order_status` 變成 `paid`。建議每 5–10 秒輪詢一次，最多約 2 分鐘；仍是 `created` 就顯示「付款確認中」，讓買家稍後回訂單頁查看。
- **不要因為還是 `created` 就讓買家再付一次。** 付款期限內明確失敗（`failed`）才可以重新走 ①②。
- `payuni_query.outcome=matched` **不是已付款**，不要據此顯示成功。
- 付款期限過後訂單會自動取消；若 PAYUNi 表單已交付，系統會先寬限並查核一次，期限過後的付款會由平台人工處理退款。

詳見 [buyer/04](buyer/04-orders-and-payment.md)。

### 2.4 訂單追蹤與收貨

| 畫面 | 呼叫 | 重點 |
| --- | --- | --- |
| 訂單列表、詳情 | `GET buyer/orders`、`buyer/orders/{orderNo}` | 主訂單只有 4 種狀態；出貨進度看子訂單狀態與 `fulfillment` |
| 超商門市資訊 | 同上，`sub_orders[].recipient` | 付款成立後才有 `pickup_store_code`／`pickup_store_name`，門市地址在 `address` |
| 取消未付款訂單 | `POST buyer/orders/{orderNo}/cancel` | 只能整張取消 |
| 確認收貨 | `POST buyer/sub-orders/{orderNo}/confirm-receipt` | 用**子訂單編號**（含 `-01`）。提前確認不會縮短退貨期限 |

- **貨態不是即時的**：系統定時向 PAYUNi 查詢（封測 30 分鐘一次），畫面上的「已到店」「已送達」可能比實際晚。
- 追蹤號 `shipment.tracking_number` 目前是 PAYUNi 物流序號，**不能直接拿去 7-ELEVEN 或黑貓查件**，請勿做成外部查件連結。
- 超商到店後子訂單轉 `pending_pickup`，取件期限看 `pickup_deadline_at`；送達後自動完成時間看 `auto_complete_at`。

### 2.5 退貨退款、取消、評價

| 功能 | 文件 |
| --- | --- |
| 申請退貨退款（以 `items[].id` 為單位）、登錄寄回單號、爭議 | [buyer/05](buyer/05-returns-and-refunds.md) |
| 評價 | [buyer/06](buyer/06-reviews.md) |
| 未取貨紀錄與申訴 | [buyer/09](buyer/09-pickup-failures.md) |

首波退貨由買家**自行寄回並登錄單號**，沒有超商退貨便。

---

## 3. 賣家：出貨與收款

### 3.1 處理訂單

| 步驟 | 呼叫 | 說明 |
| --- | --- | --- |
| 待出貨清單 | `GET seller/orders` | 以子訂單為單位；收件資料不遮罩 |
| 訂單詳情 | `GET seller/orders/{orderNo}` | |
| 印出貨單（PAYUNi 超商／黑貓） | `POST seller/orders/{orderNo}/payuni-label` | 見 3.2 |
| 印單（其他物流） | `POST seller/orders/{orderNo}/shipping-label` | PAYUNi 物流訂單用這支會 422 |
| 確認出貨 | `POST seller/orders/{orderNo}/ship` | **這一步才扣庫存**。已印 PAYUNi 出貨單時 body 送 `{}`，不要帶物流商或單號 |
| 自取交付 | `POST seller/orders/{orderNo}/handover` | 自取訂單用 |

### 3.2 PAYUNi 出貨單

```
① POST seller/orders/{orderNo}/payuni-label
   超商：{"ship_date":"YYYY-MM-DD"}（可選 label_mode：1=A4、2=直立式）
   黑貓：{"ship_date":"…","delivery_date":"…","spec":1~4}
② 瀏覽器以 Form POST 送出回傳的 form
   超商：PAYUNi 跳轉到 7-ELEVEN 列印頁
   黑貓：直接下載託運單 PDF
③ 賣家交寄後，回商城 POST seller/orders/{orderNo}/ship（body: {}）
```

- 物流商與單號由系統從付款結果取得，前端**不能也不需要**提供。
- **可重印**：再呼叫一次 ① 只會產生新的列印表單，不會新增出貨單。表單同樣不要保存。
- 黑貓日期：出貨日要晚於今天，配達日要晚於出貨日，兩者都不能是週日或國定假日（週六可以）。日期規格錯誤會 422，訊息可直接顯示。
- 每分鐘最多 10 次。
- 印單後要提醒賣家：**超商須 6 天內交寄；黑貓要另外預約收件**（PAYUNi 後台或黑貓客服）。可連到[賣家須知](../guides/seller-payuni-backend.md)。

詳見 [seller/05](seller/05-orders-and-shipping.md)。

### 3.3 其他賣家功能

| 功能 | 文件 |
| --- | --- |
| 開店申請、店面設定 | [seller/01](seller/01-shop-application.md)、[seller/02](seller/02-shop.md) |
| 商品、SKU、庫存 | [seller/03](seller/03-products-and-skus.md)、[seller/04](seller/04-inventory.md) |
| 退貨驗收、取消申請 | [seller/06](seller/06-returns-and-refunds.md) |
| 撥款查詢 | [seller/07](seller/07-payouts.md)：撥款單上的平台費、工會分潤為應計金額；物流費由 PAYUNi 直接從賣家 UNi 帳戶扣，不在撥款單上 |
| 評價回覆、訊息、報表、違規 | [seller/08](seller/08-reviews-and-messages.md)、[seller/09](seller/09-reports.md)、[seller/10-violations](seller/10-violations.md) |

---

## 4. 工會後台

本工會的店面與商品審核、訂單與統計查詢、商城內容、會員帳號核發。全部見 [union/](union/README.md)。工會管理者看不到其他工會的資料（403）。

---

## 5. 平台後台

平台人員有三種預設角色，**同一個畫面依角色顯示不同按鈕**，請依 `GET platform/me` 回傳的權限決定，不要「是管理員就全開」。權限對照表在 [platform/README](platform/README.md)。

| 功能 | 文件 |
| --- | --- |
| 營運總覽 | [platform/02](platform/02-overview.md) |
| **訂單與付款查詢**（2026-10-18 新增） | [platform/20](platform/20-orders.md)：`GET platform/orders`（`order_no` 主／子單號皆可）、`orders/{orderNo}`、`GET platform/payments`（可篩「已付款未對帳」）。三個平台角色都看得到，聯絡資料遮罩 |
| **稽核日誌**（2026-10-18 新增） | [platform/21](platform/21-audit-logs.md)：`GET platform/audit-logs`，只有 `audit.view`（預設僅平台管理者）；原值新值已遮罩 |
| 組織、會籍、帳號核發 | [platform/03](platform/03-organizations-and-members.md) |
| 店面、商品審核 | [platform/04](platform/04-shops.md)、[platform/05](platform/05-products-and-categories.md) |
| 系統參數、費率、運費、行事曆 | [platform/06](platform/06-settings.md) |
| 退款、對帳、撥款、調整單、拒付（財務） | [platform/07](platform/07-finance.md) |
| 違規、內容處置、個資申請 | [platform/08](platform/08-risk.md)、[platform/09](platform/09-moderation-and-privacy.md) |
| **營運告警**（2026-10-15 新增） | [platform/19](platform/19-operation-alerts.md)：`GET platform/operation-alerts`（可篩類別、未處理）、`PATCH …/{id}/acknowledge`。客服只看得到出貨類、財務看得到金流類 |
| PAYUNi 商家申請、證據、核實、憑證、付款設定、能力、查核 | [platform/10](platform/10-payuni-applications.md)～[platform/18](platform/18-payuni-grant-query.md)，多為平台內部作業 |

---

## 6. 已知缺口與待決事項

| 項目 | 對前端的影響 | 狀態 |
| --- | --- | --- |
| 付款後返回頁 | 需部署設定 `PAYUNI_FRONTEND_RETURN_URL`，否則停在 API 的 JSON 提示；2026-10-15 前已交付的付款表單導回時不帶 `order_no` | **已處理**（2026-10-15） |
| 物流追蹤號無法給買家查件 | 不要做外部查件連結 | 待改為超商／黑貓寄件編號 |
| 貨態更新有延遲 | 畫面標示「最後更新時間」較佳；目前回應沒有同步時間欄位 | 規劃中：開訂單時補查一次，並回傳 `logistics_synced_at` |
| 只支援信用卡一次付清 | 付款方式選單只放信用卡 | 第一階段範圍 |
| PAYUNi 只在 sandbox | 封測站以測試卡付款：`4147631000000001`（成功），到期日與末三碼任填 | 正式放行施工中 |
| 超商選店 API（`buyer/shops/{shop}/payuni-store-maps`） | **結帳不要用**，帶 `store_map_id` 結帳會 422 | 保留給日後重選門市使用 |

## 7. 2026-10-14 之後的異動

- 超商結帳不再接受任何門市欄位，門市改在 PAYUNi 付款頁選擇；付款成立後才出現在訂單上。
- 新增 `POST seller/orders/{orderNo}/payuni-label`；PAYUNi 物流訂單的確認出貨不帶物流商與單號。
- 新增營運告警兩支端點與三項權限（`operation_alert.all`／`.finance`／`.fulfillment`），權限總數 52。
- 系統參數 56 項（新增 `payuni_grant_overdue_days`、`payuni_expiry_precheck_grace_minutes`）。
- 待付款逾期：已交付 PAYUNi 表單的訂單會寬限後才取消，訂單可能在 `payment_deadline_at` 之後數分鐘才轉為已取消。
- 付款後前景返回可導回前端頁（`PAYUNI_FRONTEND_RETURN_URL`），網址帶 `order_no`。

## 8. 2026-10-18 之後的異動

- **新端點**：`POST auth/password/forgot`、`auth/password/reset`、`GET storefront/categories`、`platform/orders`、`platform/orders/{orderNo}`、`platform/payments`、`platform/audit-logs`。
- **token 有時限**：登入／註冊回應新增 `token_expires_at`、`token_idle_timeout_minutes`；逾時一律 401（見 §0 第 6 點）。
- **行為變更**：`storefront/products?category_id=` 給主分類時涵蓋其子分類（原為完全比對，點主分類會是空的）。
- **回應修正**：`platform/organizations*` 與開店申請詳情（賣家、工會、平台）原本會把**值為 null 的欄位整個拿掉**（如 `status_reason`、`submitted_at`、`decided_at`），現在一律輸出、值為 `null`。若前端曾用 `'key' in obj` 判斷，請改看值。清單視角不帶的 `pending_actions`、`approval_checklist` 仍然不帶。
- **回應補欄位**：`storefront/shops/{slug}` 新增 `opened_at`（文件早已列出）。
- 系統參數 62 項（新增 `password_reset_link_minutes` 與四個 `*_token_*_minutes`）。
- 文件補上原本漏寫的回應欄位（登入的 `membership.can_open_shop` 等、平台店面的 `open_violation_*`、買家訂單的 `invoice`／`cancel_request` 等）。之後回應範例與實際回應由整鏈自動比對，漂移即紅燈。

