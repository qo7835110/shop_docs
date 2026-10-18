# 05 退貨、取消與退款

對應 spec B-03、B-06、§14.3、§14.4、§14.5、§14.5.1。

## 退貨運費由賣家負擔

**不自退款金額中扣除**（§14.5.1）。每一個買家視角的退款回應都會帶一句
`return_shipping_notice`，逐字說明這件事 —— 那是規則，不是可協商的項目。

賣家在對話中要求你分擔退貨運費是**違規**（3 點），可以檢舉。

## 退款單的兩條路徑（§14.4）

```
需退貨： pending → approved → return_received → inspected → processing → processed
免退貨： pending → processing → processed
```

- 終態只有 `processed`（已退款）與 `closed`（結案不退）。
- `disputed` 是**退款單的子狀態**，不是訂單狀態。
- **賣家拒絕退貨申請會自動轉入爭議**，由平台客服裁定 —— 你不需要再按一次
  「提出爭議」（§12.3-81）。第 7 條是給**其他**情形用的。

---

## 1. 申請退貨退款

```
POST buyer/sub-orders/{orderNo}/return-requests
```

`{orderNo}` 是**子訂單編號**。**本端點是 `multipart/form-data`**，不是 JSON ——
佐證照片是檔案，而它得與其餘欄位在同一次送出（理由見下）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `return_type` | string | ✅ | `withdrawal`（七日解除權）\| `defect`（瑕疵／錯發／短少） |
| `expected_resolution` | string | ✅ | `refund_with_return`（退貨退款）\| `refund_only`（僅退款不退貨） |
| `reason` | string | ✅ | 最長 500 |
| `items` | array | ✅ | 至少一項 —— **可以只退部分品項** |
| `items[].order_item_id` | integer | ✅ | —— |
| `items[].quantity` | integer | ✅ | ≥ 1 |
| `photos[]` | file[] | **視 `return_type` 而定** | `jpg` / `jpeg` / `png`，單檔最大 `return_photo_max_kb`（預設 5 MB），最多 `return_photo_limit` 張（預設 5） |

**回應 201**，`data` 為退貨申請，`message`：`退貨申請已送出。`

### 佐證照片：`defect` 必填，`withdrawal` 選填

| `return_type` | 照片 | 理由 |
| --- | --- | --- |
| `defect` | ✅ **至少一張** | 照片就是「瑕疵確實存在」的證據本身。少了它，賣家的驗收與客服的爭議裁定都只能靠雙方各說各話。 |
| `withdrawal` | — 選填 | 消保法第 19 條明訂行使解除權「**無須說明理由**及負擔任何費用或對價」。要求照片等於在法定權利前面加一道法所不許的障礙，而你要拍的東西根本不存在（商品沒有問題）。 |

缺照片的 `defect` 申請回 **422**，且**不會留下半張退貨單** ——
否則 §14.4.2 的冪等鍵會讓補了照片的重送變成「已存在」，於是你再也送不出去。

> 這也是為什麼照片不做成「先建單、再上傳」的兩步：那樣一來「至少一張」
> 就無從在建立時檢查。

### 兩種事由，賣家的權利不同

| `return_type` | 賣家可否拒絕 | 說明 |
| --- | --- | --- |
| `withdrawal` | ❌ **不可** | 七日解除權是法定權利 |
| `defect` | ✅ 可以 | 拒絕後自動進入爭議，由平台客服裁定 |

### 照片在回應裡怎麼呈現

`data.attachments` 是**買賣雙方共用的一份清單**，以 `uploaded_by` 區分：

```json
"attachments": [
  { "id": 1, "uploaded_by": "buyer",  "url": "http://…/storage/return-requests/5/aaa.jpg" },
  { "id": 2, "uploaded_by": "seller", "url": "http://…/storage/return-requests/5/bbb.jpg" }
]
```

`buyer` 是你寄出前拍的，`seller` 是賣家驗收時拍的 —— 同一件事的兩個時點。
驗收不通過而你有異議時，那兩組照片就是客服裁定的依據（§S-07）。

### 退貨申請的形狀

```json
{
  "id": 5,
  "order_sub_no": "M2026082400019761-01",
  "return_type": "defect",
  "reason": "包裝破損",
  "refund_no": "R2026082400005194",
  "status": "approved",
  "seller_response_deadline_at": "2026-08-27 23:59:59",
  "ship_deadline_at": "2026-08-31 23:59:59",
  "inspection_deadline_at": null,
  "return_carrier": null,
  "return_tracking_number": null,
  "inspection_result": null,
  "expected_resolution": "refund_with_return",
  "items": [
    { "order_item_id": 8, "product_sku_id": 1, "quantity": 1, "unit_price": 160, "subtotal": 160 }
  ],
  "return_shipping_notice": "退貨運費由賣家負擔，不自退款金額中扣除 (spec §14.5.1)。"
}
```

**三個期限，三種逾期預設結果**：

