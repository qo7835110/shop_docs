# 04 訂單與付款

對應 spec B-02、§11.2、§11.3、§12.1、§14.2、§14.3。

## 一次結帳 = 一家店面 = 一張主訂單 ＋ 一張子訂單

`order_main` 是**付款彙總**：一次結帳只付一次錢。`order_sub` 是**履約單位**：
出貨、退貨、評價都掛在它上面（§11.5）。

**不同店面分開結帳、分別付款**（§F-06），因此新訂單的 `sub_orders` 恆只有一個元素，
子訂單編號恆為 `-01`。主訂單編號長成 `M2026082400019761`，子訂單是它加上 `-01`。

> `sub_orders` 仍是陣列：**既有的多子訂單仍讀得到**（`-01`、`-02`……）。
> 分開結帳只改變新訂單怎麼成立，不回頭改寫已成立的訂單（§11.3）。

### 主訂單只有四個狀態（§11.2）

```
awaiting_payment ──付款成功──> paid ──全部子訂單走完──> completed
       └────────逾期或買家取消────────> cancelled
```

**`cancelled` 的意思是「從來沒有收到錢」。** 付款成功之後主訂單一定會走到
`completed`，**即使每一張子訂單都被全額退款** —— 否則對帳時分不出
「沒收到錢」與「收了又退」（§14.2）。要看退款狀況請讀
[退款單](05-returns-and-refunds.md#5-退款單列表)，不是主訂單狀態。

---

## 1. 訂單列表

```
GET buyer/orders
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `awaiting_payment` \| `paid` \| `completed` \| `cancelled` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得訂單列表。",
  "data": [
    {
      "order_no": "M2026082400019761",
      "status": "completed",
      "payable_amount": 260,
      "payment_deadline_at": "2026-08-24 11:04:47",
      "paid_at": "2026-08-24 10:34:47",
      "cancelled_at": null,
      "cancel_reason": null,
      "created_at": "2026-08-24 10:34:47",
      "fulfillment": {
        "total": 1,
        "shipped": 0,
        "pending_shipment": 0,
        "is_partially_shipped": false,
        "is_fully_shipped": false
      },
      "sub_orders": [
        {
          "order_no": "M2026082400019761-01",
          "status": "refund_pending",
          "goods_amount": 160,
          "shipping_fee": 100,
          "discount_amount": 0,
          "payable_amount": 260,
          "tax_amount": 7.619,
          "invoice_tax_amount": 8,
          "shipping_method": "home",
          "reservation_expires_at": null,
          "ship_deadline_at": "2026-08-27 23:59:59",
          "pickup_deadline_at": null,
          "delivered_at": "2026-08-24 10:34:47",
          "auto_complete_at": "2026-08-31 23:59:59",
          "return_window_ends_at": "2026-08-31 23:59:59",
          "is_return_window_open": true,
          "completed_at": "2026-08-24 10:34:47",
          "first_completed_at": "2026-08-24 10:34:47",
          "shipment": {
            "carrier": "demo-express",
            "tracking_number": "TN-DEMO-DISPUTABLE",
            "status": "delivered",
            "label_created_at": null,
            "shipped_at": "2026-08-24 10:34:47",
            "delivered_at": "2026-08-24 10:34:47",
            "last_event_at": "2026-08-24 10:34:47",
            "correction_reason": null
          },
          "items": [ { "…": "見下" } ],
          "shop": { "id": 1, "name": "示範開業店面" },
          "recipient": { "…": "見下" }
        }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 14, "last_page": 1 } }
}
```

### `fulfillment` —— 出貨進度

主訂單層的彙總。**主訂單狀態表達不出履約進度** —— 它只有四格（§14.2）。
所以進度另外給：

| 欄位 | 說明 |
| --- | --- |
| `total` / `shipped` / `pending_shipment` | 子訂單總數、已出貨數、待出貨數 |
| `is_partially_shipped` | 部分出貨 —— 前端該顯示「部分出貨」而不是「處理中」 |
| `is_fully_shipped` | 全部出貨 |

> **新訂單不會出現「部分出貨」。** 一張子訂單只有「未出」與「已出」兩種，
> 一張子訂單也不分批出貨（§B-02）。`is_partially_shipped` 為 `true` 只可能
> 發生在分開結帳之前成立的多子訂單上；那些訂單還在，畫面仍要顯示得出來。
>
> 買家同時向兩家店買東西，得到的是**兩張各自獨立的主訂單**，各自顯示履約進度——
> 不得把它們合成一個「部分出貨」狀態（§B-02、§12.3-13）。

### 子訂單的四組時間

| 欄位 | 單位 | 意義 |
| --- | --- | --- |
| `reservation_expires_at` | 分鐘 | **待付款階段**的庫存預留到期；付款後為 `null` |
| `ship_deadline_at` | 工作天 | 賣家的出貨期限。逾時**自動取消並全額退款** |
| `pickup_deadline_at` | 日曆天 | 超商保管期限 |
| `auto_complete_at` | 日曆天 | 送達後自動確認收貨 |
| `return_window_ends_at` | 日曆天 | **退貨期限** |

**`auto_complete_at` 與 `return_window_ends_at` 是兩條線。** 訂單自動完成之後，
你依然可以在退貨期限前申請退貨（§14.3）。`is_return_window_open` 是後端算好的
布林值，**只看時間、不看訂單是否已完成** —— 直接用它。

### `items[]` 與 `recipient`

```json
"items": [
  {
    "id": 7,
    "product_id": 1,
    "product_sku_id": 1,
    "product_name": "示範包裝食品（多規格）",
    "spec_name": "小包裝 250 公克",
    "condition": "new",
    "unit_price": 160,
    "price_type": "member",
    "price_basis_organization_id": 2,
    "quantity": 1,
    "subtotal": 160
  }
],
> **`items[].id` 是申請退貨與寫評價的鑰匙。**
>
> 兩條路徑都以訂單品項為單位：
> `POST buyer/sub-orders/{orderNo}/return-requests` 的 `items[].order_item_id`（§B-03）、
> `POST buyer/order-items/{id}/review`（§B-04.2）。
> 因此訂單詳情的每一列旁邊都放得下「退貨」與「評價」兩顆按鈕。
>
> 這一條是使用者旅程模擬（plans/24）補上的 —— 在此之前買家端沒有任何端點
> 提供得了 `order_item_id`，退貨那條路在前端根本組不出請求。

"recipient": {
  "name": "示範買家",
  "phone": "0912345678",
  "postal_code": "10041",
  "address": "臺北市中正區重慶南路一段 122 號",
  "pickup_store_code": null,
  "pickup_store_name": null
}
```

商品名稱、規格與價格都是**下單當下的快照**：賣家事後改名或改價，
這張訂單賣的仍然是當初那個名字與那個價格（§3.5）。

`recipient` 是**你自己填的**收件資訊，因此不遮罩。

### `invoice` 與 `cancel_request`

| 欄位 | 說明 |
| --- | --- |
| `invoice` | 結帳時填的發票資訊：`invoice_type`、`buyer_tax_id`、`invoice_title`、`carrier`、`donation_code`，以及 `shop_invoice_capability`（**下單當下**店面的開立能力，§13.6.2）。發票由賣家開立，不是平台。 |
| `cancel_request` | **現在等著回應**的取消申請：`id`、`requested_by`（`buyer` \| `seller`）、`reason`、`status`、`response_deadline_at`（工作天，§14.3）。沒有待回應的申請時為 `null`。`requested_by` 是 `seller` 時，回應用 `POST buyer/cancel-requests/{id}/response`。 |

---

## 2. 訂單詳情

```
GET buyer/orders/{orderNo}
```

`{orderNo}` 是**主訂單編號**（不含 `-01` 後綴）。`data` 形狀與列表單筆相同。

別人的訂單編號回 **403**（`OrderMainPolicy::view`）。

---

## 3. 建立付款單

```
POST buyer/orders/{orderNo}/payments
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `provider` | string | ✅ | 已啟用的金流商：`payuni`；`demo` 只在 local／testing 或部署明示 `PAYMENT_DEMO_ENABLED=true` 時接受。未知或已關閉的金流商回 422，不建立付款 |
| `method` | string | ✅ | `credit_card` \| `mobile_payment` |

### 回應 201

```json
{
  "success": true,
  "message": "付款單已建立，請於期限內完成付款。",
  "data": {
    "payment_no": "Y2026082400019175",
    "provider": "demo-pay",
    "method": "credit_card",
    "amount": 260,
    "status": "created",
    "order_no": "M2026082400019761",
    "payment_deadline_at": "2026-08-24 11:04:47"
  },
  "errors": null,
  "meta": null
}
```

**`amount` 由後端取自訂單**，不由你提供（§11.3）。請求裡沒有金額欄位。

`status` 初始值為 `created`（§11.2）：本地付款單已建立。既有 demo 模型沒有 `processing`，
可信成功／明確失敗後為 `succeeded` 或 `failed`，待付款逾期由排程轉為 `expired`。
未知通知只留下 `unconfirmed` 事件，付款維持 `created`；成功通知缺少可核對金額時亦不入帳。
此時同意圖請求仍回原付款，不建立另一筆，也不發送「失敗可重試」通知。
這是本地付款防護，不是 PAYUNi ACK／UNKNOWN 最終性或安全重送契約；本地逾期也不證明遠端沒有收款。

### 重播與期限

- 同張訂單已有 `created` 付款時，相同 provider／method 回原付款單（仍為 201），不改寫紀錄；任一不同回 422，不能替換在途嘗試的供應商或工具。
- 以鎖後重讀的 `payment_deadline_at` 為準；等於或超過期限即 422，包含重播或前次失敗後重新發起，即使排程還沒取消訂單也一樣。
- 此入口拒絕逾期不自行取消訂單、釋放庫存或改寫付款；先前付款的晚到回調仍由既有終態／差異流程處理。
- 明確失敗且仍在期限內，才可建立新付款並重新選擇允許的工具。已付款、已完成或已取消的訂單仍回 422。

### 這一支不會扣你的錢

它只建立一張本地付款單，不外呼金流；PAYUNi 表單須另走下方的買家一次交付端點。
實際付款由**第三方金流商**處理，平台記錄結果（§12.1）。
`payment_no` 是本地嘗試識別，不是可用支付網址，也不保證供應商永久防重或允許未知結果重送。

### 付款失敗什麼都不會發生

**一次失敗的付款嘗試不釋放庫存、不取消訂單**（§14.2）。你在 `payment_deadline_at`
之前再打一次這支，會拿到**一張新的付款單**（失敗是終態，不重用），再付一次即可。

只有兩件事會釋放預留：**待付款逾期**，或**你自己取消訂單**（下一節）。

第一階段只支援**即時付款方式**（信用卡、行動支付），沒有 ATM 轉帳或超商代碼。

---

## 3b. PAYUNi 一次表單交付

```
POST buyer/orders/{orderNo}/payments/{paymentNo}/payuni-form
```

原下單會員 token，不接受 body／query 參數；不得指定金額、MerID、金鑰或回呼網址。
每分鐘 10 次；匿名／平台 token 401、其他買家 403、付款不屬該訂單 404、不符資格 422。
本端點所有回應均 no-store。只支援 sandbox 信用卡一次付清，仍需 `PAYUNI_ENABLED=true`、
受控 HTTPS origin、已核實綁定、原版本憑證及新鮮能力／付款設定審查；缺證據不放行。

首次成功 200：`data` 包含 `payment_no`、`delivery_status=prepared` 及 `form`。
`form` 為 `action`（固定 PAYUNi sandbox UPP）、`method=POST`、`fields`；
fields 僅 `MerID`、`Version`、`EncryptInfo`、`HashInfo`、`IsPlatForm`，不是 Key／IV。
前端只可在當次記憶體組裝表單並明示送至 action；不得存 localStorage、快取、分析平台或一般日誌。
服務只交付表單，不自行向 PAYUNi 外送；可執行表單亦不得放入除錯回報。

重播 200：`delivery_status=already_prepared`、`form=null`。即使收款停用、期限經過或回條遺失，
也不重建表單／改短碼。請以原訂單 payment-status 讀回；需要恢復時由受控原交易查核處理。
準備或交付中斷可保留原 session／占用；500 不表示供應商未收到，不可自動重新付款。

`NotifyURL` 隨請求指向原 payment_no 的驗簽接收器；前景 Return 與 Back 綁原訂單編號。部署設定
`PAYUNI_FRONTEND_RETURN_URL` 時以 303 導回該前端頁並附 `?order_no=`，否則回靜態 JSON 提示；
兩者都不回顯輸入、不自動入帳、不發 M3、不採用任意導向網址（見 [前景返回](../webhooks/payuni-upp-return.md)）。
前端頁須以會員授權讀取本地 payment-status，不能將返回視為付款完成。

## 3a. 查核本地付款

```http
GET buyer/orders/{orderNo}/payment-status
```

§B-02／§P-10／§12.1：僅限下單會員。建立付款單的回應遺失時，用訂單編號找回
本地最新一次付款，**不需要 payment_no**。無 body／查詢參數；成功為 200，
`Cache-Control: private, no-store`。無會員 token 或使用平台 token 為 401，
他人訂單為 403，不存在訂單為 404。

`data` 固定含 `order_no`、`order_status`、`payment_deadline_at`、`payment`。
尚未建立付款時 `payment: null`；否則只回按建立順序最新的嘗試：

| 回應欄位 | 型別 | 意義 |
| --- | --- | --- |
| `payment_no` | string | 本地付款編號 |
| `provider` / `method` | string | 原供應商與工具 |
| `amount` | integer | 原付款金額（TWD），不因退款重新計算 |
| `status` | string | 本地 `created`／`succeeded`／`failed`／`expired` |
| `created_at` | string | 原建立時間 |
| `paid_at` | string 或 null | 本地確認成功時間 |
| `payuni_prepared_at` | string 或 null | 該次 PAYUNi session 準備時間 |
| `payuni_delivery_reserved_at` | string 或 null | 該 session 交付槽占用時間，不代表瀏覽器收到表單 |
| `payuni_page_expires_at` | string 或 null | 原支付頁截止時間，不代表遠端付款失敗 |
| `payuni_query` | object 或 null | 此次付款 session 最新一筆本地查核工作摘要；未有工作為 null |

`payuni_query` 僅含三個欄位：

| 回應欄位 | 型別 | 意義 |
| --- | --- | --- |
| `outcome` | string | `pending`＝工作已保存但無觀察，可能尚未送出、在途或中斷，不保證背景仍在執行；`unknown`＝有觀察但結果未確認；`matched`＝觀察列與原付款相符，**不是已付款** |
| `started_at` | string | 該工作保存的 UTC 開始時間 |
| `recorded_at` | string 或 null | 該工作觀察的 UTC 保存時間；pending 為 null |

摘要與訂單、付款同一 SQL 快照，按此 session 的工作建立序號取最新，不以收到觀察的先後
或時間戳排序。舊工作晚到不會覆蓋較新工作的 pending／unknown；前次付款及他人工作的結果不混入。
`matched` 可能包含供應商失敗列或處理中的資料，不能據此顯示付款成功；
畫面仍以本地 `payment.status`／`paid_at` 顯示是否已確認入帳。任何摘要都不是付款／重付許可。

時間格式與其他 API 相同，`YYYY-MM-DD HH:mm:ss`（UTC）。三個 PAYUNi 時間可為 null；
目前公開建立付款單入口尚未準備或交付支付頁，不可把 null 當成付款許可。
已存在的歷史時間不受目前憑證／開關影響，讀回不需秘密。

此端點**只讀本地資料**，不是即時查詢 PAYUNi，不會新增嘗試、建立表單或變更付款／
訂單／庫存；期限經過也不會自行轉失敗。重新整理可再讀同一端點，不應自動重送付款。
`created` 不代表已提交到金流，`expired` 也不證明遠端未收款。
付款資格、外部結果核實與對帳另行處理；不回傳可付款旗標、內部失敗原因、費用、
外部商家／交易識別、憑證版本或可執行表單。
查核摘要亦不含工作鍵、內部 ID、供應商狀態或原始錯誤。重新整理只讀現存摘要，
不啟動或重送 M3；平台財務另有受控的原付款查核入口，買家不得使用，也不應以輪詢此 API 期待產生新工作。

## 4. 取消尚未付款的訂單

```
POST buyer/orders/{orderNo}/cancel
```

`{orderNo}` 是**主訂單編號**。付款成功前不需要任何人同意，因此這是取消本身，
不是取消申請（§B-06、§14.2）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500。每一次取消都必須留下原因（§3.4） |

回應 200，`data` 為取消後的主訂單，`message`：`訂單已取消，預留庫存已釋出。`

主訂單轉 `cancelled`，**每一張**子訂單一併轉 `cancelled`，各自的預留全部釋放
（庫存動作是 `release_reservation`；`actual` 從未因這張訂單減少過，所以沒有「回補」
這回事，§S-05.5）。

### 只做整張，不做部分

那一筆付款是整張主訂單的單一彙總（§11.3），留下半張訂單等於留下一筆對不上的
應付金額。想只取消其中一家店，得等付款成功之後走
[05 退貨與退款](05-returns-and-refunds.md) 的**子訂單取消申請**，且需賣家回應
或逾 `buyer_cancel_response_days` 視為同意（§14.3）。

### 與逾期取消是同一段程式

不按也可以：`payment_deadline_at` 過了排程會做同一件事。兩條路的狀態轉換與
庫存釋放完全相同，因此不會出現「自己取消」與「逾期取消」結果不一樣的訂單。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 不是你的訂單 |
| 404 | 查無此訂單編號 |
| 422 | 缺少 `reason` |
| 422 | 訂單已不是待付款（已付款、已完成或已取消；訊息會指出目前狀態） |

> 已付款的訂單打這一支會拿到 422，而不是靜默的成功 —— 送出取消卻拿到一張
> 仍然有效的訂單，畫面沒有第二次機會說明。

---

## 5. 確認收貨

```
POST buyer/sub-orders/{orderNo}/confirm-receipt
```

`{orderNo}` 是**子訂單編號**（含 `-01`）。無請求 body。

回應 200，`data` 為該子訂單，`message`：`已確認收貨。`

### 提前確認不會縮短退貨期限

你可以在送達當日就按確認收貨，子訂單即刻轉 `completed`。但
**退貨期限仍然自送達日起算**，不因訂單完成而提前關閉（§14.3）——
`is_return_window_open` 仍是 `true`。

不按也沒關係：`auto_complete_at` 到了會自動完成。

確認收貨與建立退貨申請共用主訂單鎖，依鎖後狀態判斷。退貨先成立時，
晚到的收貨請求回 422，不覆寫 `refund_pending`；收貨先成立也不會關閉退貨窗口。
重送已完成的收貨請求回 422，不重寫首次完成時間或增加稽核。
排程另重查鎖後 `auto_complete_at`，期限未到或已進退貨時略過（§14.3）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 不是你的訂單 |
| 422 | 子訂單狀態不允許（訊息會指出目前狀態） |

> 只有已送達的子訂單確認得了。`shipped` / `in_transit` 還在路上，
> `pending_pickup` 要先去超商領。
