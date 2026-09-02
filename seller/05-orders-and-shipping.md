# 05 訂單與出貨

對應 spec S-06、S-12、§11.2、§11.4、§13.3.1、§14.3。

## 你處理的是**子訂單**

一次結帳產生一張 `order_main`（付款彙總）與**每家店一張** `order_sub`（履約單位）。
賣家端看到的全部是子訂單：出貨、退貨、撥款都掛在它上面（§11.5）。

`order_no` 長成 `M2026082400020680-01` —— 後綴看得出它屬於哪一張主訂單。

### 子訂單狀態（§11.2 的完整列舉，共 12 種）

`awaiting_payment`、`awaiting_shipment`、`shipped`、`in_transit`、`pending_pickup`、
`pickup_returned`、`delivery_exception`、`delivered`、`completed`、`refund_pending`、
`cancelled`、`refunded`。

> **「爭議中」不是子訂單狀態** —— 那是退款單的子狀態（`disputed`）。

## 出貨才扣庫存

確認出貨會產生 `sale`（實際 −、預留 −）。在那之前商品還在你的倉庫裡，
訂單只占用了預留（§11.4）。因此**印單不等於出貨**，見第 3、4 條。

---

## 1. 訂單列表

```
GET seller/orders
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | 上列 12 種之一；**不合法的值回 422** |
| `overdue` | boolean | — | **只列已逾出貨期限的**。要一起帶 `status=awaiting_shipment` |
| `pending_cancel_response` | boolean | — | **只列等你回應取消申請的** |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

### 兩個篩選各對應總覽的一個待辦

    ?overdue=1                  ⇄ todo.overdue_shipment
    ?pending_cancel_response=1  ⇄ todo.pending_cancel_response

兩邊用同一組條件（§S-02）。**數得出來卻列不出來的待辦比沒有待辦更糟** ——
它讓人以為自己已經看過了。

`overdue` 不預設 `status`，要自己一起帶。綁死的話「已出貨但當初逾期」
這種查詢就再也做不到，而違規記錄的查證需要它（§P-07）。

`pending_cancel_response` 只看**買家提出**的：你自己提的那些在等買家回應，
不是你的待辦。

### 回應 200

```json
{
  "success": true,
  "message": "取得店面訂單列表。",
  "data": [
    {
      "order_no": "M2026082400020680-01",
      "order_main_no": "M2026082400020680",
      "status": "awaiting_shipment",
      "cancel_request": {
        "id": 4,
        "requested_by": "buyer",
        "reason": "尺寸下錯了",
        "status": "pending",
        "response_deadline_at": "2026-08-28 23:59:59"
      },
      "goods_amount": 160,
      "shipping_fee": 100,
      "discount_amount": 0,
      "payable_amount": 260,
      "tax_amount": 12.381,
      "invoice_tax_amount": 12,
      "shipping_method": "home",
      "reservation_expires_at": null,
      "ship_deadline_at": "2026-08-27 23:59:59",
      "pickup_deadline_at": null,
      "delivered_at": null,
      "auto_complete_at": null,
      "return_window_ends_at": null,
      "is_return_window_open": false,
      "completed_at": null,
      "first_completed_at": null,
      "shipment": null,
      "items": [
        {
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
      "recipient": {
        "name": "示範買家",
        "phone": "0912345678",
        "postal_code": "10041",
        "address": "臺北市中正區重慶南路一段 122 號",
        "pickup_store_code": null,
        "pickup_store_name": null
      },
      "invoice": {
        "invoice_type": null,
        "buyer_tax_id": null,
        "invoice_title": null,
        "carrier": null,
        "donation_code": null,
        "shop_invoice_capability": "none"
      },
      "fee_rule_snapshot": { "rules": { "…": "見下" } }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 19, "last_page": 1 } }
}
```

### `recipient` **不遮罩**

這是賣家視角與工會、平台視角最大的差別：**你要寄東西給他，所以看得到完整的姓名、
電話與地址**（工會端看到的是遮罩值，§U-01）。

隨之而來的義務：這份資料只能用於履約。挪作行銷或轉出站外都是違規（§2.3、§X-03.6）。

### 三組金額欄位

| 欄位 | 說明 |
| --- | --- |
| `payable_amount` | 應付金額 = 商品小計 + 運費 − 折扣。**付款成功後凍結**，全額退款也不會變小。 |
| `tax_amount` | 由商品小計反推的浮點數，**四位小數是權威值**（§13.4.1），**不參與應付加總**。 |
| `invoice_tax_amount` | **發票上要寫的整數元稅額**。捨入由後端做一次 —— 開立人是你（§13.6），讓每個賣家自己捨入會出現不只一個「正確」答案。 |
| `discount_amount` | 第一階段**恆為 0**（優惠券、點數、購物金只保留欄位不啟用）。畫面可直接不顯示。 |
| `cancel_request` | 現在等著回應的取消申請；**多數訂單是 `null`**。有值時帶得出 `POST seller/cancel-requests/{id}/response` 要用的整數 `id` —— 那個 id 沒有別的地方拿得到。 |
| `cancel_request.requested_by` | `buyer`（等你回應）\| `seller`（你提的，等買家回應）。 |
| `cancel_request.response_deadline_at` | 回應期限，`buyer_cancel_response_days`（**工作天**，§13.3.1）。逾期有預設結果 —— 看得到期限才知道要不要急。 |

### 四組時間欄位

| 欄位 | 單位 | 說明 |
| --- | --- | --- |
| `ship_deadline_at` | **工作天**推算 | 出貨期限。逾時**自動取消並全額退款**，且記 3 點違規（§14.3） |
| `pickup_deadline_at` | 日曆天 | 超商保管期限；非超商取貨為 `null` |
| `auto_complete_at` | 日曆天 | 送達後自動確認收貨 |
| `return_window_ends_at` | 日曆天 | 退貨期限 |

`auto_complete_at` 與 `return_window_ends_at` **並列且互不影響**：訂單自動完成之後，
買家依然可以在退貨期限前申請退貨。`is_return_window_open` 是後端算好的布林值，
**只看時間、不看訂單是否已完成** —— 直接用它，不要自己比日期。

`first_completed_at` 是**首次**完成時間；退貨結案會改寫 `completed_at`，這一欄不動（§11.1）。

### `items[]`

`price_type` 說明成交價是原價還是會員價；`price_basis_organization_id` 是
「會員價憑哪一個組織的會籍成立」—— 跨工會的會員價爭議看這一欄。

### `fee_rule_snapshot`

下單當下的費率快照（金流費、平台費、工會分潤），每一項含 `rate`、`basis`、
`rounding`、`calc_type`、`refund_behavior`、`effective_from` 與 `fee_rule_id`。

**平台事後調整費率不會影響這張訂單** —— 撥款金額由這份快照推導（§P-04）。
前端如果要解釋「為什麼撥款是這個數字」，答案在這裡加上[撥款單](07-payouts.md)。

### `invoice`

買家結帳時填的發票資訊。`shop_invoice_capability` 是**你的店**的開立能力，
一併回傳是為了讓畫面能解釋「為什麼買家不能選三聯式」。

---

## 2. 訂單詳情

```
GET seller/orders/{orderNo}
```

`{orderNo}` 是**子訂單編號字串**（例如 `M2026082400020680-01`），不是數字 ID。
`data` 形狀與列表單筆**完全相同**。

---

## 3. 建立物流單號（印單）

```
POST seller/orders/{orderNo}/shipping-label
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `carrier` | string | ✅ | 已登記的物流商代碼 |
| `tracking_number` | string | ✅ | 最長 64 |

**回應 201**，`data` 為訂單，`message`：`已建立物流單號，尚未出貨。`

### 印單 ≠ 出貨

這一步只寫 `shipment.label_created_at`，**不扣庫存、不改子訂單狀態、不啟動配送時效**。
`label_created_at` 有值而 `shipped_at` 為 `null`，代表單號印出來了但商品還沒交運 ——
兩個時間戳並列就是為了表達這段期間（§11.2）。

> `carrier` 必須是**已登記**的代碼，否則這張出貨單的物流回調永遠對不上（§S-12）。
> 未登記的代碼在這一步就回 422 —— 比等到交運後才發現便宜得多。
> 開發環境目前登記的是 `demo-express`（宅配）與 `demo-store`（超商）。

---

## 4. 確認出貨

```
POST seller/orders/{orderNo}/ship
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `carrier` | string\|null | — | 已登記的代碼；**已印單時可省略** |
| `tracking_number` | string\|null | — | 最長 64；同上 |
| `shipped_at` | date\|null | — | **不得晚於現在**（可回填，不可預填） |

**回應 201**，`data` 為訂單（`status` 為 `shipped`），`message`：`已確認出貨。`

這一步才是真正的出貨：

- 產生 `sale` 異動 —— **實際庫存在此扣減**（§11.4）。
- 子訂單 `awaiting_shipment → shipped`。
- 開始計算配送時效。

`shipped_at` 允許回填是因為賣家常在實際交運之後才登錄；但**不接受未來時間** ——
那會讓出貨期限的判定失真。

> **冪等**：對已出貨的子訂單重送會得到既有結果，不會扣第二次庫存
> （`shipment` 對 `order_sub_id` 有唯一鍵）。

---

## 5. 更正物流單號

```
PATCH seller/orders/{orderNo}/shipment
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `carrier` | string\|null | — | 已登記的代碼 |
| `tracking_number` | string\|null | — | 最長 64 |
| `reason` | string | ✅ | 最長 255 |

回應 200，`message`：`物流單號已更正。`

`reason` **必填**：更正後買家的追蹤連結會變，而客服在爭議時需要知道為什麼變。
更正原因會出現在 `shipment.correction_reason`，**買家也看得到** ——
買賣雙方讀的是同一張出貨單（§3.5）。

---

## 6. 自取交付

```
POST seller/orders/{orderNo}/handover
```

無請求 body。回應 200，`message`：`已登錄自取交付。`

用於 `self_pickup`（店面自取）與 `union_pickup`（工會自取）——
**這兩種配送方式沒有物流商，因此沒有回調可以推進狀態**，必須由你登錄。

登錄後等同送達，開始跑確認收貨與退貨期限。

> 宅配與超商取貨**不要打這一支** —— 它們由物流回調推進。