| 欄位 | 誰的期限 | 逾期會怎樣 |
| --- | --- | --- |
| `seller_response_deadline_at` | 賣家回應 | **視為同意** |
| `ship_deadline_at` | **你**寄回並登錄單號 | **自動結案，不予退款** |
| `inspection_deadline_at` | 賣家驗收 | 視為驗收通過 |

第二個是唯一對你不利的 —— 逾期就沒了。前端請對它顯示倒數。

> `expected_resolution` 選 `refund_only` 的品項**不需寄回**，
> 因此那張申請不會有寄回期限，也不會走驗收。

### 失敗

建案前會鎖定主訂單及子訂單，重查最新狀態、退貨期限、既有申請、累計額度與自動結案次數。
同時送出的申請只允許一筆成立；另一筆回 422，請讀回既有退款，不要當成新的申請再送。
金額依資料庫的成交快照與剩餘件數重算；分次退貨最後一筆才退原運費（§B-03／§14.5.2）。
本防護涵蓋領域建案；附件上傳中斷的恢復流程仍待補，不能將 422 當成附件已上傳的證據。

| 狀態 | 情況 |
| --- | --- |
| 403 | 不是你的訂單 |
| 422 | **退貨期限已過**（`is_return_window_open` 為 `false`） |
| 422 | 子訂單狀態不允許（例如還沒送達） |
| 422 | 該品項已有進行中的退貨申請 |

---

## 2. 登錄寄回物流單號

```
POST buyer/return-requests/{returnRequest}/shipment
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `carrier` | string | ✅ | 最長 40 |
| `tracking_number` | string | ✅ | 最長 64 |

回應 200，`data` 為退貨申請，`message`：`已登錄寄回物流單號。`

**這一步有期限，而且逾期不予退款。** `ship_deadline_at` 是
`buyer_return_ship_days`（**日曆天**，預設 7）自賣家同意起算。

只有 `approved` 的申請登錄得了；`refund_only` 的申請沒有這一步。
狀態在鎖定後重查；已寄回、已結案或爭議中的舊請求回 422，不覆寫原單號，也不留下新的驗收期限。
自動結案同樣重查最新狀態、寄回期限與單號；寄回或爭議先成立時不會被舊候選結案。

---

## 3. 申請取消訂單

```
POST buyer/sub-orders/{orderNo}/cancel-requests
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

**回應 201**，`data` 為取消申請，`message`：`取消申請已送出，待賣家回應。`

### 只有未出貨的訂單取消得了

已出貨請走退貨（第 1 條）。

申請與回應都在主訂單鎖後重查。已有決定的申請重播回 422，不會把同意改成拒絕或反向翻案；
同意、庫存釋放、退款建單與通知同交易提交，不會只留下「已同意」而沒有取消訂單。
賣家在回應期限內完成出貨時，買家的取消申請視為拒絕，系統記錄理由並通知，後續改走退貨。
逾期排程重查最新狀態與期限，已回應、已出貨或期限延長時略過（§13.3／§14.3）。

賣家在 `buyer_cancel_response_days`（**工作天**，預設 2）內未回應，
**視為同意**。取消申請上的 `response_deadline_at` 就是那個時點。

```json
{
  "id": 1,
  "order_sub_no": "M2026082400002213-01",
  "requested_by": "buyer",
  "responded_by": null,
  "reason": "下錯規格",
  "status": "pending",
  "response_deadline_at": "2026-08-26 23:59:59",
  "responded_at": null,
  "response_reason": null,
  "auto_resolved": false
}
```

`auto_resolved` 區分「賣家同意了」與「賣家沒回應所以視為同意」——
兩者在爭議時是不同的事實（§14.3）。

---

## 4. 回應賣家的取消申請

```
POST buyer/cancel-requests/{cancelRequest}/response
```

賣家也可以主動提出取消（缺貨等），這時輪到你回應。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `approve` | boolean | ✅ | 同意或拒絕 |
| `reason` | string\|null | **拒絕時必填** | 最長 500 |

回應 200，`message`：`已回應取消申請。`

> 賣家主動取消會計入**他的**取消率並可能被記違規（§P-07.1）——
> 你拒絕之後他仍然有出貨義務，逾期就是他的違規。

---

## 5. 退款單列表

```
GET buyer/refunds
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending`／`approved`／`rejected`／`return_received`／`inspected`／`disputed`／`processing`／`processed`／`closed` |
| `shop_id` | integer | 限定店面 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

```json
{
  "refund_no": "R2026082400005194",
  "order_sub_no": "M2026082400019761-01",
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
    "請寄回商品並登錄物流單號（2026-08-31 23:59:59 前）；逾期將自動結案，不予退款。",
    "退貨運費由賣家負擔，不自退款金額中扣除。"
  ],
  "return_shipping_notice": "退貨運費由賣家負擔，不自退款金額中扣除 (spec §14.5.1)。"
}
```

