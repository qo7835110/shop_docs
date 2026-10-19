# 05 訂單與出貨

對應 spec S-06、S-12、§11.2、§11.4、§13.3.1、§14.3。

## 你處理的是**子訂單**

一次結帳只包含**一家店面**，產生一張 `order_main`（付款彙總）與一張 `order_sub`
（履約單位，§F-06）。賣家端看到的全部是子訂單：出貨、退貨、撥款都掛在它上面（§11.5）。

`order_no` 長成 `M2026082400020680-01` —— 後綴看得出它屬於哪一張主訂單。

> 你只看得到**自己店面**的子訂單，也沒有任何一條賣家路由接受主訂單編號（§12.3-36）。
> 分開結帳之前成立的訂單可能在同一張主訂單上有別家店的子訂單，那一張同樣看不到。

### 子訂單狀態（§11.2 的完整列舉，共 12 種）

`awaiting_payment`、`awaiting_shipment`、`shipped`、`in_transit`、`pending_pickup`、
`pickup_returned`、`delivery_exception`、`delivered`、`completed`、`refund_pending`、
`cancelled`、`refunded`。

> **「爭議中」不是子訂單狀態** —— 那是退款單的子狀態（`disputed`）。

## 出貨才扣庫存

確認出貨會產生 `sale`（實際 −、預留 −）。在那之前商品還在你的倉庫裡，
訂單只占用了預留（§11.4）。因此**印單不等於出貨**，見第 3、4 條。

出貨、印單、更正單號、自取交付與取消共用主訂單 → 子訂單的鎖定順序，取得鎖後才判斷目前狀態。
取消先成立時，晚到的出貨／新印單回 422，不扣庫；出貨先成立時，不再走未出貨取消退款。
出貨的庫存、狀態、稽核與通知同交易提交，重送原出貨不重複扣庫或通知。
物流回呼也先取得同一主幹鎖，再重查事件與出貨狀態，避免與出貨反向卡鎖（§11.4／§14.3）。

買家已有待回應取消申請時，在回應期限內出貨會將該申請記為 `rejected`，附理由並通知買家；
期限已到或不明則回 422，須先處理取消，不允許靠搶先出貨避開逾期同意（§13.3）。

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

邊界含**期限那一秒本身**：期限是當日 23:59:59（§13.3.1），那一秒已經算逾期。
自動處置的排程用的是同一條判定，因此清單、待辦與「會不會被自動取消」三者
在任何一秒都給同一個答案 —— 不會出現排程已經取消了、而你的畫面說你沒逾期。

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
`rounding`、`calc_type`、`refund_behavior`、`effective_from` 與 `fee_rule_id`，放在 `rules` 底下（`payment_fee`、`platform_fee`、`union_share`）。
另帶 `beneficiary_organization_id`：工會分潤的受益組織，恆為**你的店所屬的組織**，與買家會籍無關（§13.5.2）。

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
>
> **付款頁物流訂單不走這一步**（Plan32-4，spec §S-12.3）：交付時已建立 PAYUNi 物流單的訂單，本端點、
> 自填物流商／單號的確認出貨與更正單號一律 422，請改用 [3a](#3a-payuni-出貨單付款頁物流訂單)。
> `payuni-c2c`／`payuni-home` 也不接受手動指定。32-4 之前建立、沒有 PAYUNi 物流單的舊訂單維持本流程。

---

## 3a. PAYUNi 出貨單（付款頁物流訂單）

```
POST seller/orders/{orderNo}/payuni-label
```

spec §S-12.3。物流商與物流序號由伺服器自原通知取得（付款頁選門市時 PAYUNi 回傳），**不接受用戶端提供**。
限每分鐘 10 次。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `ship_date` | string | ✅ | `YYYY-MM-DD`（台北日曆日）。超商不得早於今日；黑貓須晚於今日，且不得為週日或國定假日 |
| `delivery_date` | string | 黑貓 ✅ | `YYYY-MM-DD`，晚於 `ship_date`，不得為週日或國定假日 |
| `spec` | integer | 黑貓 ✅ | 包裹規格：1=60、2=90、3=120、4=150 |
| `label_mode` | integer | — | 超商標籤版型：1=A4（預設）、2=直立式 |

**回應 200**，`data` 為訂單（同 [2](#2-訂單詳情)），另含 `form`：`{action, method: "POST", fields}`。
`shipment.carrier` 為 `payuni-c2c` 或 `payuni-home`，`shipment.tracking_number` 為 PAYUNi 物流序號，
`shipment.status` 為 `label_created`。回應帶 `Cache-Control: private, no-store` 與 `Referrer-Policy: no-referrer`。

- **表單須由賣家瀏覽器以 Form POST 送出**：超商 `logistics/print_label` 由 PAYUNi 跳轉 7-ELEVEN 列印，
  商店不得自行連線取得；黑貓 `home_delivery/get_obt_number_pdf` 前景下載託運單 PDF。
  不要把 `form` 或其中密文寫入日誌、檔案或 session。
- **可重印**：再次呼叫沿用同一張出貨單，只產生新的列印表單；不新增出貨單。
- **印單 ≠ 出貨**：與 [3](#3-建立物流單號印單) 相同，不扣庫存。交寄後以 [4](#4-確認出貨) 確認出貨（不帶物流商／單號）。
- 422：未完成 PAYUNi 付款、此訂單沒有 PAYUNi 物流單、尚未取得物流序號、日期或規格不符；已有非 PAYUNi 出貨單時亦 422。

---

## 4. 確認出貨

```
POST seller/orders/{orderNo}/ship
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `carrier` | string\|null | — | 已登記的代碼；**已印單時可省略** |
| `tracking_number` | string\|null | — | 最長 64；同上 |
| `shipped_at` | date\|null | — | **不得晚於現在**（可回填，不可預填）（台北時間，見 [conventions](../conventions.md#時間)） |

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
