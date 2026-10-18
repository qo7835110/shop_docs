# 06 退貨、取消與退款

對應 spec S-07、B-03、B-06、§14.3、§14.4、§14.5、§14.7。

取消申請與回應採主訂單 → 子訂單 → 申請鎖後判斷，決定、庫存釋放、退款與通知同交易。
已回應的申請重播回 422，不可翻案；出貨先成立後不能再同意未出貨取消。
在回應期限內出貨會拒絕買家的待回應取消並通知；逾期後不得用出貨繞過預設同意（§13.3）。

## 你處理退貨，平台財務核實退款

賣家在這一章能做的是**回應與驗收**；實際退款由平台財務執行（§15.3）。
款項由金流業者處理，平台不保管或自行匯出交易款項；既有 demo 入口僅供本地模擬。
因此這裡沒有任何金額欄位可填 —— 退款金額在建單當下就由後端依 §14.5 算定並凍結。

## 退款單的兩條路徑（§14.4）

```
需退貨： pending → approved → return_received → inspected → processing → processed
免退貨： pending → processing → processed
```

- `disputed` 是**退款單的子狀態**，不是子訂單狀態。
- 終態只有 `processed` 與 `closed`。
- **你拒絕退貨申請（`rejected`）會自動轉入 `disputed`**，由平台客服裁定 ——
  買家不需要再按一次「提出爭議」（§12.3-81）。因此 `rejected` 是**過渡狀態，不是終態**。

## 退貨運費由你負擔

**不自退款金額中扣除**（§14.5.1）。這是規則不是設定，端點上沒有「扣除運費」的欄位。
在對話中要求買家分擔會被記 3 點違規（`return_shipping_shifted`）。

反過來說：退貨運費**不進撥款算式** —— 你與物流業者直接結算，平台未經手也無紀錄
（§13.2.1）。報表的毛利表會明講這一點。

---

## 1. 退款單列表