| 欄位 | 說明 |
| --- | --- |
| `goods_amount` / `shipping_amount` / `total_amount` | **三個金額分開**：只給總額就答不出「這筆有沒有退運費」。部分退貨與超商未取的答案都是「沒有」（§14.5、§14.7.2）。 |
| `items[].returned_quantity` / `refund_amount` | 退回件數與退款金額並列。`refund_only` 的品項件數為 `0`、金額為正。 |
| `dispute_deadline_at` | 進入爭議後，客服的裁定期限。 |
| `order_sub_status` | 這張退款單對應的**子訂單狀態**。「我的訂單」與「我的退貨」是同一張單的兩個畫面，不要各講各的。 |
| `payment_method` | 原付款管道。退款一律退回原管道 —— 只有管道別，沒有卡號或帳號（§2.3）。 |
| `return_request` | 關聯的退貨申請；**無需寄回的退款單為 `null`**（買家取消、賣家取消、逾期未出貨、配送異常、超商未取）。 |
| `return_request.ship_deadline_at` | **你**的寄回期限（日曆天）。逾期自動結案、不予退款 —— 唯一需要遵守它的人是你。 |
| `return_request.inspection_result` / `inspection_note` | 賣家的驗收結果與理由。看不到它，就不會知道自己有理由提爭議。 |
| `return_request.attachments` | 你上傳的佐證照片與賣家的驗收照片，同一份清單，以 `uploaded_by` 區分。 |
| `pending_actions` | **可直接顯示的句子**，說明現在輪到誰、要做什麼、什麼時候之前。詳見下方。 |

### 物流退回的實體收件／驗收（§14.4.1）

退款回應另含 `returned_shipment`：配送異常／超商未取商品已退回賣家時為物件，
其他情形（含一般退貨申請、未出貨取消）為 `null`。買家、賣家、平台讀取同一份紀錄。

| 回應欄位 | 意義 |
| --- | --- |
| `received_at` | 實際收件時間，不隨重播改寫 |
| `inspection_deadline_at` | 按下單快照的驗收工作天及行事曆，從收件時間計算；缺行事曆時為 `null` |
| `inspected_at` | 驗收完成時間；待檢時為 `null` |
| `inspection_result` | `passed`／`failed`；待檢時為 `null` |
| `inspection_note` | 驗收說明，可為 `null`；不通過須有原因 |
| `auto_resolved` | 是否逾期由系統視為通過 |

財務 `processing`／`processed` 不代表商品已驗收。尚待檢時 `pending_actions` 另提示
等待賣家驗收及期限；驗收不影響退款。缺行事曆會提示平台補齊，暫停自動判定。

### `pending_actions`：後端算好的下一步

退款單有九個狀態（§14.4），其中只有 `approved` 一個需要**你**行動。
那個判斷由後端做一次，不由每個畫面各推導一次 —— 推錯的那一次，
代價是錯過寄回期限，退款單被自動結案。

期限也一起寫在句子裡：「請寄回商品」與「請於 8/31 前寄回，逾期不予退款」
是兩句不同的話，只有後者讓人知道現在該不該急。

**同一個狀態，三方看到的不是同一句。** `pending` 時你看到「賣家回應中」，
賣家看到「請回應」。要分支的人用 `status`，這裡回的是給人看的字。

> **買家視角沒有 `fee_refund_detail`** —— 那是賣家被收了多少平台費、
> 撥款要怎麼扣抵，與你無關（§2.3）。

---

## 6. 退款單詳情

```
GET buyer/refunds/{refundNo}
```

`{refundNo}` 是**退款單號字串**。`data` 形狀與列表單筆相同。

---

## 7. 提出爭議

```
POST buyer/refunds/{refundNo}/dispute
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

回應 200，`data` 為退款單（`status` 為 `disputed`），
`message`：`已提出爭議，將由平台客服裁定。`

### 什麼時候需要按這個

**賣家拒絕退貨申請時不需要** —— 那會自動升級成爭議。這一支是給其他情形用的，
例如驗收結果你不同意、或退款金額有疑義。

### 到什麼時候之前可以按

**到財務把錢退出去之前**，也就是退款單還沒進 `processed` 或 `closed` 之前
（§14.4）。驗收之後退款單會直接進 `processing`（驗收不改變退款金額，
沒有理由停下來等人），而那個狀態**仍然可以提出爭議** ——
否則「不同意驗收結果」就只剩理論上的權利。

提出後退款會**暫停執行**，等客服裁定。`processed` 之後不得再提，
更正一律走調整單（§14.6）。
同時送出爭議與退款時以鎖後最新狀態判斷：已結案不得重開；已在爭議中的重送不改原因或延長裁定期限。

### 進入爭議之後

- 平台客服會**加入你與賣家的對話**（§X-03.5），加入與退出皆留痕；
  你會在對話的 `cs_present` 看到他在不在（[08](08-messages.md#1-對話列表)）。
- 客服裁定「該退」只是把退款單推到 `processing`，**實際出款由平台財務執行**
  （§15.3）——「裁定」與「執行」是兩個身分。
- 裁定「不該退」則退款單 `closed`；子訂單依申請前狀態與原自動完成期限回復為已送達或已完成，
  不重新起算期限（§14.4.0）。

裁定期限是 `dispute_resolution_days`（**工作天**，預設 5）。
**逾期不會自動裁定** —— 由平台主管指派處理。
