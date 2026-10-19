# 20 訂單與付款查詢

跨工會的訂單查詢 —— **唯讀**（§P-02、§15 角色表）。客服「可處理退貨爭議與查詢訂單，
不得執行退款、撥款或修改商品與庫存原始紀錄」；兩支端點都只需 `order.view`
（`platform_admin`／`platform_finance`／`platform_cs` 三個平台角色都有）。
任何處置都在各自的端點與權限上，不從這裡延伸：

- 付款查核 → [17 原付款查核](17-payuni-payment-query.md)（用詳情的 `payments[].payment_no`）
- 退款裁定與執行 → [07 金流作業](07-finance.md)（用詳情的 `sub_orders[].refunds[].refund_no`）

**聯絡資料一律遮罩**（§2.3：遮罩是輸出規則，平台人員也不例外）。客服要聯絡買家時走
[站內對話介入](09-moderation-and-privacy.md)，不需要電話全碼。品項**不含成本**——
成本「僅賣家自己可見」。

## 1. 訂單列表

```
GET platform/orders
```

需要 `order.view`。**分頁**，新到舊。以**主訂單**為單位：付款、對帳與退款的追查都從那一次結帳出發。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `order_no` | string | 最長 40。**同時比對主訂單與子訂單編號** —— 買家手上是主訂單編號，賣家手上是子訂單編號，接到哪一方的電話都查得到同一筆 |
| `status` | string | 主訂單狀態：`awaiting_payment`／`paid`／`completed`／`cancelled` |
| `sub_status` | string | 任一子訂單處於此狀態，值見 §11.2（如 `awaiting_shipment`、`refund_pending`、`delivery_exception`） |
| `organization_id` | integer | **賣方**店面所屬組織，不是買家的工會（§13.5.2） |
| `shop_id` | integer | 任一子訂單屬於此店面 |
| `buyer_member_id` | integer | 買家會員 |
| `created_from` | string | `Y-m-d`，成立日（含）起；**台北日**，見[共通約定](../conventions.md#時間) |
| `created_to` | string | `Y-m-d`，成立日（含）止；須不早於 `created_from` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得訂單列表。",
  "data": [
    {
      "order_no": "M2026101700001245",
      "status": "paid",
      "payable_amount": 510,
      "payment_deadline_at": "2026-10-17 10:30:00",
      "paid_at": "2026-10-17 10:12:41",
      "cancelled_at": null,
      "cancel_reason": null,
      "completed_at": null,
      "created_at": "2026-10-17 10:00:00",
      "buyer": { "member_id": 31, "organization_id": 2, "name": "陳○○" },
      "fulfillment": { "total": 1, "shipped": 0, "pending_shipment": 1, "is_partially_shipped": false, "is_fully_shipped": false },
      "sub_orders": [
        {
          "order_no": "M2026101700001245-01",
          "status": "awaiting_shipment",
          "goods_amount": 450,
          "shipping_fee": 60,
          "discount_amount": 0,
          "payable_amount": 510,
          "tax_amount": 21.4286,
          "invoice_tax_amount": 21,
          "shipping_method": "cvs_711",
          "ship_deadline_at": "2026-10-20 23:59:59",
          "shipment": null,
          "items": [ { "id": 88, "product_name": "工會紀念杯", "unit_price": 450, "quantity": 1, "subtotal": 450 } ],
          "shop": { "id": 4, "name": "職人小舖", "organization_id": 2 },
          "recipient": { "name": "陳○○", "phone": "0912***678", "address": "臺北市中正區****", "pickup_store_name": "7-ELEVEN 博愛門市" }
        }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "total": 1, "per_page": 20, "current_page": 1, "last_page": 1 } }
}
```

子訂單其他欄位（`reservation_expires_at`、`pickup_deadline_at`、`delivered_at`、`auto_complete_at`、
`return_window_ends_at`、`is_return_window_open`、`completed_at`、`first_completed_at`、`shipment.*`、
`items[].*`）與[賣家訂單](../seller/05-orders-and-shipping.md)同一套定義。品項另帶 `price_type`（`normal` \| `member`）與 `price_basis_organization_id`（會員價憑哪一個組織的會籍成立，§13.5.3；一般價為 `null`）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 401 | 未帶 token，或帶的是**會員** token（認證層即拒絕） |
| 403 | 平台人員沒有 `order.view` |
| 422 | `status`／`sub_status` 不在列舉內、日期格式錯誤、`created_to` 早於 `created_from` |

## 2. 訂單詳情

```
GET platform/orders/{orderNo}
```

需要 `order.view`。`{orderNo}` 是**主訂單編號**；手上只有子訂單編號時，先用列表的
`order_no` 篩選換成主訂單 —— 詳情只有一種單位，前端不必猜回來的是哪一種。
子訂單編號或不存在的編號回 404。

回應為列表單筆的全部欄位，另加：

| 欄位 | 說明 |
| --- | --- |
| `payments[]` | 這次結帳的**每一次**付款嘗試（舊到新）：`payment_no`、`provider`、`method`、`status`、`amount`、`paid_at`、`failure_reason`、`reconcile_status`、`reconciled_at`、`created_at`。不含金流商交易序號 —— 查核走 `payments/{paymentNo}/payuni-query` |
| `sub_orders[].refunds[]` | 該子訂單的退款單（舊到新）：`refund_no`、`status`、`reason_type`、`return_type`、`total_amount`、`requested_at`、`processed_at`、`closed_at` |
| `sub_orders[].fee_rule_snapshot` | 下單時快照的費率；撥款金額由它推導（§P-04），結構見[賣家訂單](../seller/05-orders-and-shipping.md#feerulesnapshot)。`beneficiary_organization_id` 是工會分潤的受益組織 —— 恆為**店面所屬組織**，與買家會籍無關（§13.5.2） |
| `sub_orders[].cancel_request` | 待回應的取消申請，沒有時為 `null`；欄位同[賣家訂單](../seller/05-orders-and-shipping.md) |

```json
{
  "success": true,
  "message": "取得訂單資料。",
  "data": {
    "order_no": "M2026101600001292",
    "status": "completed",
    "payments": [
      { "payment_no": "Y2026101600001805", "provider": "payuni", "method": "credit_card", "status": "succeeded", "amount": 510, "paid_at": "2026-10-16 09:01:12", "failure_reason": null, "reconcile_status": "matched", "reconciled_at": "2026-10-17 14:30:05", "created_at": "2026-10-16 09:00:30" }
    ],
    "sub_orders": [
      {
        "order_no": "M2026101600001292-01",
        "status": "refunded",
        "fee_rule_snapshot": { "rules": { "…": "見賣家訂單文件" } },
        "cancel_request": null,
        "refunds": [
          { "refund_no": "R2026101700001540", "status": "processed", "reason_type": "return_defect", "return_type": "defect", "total_amount": 510, "requested_at": "2026-10-17 09:00:00", "processed_at": "2026-10-18 11:20:00", "closed_at": null }
        ]
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 401 | 未帶 token，或帶的是會員 token |
| 403 | 平台人員沒有 `order.view` |
| 404 | 編號不存在，或給的是子訂單編號 |

## 3. 付款清單

```
GET platform/payments
```

需要 `order.view`（plans/35 35-7）。**分頁**，新到舊。付款原本只能逐筆以 `payments/{paymentNo}/…` 查核；
這一支回答「有哪些」，例如**已付款、還沒對帳**的那幾筆。查核、採用與對帳仍在原本的端點與權限上。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `created`／`succeeded`／`failed`／`expired` |
| `reconcile_status` | string | `pending`／`matched`／`mismatched`。與 `status` 分開：付款成立不等於對帳相符（§12.1） |
| `provider` | string | 最長 30，如 `payuni` |
| `order_no` | string | 最長 40，**主訂單**編號（付款掛在主訂單上） |
| `created_from` | string | `Y-m-d`，付款嘗試建立日（含）起；**台北日** |
| `created_to` | string | `Y-m-d`，建立日（含）止；須不早於 `created_from` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

每一列與[訂單詳情](#2-訂單詳情)的 `payments[]` 同一個形狀，另加 `order_no`（接 `platform/orders/{orderNo}`）與 `fee`（金流手續費）。
同樣**不含金流商交易序號**。

```json
{
  "success": true,
  "message": "取得付款清單。",
  "data": [
    {
      "order_no": "M2026101600001292",
      "payment_no": "Y2026101600001805",
      "provider": "payuni",
      "method": "credit_card",
      "status": "succeeded",
      "amount": 510,
      "paid_at": "2026-10-16 09:01:12",
      "failure_reason": null,
      "reconcile_status": "pending",
      "reconciled_at": null,
      "created_at": "2026-10-16 09:00:30",
      "fee": 14
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 401 | 未帶 token，或帶的是會員 token |
| 403 | 平台人員沒有 `order.view` |
| 422 | `status`／`reconcile_status` 不在列舉內、日期格式錯誤、`created_to` 早於 `created_from` |