```
GET seller/refunds
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | `pending`／`approved`／`rejected`／`return_received`／`inspected`／`disputed`／`processing`／`processed`／`closed` |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得退款單列表。",
  "data": [
    {
      "refund_no": "R2026082400005194",
      "order_sub_no": "M2026082400019077-01",
      "reason_type": "return_defect",
      "reason_label": "瑕疵／錯發／短少",
      "requires_return": true,
      "return_type": "defect",
      "status": "approved",
      "items": [
        {
          "order_item_id": 19,
          "product_sku_id": 1,
          "product_name": "示範包裝食品（多規格）",
          "spec_name": "小包裝 250 公克",
          "unit_price": 160,
          "returned_quantity": 1,
          "refund_amount": 160
        }
      ],
      "goods_amount": 160,
      "shipping_amount": 100,
      "total_amount": 260,
      "requested_at": "2026-08-24 02:19:28",
      "approved_at": "2026-08-24 02:19:29",
      "return_received_at": null,
      "inspected_at": null,
      "dispute_deadline_at": null,
      "processed_at": null,
      "closed_at": null,
      "reject_reason": null,
      "closed_reason": null,
      "dispute_reason": null,
      "dispute_resolution": null,
      "order_sub_status": "returning",
      "payment_method": "credit_card",
      "return_request": {
        "id": 4,
        "return_type": "defect",
        "expected_resolution": "refund_with_return",
        "reason": "側邊有一道裂痕",
        "seller_response_deadline_at": "2026-08-27 23:59:59",
        "ship_deadline_at": "2026-08-31 23:59:59",
        "inspection_deadline_at": null,
        "return_carrier": null,
        "return_tracking_number": null,
        "inspection_result": null,
        "inspection_note": null,
        "attachments": [
          { "id": 7, "uploaded_by": "buyer", "url": "http://localhost/storage/returns/7.jpg" }
        ]
      },
      "pending_actions": [
        "等待買家寄回商品（2026-08-31 23:59:59 前）；逾期未寄回將自動結案，撥款凍結同時解除。"
      ],
      "fee_refund_detail": {
        "payment_fee": { "original_amount": 7, "refund_behavior": "non_refundable", "refundable_amount": 0 },
        "platform_fee": { "original_amount": 3, "refund_behavior": "proportional", "refundable_amount": 3 },
        "union_share": { "original_amount": 1, "refund_behavior": "proportional", "refundable_amount": 1 }
      }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `items[].returned_quantity` / `refund_amount` | **退回件數與退款金額並列**（§11.1）：僅退款不退貨的品項件數為 `0`、金額為正。只看一個數字就答不出「東西該不該寄回來」。**不是 `quantity` / `subtotal`。** |
| `fee_refund_detail` | 各項費用的退還明細，**每一項是物件不是數字**：`original_amount`（當初收了多少）、`refund_behavior`、`refundable_amount`（這次實退多少）。**這就是撥款扣抵的依據**（§S-08）。 |
| `dispute_deadline_at` | 客服的裁定期限（`dispute_resolution_days`，**工作天**）；未進爭議時為 `null`。 |
| `return_request.id` | **回應退貨與驗收都要用它**：`POST seller/return-requests/{returnRequest}/decision` 與 `.../inspection` 吃的是這個整數 id，不是 `refund_no`。 |
| `return_request.attachments` | 買家依 §B-03 上傳的佐證照片，以及你自己的驗收照片。**同意與否的依據就在這裡** —— 不必先點開明細。 |
| `return_request.seller_response_deadline_at` | **你**的回應期限（工作天）。逾期視為同意。 |
| `pending_actions` | 可直接顯示的句子，說明現在輪到誰、什麼時候之前。詳見下方。 |
| `order_sub_status` | 對應的子訂單狀態。 |
| `payment_method` | 原付款管道；退款由平台財務執行，你這裡是唯讀資訊。 |
| `return_request` | 免退貨的退款單（買家取消、賣家取消、逾期未出貨、配送異常、超商未取）為 `null` —— 那條路徑本來就沒有申請單。 |

退款回應另含 `returned_shipment`（物件或 `null`），六個欄位及意義見
[物流退回實體收件／驗收](../buyer/05-returns-and-refunds.md#物流退回的實體收件驗收1441)。
它與 `return_request` 分開：後者是買家申請退貨，前者是物流退回商品的實際收件紀錄。
財務已退款仍可有待驗收提醒，不能僅依退款狀態隱藏賣家的驗收工作。

### `pending_actions`：同一個狀態，你和買家看到的不是同一句

`pending` 時買家看到「賣家回應中」，你看到「請回應這筆退貨申請（X 前）；
逾期未回應視為同意」。同一份文案給兩個人看，總有一個人會照著做錯的事。

三個期限各自綁著一個逾期預設結果，而句子裡會把期限一起說出來：

| 狀態 | 你的待辦 | 逾期會怎樣 |
| --- | --- | --- |
| `pending` | 請回應這筆退貨申請 | 視為同意（`seller_return_response_days`，工作天） |
| `approved` | 等待買家寄回 | 自動結案不予退款，**你的撥款凍結解除** |
| `return_received` | 請完成驗收 | 視為通過（`seller_inspection_days`，工作天） |
| `processing` | 等待平台財務執行退款 | — |
| `processed` / `closed` | 撥款凍結已解除 | — |

### 退款事由與你的同意權

| `reason_type` | 路徑 | 你可否拒絕 |
| --- | --- | --- |
| `return_withdrawal`（七日解除權） | 需退貨 | ❌ |
| `return_defect`（瑕疵／錯發／短少） | 需退貨 | ✅ |
| `buyer_cancel`（買家取消未出貨訂單） | 免退貨 | ❌ |
| `seller_cancel`（賣家主動取消） | 免退貨 | ❌ |
| `ship_deadline_expired`（逾期未出貨自動取消） | 免退貨 | ❌ |
| `delivery_exception_returned`（配送異常退回） | 免退貨 | ❌ |
| `pickup_not_collected`（超商逾期未取貨退回） | 免退貨 | ❌ |

**只有一種你拒絕得了**。七日解除權是法定權利，不是協商項目。

---

## 2. 回應退貨申請

依鎖後最新退款狀態判斷，重送已回應的申請回 422，不改寄回期限。
申請異動、退款轉態與通知同交易，通知失敗不保留前半段修改。

```
POST seller/return-requests/{returnRequest}/decision
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `approve` | boolean | ✅ | `true` 同意、`false` 拒絕 |
| `reason` | string\|null | **拒絕時必填** | 最長 500 |

回應 200，`data` 為**退貨申請**（不是退款單），`message`：`已回應退貨申請。`

### 逾時視為同意（§14.4）

`seller_return_response_days`（**工作天**，預設 3）內沒有回應，系統自動視為同意。
申請上的 `seller_response_deadline_at` 就是那個時點 —— 前端請顯示倒數，
不要讓賣家以為「不理它就沒事」。

