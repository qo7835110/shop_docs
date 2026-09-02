# 04 訂單與付款

對應 spec B-02、§11.2、§11.3、§12.1、§14.2、§14.3。

## 一張主訂單，每家店一張子訂單

`order_main` 是**付款彙總**：一次結帳只付一次錢。`order_sub` 是**履約單位**：
出貨、退貨、評價都掛在它上面（§11.5）。

主訂單編號長成 `M2026082400019761`，子訂單是它加上 `-01`、`-02`。

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

### `fulfillment` —— 跨店出貨進度

主訂單層的彙總。跨店訂單常常只出了一半，而**主訂單狀態表達不出這件事** ——
它只有四格。所以進度另外給：

| 欄位 | 說明 |
| --- | --- |
| `total` / `shipped` / `pending_shipment` | 子訂單總數、已出貨數、待出貨數 |
| `is_partially_shipped` | 部分出貨 —— 前端該顯示「部分出貨」而不是「處理中」 |
| `is_fully_shipped` | 全部出貨 |

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
| `provider` | string | ✅ | 最長 40，金流商代碼 |
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

`status` 初始值為 `created`（§11.2）：代表付款請求已建立、尚未收到金流商結果。
第一階段只支援即時付款，因此不另設沒有實際進入點的 `processing`；後續由回調單向轉為
`succeeded` 或 `failed`，待付款逾期則轉為 `expired`。

### 這一支不會扣你的錢

它只建立一張付款單。實際付款由**第三方金流商**處理，平台記錄結果（§12.1）——
拿到 `payment_no` 之後前端該做的是導向金流商，然後等回調。

### 付款失敗什麼都不會發生

**一次失敗的付款嘗試不釋放庫存、不取消訂單**（§14.2）。你在 `payment_deadline_at`
之前再打一次這支，會拿到**一張新的付款單**（失敗是終態，不重用），再付一次即可。

只有兩件事會釋放預留：**待付款逾期**，或**你自己取消訂單**（下一節）。

第一階段只支援**即時付款方式**（信用卡、行動支付），沒有 ATM 轉帳或超商代碼。

---

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

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 不是你的訂單 |
| 422 | 子訂單狀態不允許（訊息會指出目前狀態） |

> 只有已送達的子訂單確認得了。`shipped` / `in_transit` 還在路上，
> `pending_pickup` 要先去超商領。