### 拒絕會直接進爭議

拒絕不是結案，是**升級**：退款單轉入 `disputed` 由平台客服裁定。
因此 `reason` 必填且會被客服讀到 —— 它是你這一方唯一的陳述。

### 退貨申請的形狀

```json
{
  "id": 1,
  "order_sub_no": "M2026082400008587-01",
  "return_type": "defect",
  "reason": "…",
  "refund_no": "R2026082400005194",
  "status": "approved",
  "seller_response_deadline_at": "2026-08-27 23:59:59",
  "ship_deadline_at": "2026-08-31 23:59:59",
  "inspection_deadline_at": null,
  "return_carrier": null,
  "return_tracking_number": null,
  "inspection_result": null,
  "expected_resolution": "refund",
  "items": [
    { "order_item_id": 8, "product_sku_id": 1, "quantity": 1, "unit_price": 160, "subtotal": 160 }
  ],
  "return_shipping_notice": "退貨運費由賣家負擔，不自退款金額中扣除 (spec §14.5.1)。"
}
```

**三個期限，三種逾期預設結果**（§14.4）：

| 欄位 | 誰的期限 | 逾期 |
| --- | --- | --- |
| `seller_response_deadline_at` | 你回應 | 視為同意 |
| `ship_deadline_at` | 買家寄回並登錄單號 | 自動結案，不予退款 |
| `inspection_deadline_at` | 你驗收 | 視為驗收通過 |

> `expected_resolution` 為 `refund_only` 的品項**不需寄回**，因此那張申請沒有寄回期限，
> 也不會走驗收。

---

## 3. 退貨驗收

先鎖主訂單、子訂單、退款與申請，再異動庫存；已進爭議或已驗收的舊請求回 422。
驗收結果、庫存、退款狀態及通知一起提交或回滾，重送不改原結果。

```
POST seller/return-requests/{returnRequest}/inspection
```

**本端點接受 `multipart/form-data`**（要附驗收照片時）：

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `passed` | boolean | ✅ | 驗收是否通過 |
| `note` | string\|null | **不通過時必填** | 最長 500 |
| `photos[]` | file[] | — | `jpg` / `jpeg` / `png`，單檔最大 `return_photo_max_kb`，最多 `return_photo_limit` 張 |

回應 200，`data` 為退貨申請，`message`：`已完成驗收。`

### 驗收照片是選填，但驗收不通過時值得拍

通過時沒有東西要證明。**不通過而買家有異議時，那是唯一能還原「你收到的是什麼
狀態」的東西**（§S-07）—— 客服裁定爭議時看的就是它，與買家寄出前拍的那組並列。

照片存入與買家共用的同一份清單，以 `uploaded_by` 區分（`buyer` / `seller`），
出現在 `data.attachments`。

### 驗收結果決定貨去哪裡

| `passed` | 庫存異動 | 後果 |
| --- | --- | --- |
| `true` | `return_to_sellable`（待檢 −） | 商品回到可售 |
| `false` | `return_to_defective`（待檢 −、瑕疵 +） | 進瑕疵區，**必須另外處置**（[04 §6](04-inventory.md#6-瑕疵品處置)） |

**驗收不通過不會阻止退款** —— 那是兩件事。商品狀況的爭議走客服裁定，
而買家的退款權利不因你判定瑕疵而消滅。

逾 `seller_inspection_days`（**工作天**，預設 3）未驗收視為通過。

---

## 4. 對驗收結果提出爭議

爭議以鎖後最新退款狀態判斷，已結案不可由舊請求重開；重送已成立爭議不覆寫原因、不延長客服期限。

```
POST seller/refunds/{refundNo}/dispute
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

回應 200，`data` 為退款單（`status` 為 `disputed`），
`message`：`已提出爭議，將由平台客服裁定。`

### 你什麼時候用得到它

**驗收不通過但退款金額一分不減**（§14.4.1）：商品在運送途中的毀損風險由你與
物流業者承擔，不轉嫁買家。上一條的驗收端點因此**沒有任何金額欄位**——
你不接受那個結果時，唯一的救濟就是這裡。

### 與買家共用同一條路

買賣雙方都提得出爭議，而**裁定期限自第一次升級起算**：雙方先後各提一次，
不會把客服的裁定期限延長一倍。

裁定由**客服**（`refund.dispute`），執行退款由**財務**（`refund.process`）——
兩個端點、兩種身分（§15.3）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 別家店面的退款單 |
| 422 | 已結案（`processed` / `closed`）—— 更正一律走調整單（§14.6） |

---

## 5. 確認收到退回商品（配送異常／超商未取）

```
POST seller/orders/{orderNo}/confirm-return-received
```

無請求 body。**回應 201**，`data` 為**退款單**，
`message`：`已確認收到退回商品，商品計入待驗收。`

這一條路與第 2、3 條**不是同一件事**：

| | 買家申請退貨 | 配送異常／超商未取 |
| --- | --- | --- |
| 起點 | 買家提出申請 | 物流把貨退回來 |
| 有沒有申請單 | 有（`return_requests`） | **沒有** |
| 你的同意權 | 瑕疵類可拒絕 | 無 —— 貨已經回來了 |
| 端點 | `return-requests/{id}/...` | `orders/{orderNo}/...` |

子訂單狀態為 `delivery_exception` 或 `pickup_returned` 時走這一條。

收件依主訂單 → 子訂單 → 退款鎖後重查，金額由最新成交快照計算，退款品項、入待檢與通知同交易。
仍待驗收時重播回原退款；即使財務已退款或正在爭議，也不會重新推回 `processing`。
確認收件後的重新配送回呼會留紀錄但不套用，不能把已入庫商品改成運送中。

---

## 6. 退回商品驗收（配送異常／超商未取）

收件時建立獨立實體紀錄，`inspection_deadline_at` 依下單快照的 `seller_inspection_days`
工作天及行事曆、從原收件時間起算。`refunds:auto-inspect` 亦涵蓋此路徑：逾期未驗收
視為通過，`returned_shipment.auto_resolved = true`；即使財務已退款或正在爭議仍須處理實體待檢庫存。
缺行事曆時收件、入待檢與退款照常成立並告警，期限暫為 `null`；排程暫停自動判定，
補齊後從原收件時間補算，不延後重啟。人工驗收與自動驗收只有第一次成立，結果不可覆寫。

```
POST seller/orders/{orderNo}/return-inspection
```

欄位同第 3 條（`passed`、`note`）。回應 200，`data` 為退款單，
`message`：`已完成驗收，訂單已取消。`

驗收與庫存分流同交易，鎖後讀取最新退款品項；收件前建立的舊快照不能漏驗品項。
驗收後重播（包含相反結果）及再次收件皆回 422，訊息為
`子訂單目前為 cancelled，沒有退回的商品待確認 (spec §14.3)。`，不改寫取消時間、庫存或退款。
實體驗收不修改財務狀態；退款已完成仍可驗收，不得因瑕疵而扣減退款。

> **超商未取不是買家的違規記錄，也不是你的。** 未取貨紀錄記在買家身上並有自己的
> 累計上限（§14.7.3），而「買家未取貨不是賣家的違規」（§14.7.2）——
> 你的違規點數不會因此增加。

---

## 7. 賣家提出取消申請

```
POST seller/orders/{orderNo}/cancel-requests
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 取消原因 |

**回應 201**，`data` 為取消申請，`message`：`取消申請已送出，待買家回應。`

### 這件事有代價

賣家主動取消會計入**取消率**，超標會被記 2 點違規（`cancellation_rate`，§P-07.1）。
單次取消不記點，但比率是滾動計算的 —— 缺貨就取消不是免費的。

正確的替代方案通常是聯絡買家換規格，或先出部分商品。

---

## 8. 回應買家的取消申請

```
POST seller/cancel-requests/{cancelRequest}/response
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `approve` | boolean | ✅ | 同意或拒絕 |
| `reason` | string\|null | 拒絕時必填 | —— |

回應 200，`data` 為取消申請，`message`：`已回應取消申請。`

### 取消申請的形狀

```json
{
  "id": 1,
  "order_sub_no": "M2026082400002213-01",
  "requested_by": "buyer",
  "responded_by": "seller",
  "reason": "…",
  "status": "approved",
  "response_deadline_at": "2026-08-26 23:59:59",
  "responded_at": "2026-08-24 05:01:14",
  "response_reason": null,
  "auto_resolved": false
}
```

`auto_resolved` 區分「你同意了」與「你沒回應所以視為同意」——
兩者在爭議時是不同的事實（§14.3）。逾期時限是
`buyer_cancel_response_days`（**工作天**，預設 2）。

> **已出貨的訂單取消不了。** 買家要退請走退貨流程；
> 那時退貨運費由你負擔（§14.5.1）。
