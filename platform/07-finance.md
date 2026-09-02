# 07 金流作業（退款、對帳、撥款、調整、拒付）

對應 spec §14.4、§14.6、§14.8、§14.9、P-02、P-08、§15.3。

## 先讀這三句

1. **平台不保管金流。** 付款由第三方金流商處理，平台記錄結果並據以推導退款、費用拆分與撥款。
   畫面文案不要寫成「平台代收代付」。
2. **裁定與執行是兩個身分。** 客服裁定「該不該退」（`refund.dispute`），
   財務執行「退出去」（`refund.process`）。撥款同理。這是 §15.3 的硬規則，**不可由設定放寬**。
3. **任何金額更正一律走調整單**，不改原始付款、退款或撥款紀錄。

---

# 退款

## 退款狀態機（§14.4）

兩條路徑共用同一種退款單：

```
需退貨： pending → approved → return_received → inspected → processing → processed
免退貨： pending → processing → processed
```

- `disputed` 是**退款單的子狀態**，不是子訂單狀態。
- 終態只有 `processed` 與 `closed`。
- `rejected` 表示賣家拒絕退貨申請。**駁回即自動轉入 `disputed`**，買家**不需要**
  再按「提出爭議」（§14.4、§12.3-81）。因此 `rejected` 是**過渡狀態，不是終態** ——
  停在那裡會讓賣家的款項被永久凍結（撥款條件 4 要求無進行中的退款）。
  回應同時帶 `dispute_deadline_at`：客服的裁定期限，`dispute_resolution_days`
  （**工作天**）自升級當下起算。

## 1. 退款單列表

```
GET platform/refunds
```

需要 `order.view`（三個平台角色都有）。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending`／`approved`／`rejected`／`return_received`／`inspected`／`disputed`／`processing`／`processed`／`closed` |
| `shop_id` | integer | 限定店面。走**子訂單**的店面欄位 —— 退款單掛在子訂單上，不直接掛店面 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得退款單列表。",
  "data": [
    {
      "refund_no": "RF20260818000031",
      "order_sub_no": "SO20260810000127",
      "reason_type": "return_defect",
      "reason_label": "瑕疵／錯發／短少",
      "requires_return": true,
      "return_type": "defect",
      "status": "disputed",
      "items": [
        {
          "order_item_id": 8801,
          "product_sku_id": 9001,
          "product_name": "手工皮革刀具組",
          "spec_name": "中型",
          "unit_price": 1280,
          "returned_quantity": 1,
          "refund_amount": 1280
        }
      ],
      "goods_amount": 1280,
      "shipping_amount": 60,
      "total_amount": 1340,
      "requested_at": "2026-08-15 09:12:00",
      "approved_at": null,
      "return_received_at": null,
      "inspected_at": null,
      "dispute_deadline_at": "2026-08-22 09:12:00",
      "processed_at": null,
      "closed_at": null,
      "reject_reason": "外觀無異常",
      "closed_reason": null,
      "dispute_reason": "收到時包裝已破損，有照片為證",
      "dispute_resolution": null,
      "fee_refund_detail": {
        "payment_fee": { "original_amount": 40, "refund_behavior": "non_refundable", "refundable_amount": 0 },
        "platform_fee": { "original_amount": 38, "refund_behavior": "proportional", "refundable_amount": 38 },
        "union_share": { "original_amount": 25, "refund_behavior": "proportional", "refundable_amount": 25 }
      },
      "reference_type": "return_request",
      "reference_id": "5521",
      "processed_by_admin_id": null,
      "provider_refund_id": null,
      "order_sub_status": "returning",
      "payment_method": "credit_card",
      "return_request": {
        "id": 5521,
        "return_type": "defect",
        "expected_resolution": "refund_with_return",
        "reason": "收到時包裝已破損",
        "seller_response_deadline_at": "2026-08-17 23:59:59",
        "ship_deadline_at": "2026-08-20 23:59:59",
        "inspection_deadline_at": "2026-08-21 23:59:59",
        "return_carrier": "黑貓宅急便",
        "return_tracking_number": "RTN-0001",
        "inspection_result": "failed",
        "inspection_note": "外觀有使用痕跡",
        "attachments": [
          { "id": 7, "uploaded_by": "buyer", "url": "http://localhost/storage/returns/7.jpg" },
          { "id": 9, "uploaded_by": "seller", "url": "http://localhost/storage/returns/9.jpg" }
        ]
      },
      "pending_actions": [
        "待客服裁定（2026-08-22 09:12:00 前）；逾期不自動裁定，由平台主管指派處理。"
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 12, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `reason_type` / `reason_label` | 退款事由代碼與可直接顯示的中文標籤。事由決定走哪條路徑、賣家可否拒絕、是否退還出貨運費。 |
| `requires_return` | 是否需要買家寄回商品。`false` 就是免退貨路徑（未出貨取消、超商未取回等）。 |
| `items[].returned_quantity` / `items[].refund_amount` | **退回件數與退款金額並列**（§11.1）：僅退款不退貨的品項退回件數為 `0`、金額為正。只看其中一個數字就答不出「商品到底該不該寄回去」。**不是 `quantity` / `subtotal`。** |
| `dispute_deadline_at` | 客服的裁定期限，`dispute_resolution_days`（**工作天**）自升級當下起算；未進爭議時為 `null`。 |
| `fee_refund_detail` | 各項費用的退還明細。**每一項是物件，不是數字**：`original_amount`（當初收了多少）、`refund_behavior`（`proportional` 按比例退／`non_refundable` 不退）、`refundable_amount`（這次實退多少）。三個數字都要，只給一個就答不出「金流費為什麼沒退」。鍵為 `payment_fee`、`platform_fee`、`union_share`，**費率規則未設定的項目不會出現**。 |
| `processed_by_admin_id` / `provider_refund_id` | 執行退款的財務人員與金流商的退款交易編號。 |
| `payment_method` | 原付款管道。退款一律退回原管道，因此**執行退款前**就需要它 —— `provider_refund_id` 是退款**之後**才有值的那一個。只有管道別，沒有卡號或帳號（§2.3）。 |
| `order_sub_status` | 對應的子訂單狀態。 |
| `return_request` | 關聯的退貨申請；免退貨路徑為 `null`。**裁定爭議要用的三個說法就在這裡**：買家的 `dispute_reason`（在退款單上）、賣家的 `inspection_note`、雙方的 `attachments`。少了它們，客服只能改用資料庫或另外開票去問，而 `dispute_resolution_days` 就燒在那上面。 |
| `pending_actions` | 平台的待辦，**分屬兩種身分**（§15.3）：`disputed` → 待客服裁定（`refund.dispute`）；`processing` → 待財務執行退款（`refund.process`）。其餘狀態為空陣列 —— 平台沒事要做。 |

> **爭議逾期沒有自動結果。** 其餘每一個期限逾期都有一個預設結果（視為同意、
> 視為通過、自動結案），只有這一個沒有：由平台主管指派人力處理（§14.4）。
> 理由是這裡沒有「沒有行動的那一方」——買賣雙方都已經行動了，沒行動的是平台自己，
> 而平台不該替自己的怠惰設一個自動有利的結果。

### 退款事由

| `reason_type` | 路徑 | 賣家可否拒絕 | 說明 |
| --- | --- | --- | --- |
| `return_withdrawal` | 需退貨 | ❌ | 七日解除權 |
| `return_defect` | 需退貨 | ✅ | 瑕疵／錯發／短少 |
| `buyer_cancel` | 免退貨 | ❌ | 買家取消未出貨訂單 |
| `seller_cancel` | 免退貨 | ❌ | 賣家主動取消 |
| `ship_deadline_expired` | 免退貨 | ❌ | 逾期未出貨 |
| `delivery_exception_returned` | 免退貨 | ❌ | 配送異常退回 |
| `pickup_not_collected` | 免退貨 | ❌ | 超商逾期未取 |

> **退貨運費由賣家負擔，不從退款金額中扣除**（§14.5.1）。前端不要提供「扣除運費」的欄位。

---

## 2. 爭議裁定（客服）

```
POST platform/refunds/{refundNo}/dispute-decision
```

需要 **`refund.dispute`**。`{refundNo}` 是退款單號字串（例如 `RF20260818000031`），不是數字 ID。

**裁定不等於執行退款。** 裁定為「該退」只是把退款單推到 `processing`，
錢還沒退出去；實際出款是第 3 條，由財務執行。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `should_refund` | boolean | ✅ | `true` 該退、`false` 不該退 |
| `resolution` | string | ✅ | 最長 500，裁定說明 |

```json
{ "should_refund": true, "resolution": "照片可佐證運送破損，判定應予退款" }
```

### 回應 200

`data` 為退款單（形狀同列表單筆），`message`：`已完成爭議裁定。`

| `should_refund` | 結果 |
| --- | --- |
| `true` | 退款單 → `processing`，等待財務執行 |
| `false` | 退款單 → `closed`，**子訂單轉為「已完成」**（不回到「已送達」，不重啟鑑賞期） |

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 沒有 `refund.dispute` | `您沒有裁定退款爭議的授權 (spec §15.3)。` |
| 404 | 退款單號不存在 | `找不到指定的資料。` |
| 422 | 退款單不在 `disputed` | `這張退款單不在爭議處理中 (spec §14.4)。` |

---

## 3. 執行退款（財務）

```
POST platform/refunds/{refundNo}/process
```

需要 **`refund.process`** —— **僅財務**，客服不可，且此限制不可由設定放寬。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `provider_refund_id` | string\|null | — | 最長 64，金流商的退款交易編號 |

### 回應 200

`data` 為退款單（`status` 為 `processed`），`message`：`退款已執行。`

> **冪等**：重送同一筆已 `processed` 的退款會直接回既有結果，不會退第二次。

副作用（前端可預期畫面上會跟著變）：

- 子訂單結算，撥款金額扣抵。
- 若為全額退款，該筆交易的評價會被標註「此筆交易已全額退款」。
  **評價本身保留、仍計入統計**，不會被隱藏 —— 否則賣家就有動機以退款換掉負評。

---

# 金流對帳（僅財務）

匯入與比對是**兩個權限**，因為後果不同等級：匯入只是把外部檔案放進系統，
比對則會改變付款的可出款狀態。

## 4. 結算檔列表

```
GET platform/settlement-files
```

需要 `settlement.reconcile`。**分頁**。

### 回應 200

```json
{
  "success": true,
  "message": "取得結算檔列表。",
  "data": [
    {
      "id": 12,
      "provider": "ecpay",
      "period": "2026-08-W2",
      "status": "reconciled",
      "declared_count": 431,
      "declared_amount": 1284500,
      "imported_at": "2026-08-16 08:00:00",
      "reconciled_at": "2026-08-16 08:05:12"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 12, "last_page": 1 } }
}
```

`status`：`imported`（已匯入待比對）\| `reconciled`（已比對）。

---

## 5. 匯入結算檔

```
POST platform/settlement-files
```

需要 **`settlement.import`**。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `provider` | string | ✅ | 最長 40，金流商代碼 |
| `period` | string | ✅ | 最長 20，結算期別 |
| `original_name` | string\|null | — | 最長 255，原始檔名 |
| `declared_count` | integer | — | 對方宣稱的筆數 |
| `declared_amount` | integer | — | 對方宣稱的總額（整數 TWD） |
| `lines` | array | ✅ | 至少 1 筆 |
| `lines[].provider_transaction_id` | string | ✅ | 最長 64 |
| `lines[].amount` | integer | ✅ | 整數 TWD |
| `lines[].fee` | integer | — | 手續費 |
| `lines[].settled_at` | date\|null | — | 撥付日 |

```json
{
  "provider": "ecpay",
  "period": "2026-08-W2",
  "original_name": "ecpay_20260816.csv",
  "declared_count": 2,
  "declared_amount": 3000,
  "lines": [
    { "provider_transaction_id": "TX0001", "amount": 1800, "fee": 54, "settled_at": "2026-08-15" },
    { "provider_transaction_id": "TX0002", "amount": 1200, "fee": 36, "settled_at": "2026-08-15" }
  ]
}
```

### 回應 201

```json
{
  "success": true,
  "message": "結算檔已匯入。",
  "data": {
    "id": 13,
    "provider": "ecpay",
    "period": "2026-08-W2",
    "status": "imported",
    "declared_count": 2,
    "declared_amount": 3000,
    "imported_at": "2026-08-18 08:00:00",
    "reconciled_at": null,
    "declared_totals_match": true
  },
  "errors": null,
  "meta": null
}
```

> **`declared_totals_match` 一定要顯示。** 它表示「對方宣稱的總數與逐筆加總是否一致」；
> 為 `false` 時，逐筆對帳的結論全都不可信 —— 請先向金流商確認檔案，不要急著按比對。

---

## 6. 執行比對

```
POST platform/settlement-files/{settlementFile}/reconcile
```

回應的 `summary` 除了各結果的筆數，另帶 `payouts_now_payable`：
**現在通得過出款前驗證的撥款單數**。

對帳是出款的前置 —— 出款前的五項再驗證裡有一項是「付款已完成對帳」（§12.1）。
少了這個數字，對完帳去按出款，被擋下才知道那張候選對應的付款不在剛才那個檔案裡。

用的是與出款端**同一個** `PayoutEligibility`，不是另寫一次條件：
兩份規則遲早會漂移，而漂移的症狀是「這裡說可以撥，按下去被擋」。

```
```

需要 **`settlement.reconcile`**。無請求 body。

**不會改動任何付款金額**，只寫入付款紀錄的對帳狀態與對帳時間。

### 回應 200

```json
{
  "success": true,
  "message": "對帳完成。",
  "data": {
    "file": { "…": "同結算檔列表單筆（status 已變 reconciled）" },
    "summary": {
      "matched": 428,
      "missing_in_platform": 1,
      "missing_in_provider": 2,
      "amount_mismatch": 0,
      "fee_mismatch": 0,
      "duplicated": 0
    }
  },
  "errors": null,
  "meta": null
}
```

| 比對結果 | 意義 |
| --- | --- |
| `matched` | 相符 |
| `missing_in_platform` | 金流有、平台無 |
| `missing_in_provider` | 平台有、金流無 |
| `amount_mismatch` | 金額不符 |
| `fee_mismatch` | 手續費不符 |
| `duplicated` | 重複入帳 |

`matched` 以外的每一筆都會產生一筆對帳差異，需**逐筆結案**。

---

## 7. 對帳差異列表

```
GET platform/reconciliation-discrepancies
```

需要 `settlement.reconcile`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `open`（未結案）\| `resolved`（已結案）\| `watching`（觀察中）。**可逗號分隔**，見下 |
> **`status` 接受逗號分隔的多個值**（例如 `status=open,watching`）。
> [營運總覽](02-overview.md#links--每一個數字自己的去處)的風險數字是**兩個狀態的
> 聯集**，而總覽的 `links` 帶的正是這種寫法 —— 一次只吃一個值的話，
> 那個數字會「數得出來卻列不出來」。每一段都必須是允許值，否則整個請求回 422
> 並列出允許值。


### 回應 200

```json
{
  "success": true,
  "message": "取得對帳差異列表。",
  "data": [
    {
      "id": 77,
      "type": "amount_mismatch",
      "type_label": "金額不符",
      "status": "open",
      "summary": "TX0001 平台 1800 / 金流 1790",
      "context": { "provider_transaction_id": "TX0001", "platform_amount": 1800, "provider_amount": 1790 },
      "resolution": null,
      "resolved_at": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 3, "last_page": 1 } }
}
```

---

## 8. 差異結案

```
PATCH platform/reconciliation-discrepancies/{discrepancy}
```

需要 `settlement.reconcile`。**逐筆結案，沒有整批略過的端點。**

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `open` \| `resolved` \| `watching` |
| `resolution` | string | ✅ | 最長 500 |

回應 200，`data` 為差異單筆，`message`：`差異已結案。`

> **這裡不改任何金額。** 需要更正金額請另建調整單（第 12 條）。

---

# 撥款

```
撥款金額 = 商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整
```

回應中的 `formula` 欄位就是這句話，可直接顯示在明細頁。
**退貨運費不進這條算式** —— 賣家自行與物流業者簽約，那筆錢不經過平台。

**沒有「建立撥款單」的端點**：撥款由排程產生候選，不由賣家申請、不由單筆訂單即時觸發。
少一條路，就少一個繞過候選條件的入口。

## 9. 撥款單列表

```
GET platform/payouts
```

需要 `payout.view`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending` \| `paid` \| `failed` \| `deferred` \| `aborted`。**可逗號分隔** |
> **`status` 接受逗號分隔的多個值**（例如 `status=open,watching`）。
> [營運總覽](02-overview.md#links--每一個數字自己的去處)的風險數字是**兩個狀態的
> 聯集**，而總覽的 `links` 帶的正是這種寫法 —— 一次只吃一個值的話，
> 那個數字會「數得出來卻列不出來」。每一段都必須是允許值，否則整個請求回 422
> 並列出允許值。

| `shop_id` | integer | 限定店面 |

### 回應 200

```json
{
  "success": true,
  "message": "取得撥款單列表。",
  "data": [
    {
      "payout_no": "PO20260818000004",
      "shop_id": 21,
      "status": "pending",
      "period_start_at": "2026-08-04 00:00:00",
      "period_end_at": "2026-08-10 23:59:59",
      "formula": "商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整",
      "goods_amount": 42800,
      "shipping_amount": 1200,
      "refund_amount": 1340,
      "payment_fee": 1290,
      "platform_fee": 1284,
      "union_share": 856,
      "adjustment_amount": 0,
      "net_amount": 39230,
      "retry_count": 0,
      "failed_reason": null,
      "aborted_reason": null,
      "paid_at": null,
      "sub_orders": [
        {
          "order_sub_id": 3312,
          "goods_amount": 1280,
          "shipping_amount": 60,
          "refund_amount": 0,
          "payment_fee": 40,
          "platform_fee": 38,
          "union_share": 25,
          "net_amount": 1237
        }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 18, "last_page": 1 } }
}
```

---

## 10. 執行出款（財務）

```
POST platform/payouts/{payoutNo}/execute
```

需要 **`payout.execute`** —— **僅財務**。`{payoutNo}` 是撥款單號字串。無請求 body。

**手動執行不是「跳過檢查的那條路」**：出款前一定會在同一交易內重跑**五項**驗證
（§14.9.1、§12.3-28）：

1. 子訂單 `已完成` 且距 `completed_at` 已逾 `payout_hold_days`（日曆天）。
2. **該子訂單的退貨窗口已屆滿**（送達日 ＋ `return_window_days`，日曆天）。
3. 該筆付款已完成對帳且比對相符。
4. 無進行中的退貨退款、爭議或拒付。
5. 店面銀行帳戶完整且通過驗證。

條件 1 與 2 不能合併：兩者都是「等一段日曆天」，但**起算點不同** —— 前者自
`completed_at`、後者自**送達日**，而買家可以在送達當日就點確認收貨。
建議預設值下兩者都是 7 天，只靠條件 1 會讓款項在買家仍有法定退貨權的當下撥出
（§12.3-80）。

### 回應 200 —— 三種正常結果

回應一律是 200，**用 `data.status` 判斷結果**，不要以 HTTP 狀態碼判斷：

| `status` | `message` | 意義 |
| --- | --- | --- |
| `paid` | `撥款已執行。` | 已出款，`paid_at` 有值 |
| `aborted` | `出款前再驗證未通過，已中止並記錄原因。` | 未通過再驗證，原因在 `aborted_reason` |
| `deferred` | `金額未達門檻或為負，遞延下期繼續抵扣。` | `net_amount` ≤ 0 或未達最低金額 |

> `aborted` 與 `deferred` **都是正常的營運結果，不是錯誤**。前端不要顯示成紅色錯誤訊息，
> 也不要自動重試。
>
> 負數餘額**結轉下期繼續抵扣**，既不截斷為 0，也不向賣家追收現金。

`failed` 是另一回事：出款送出後被銀行或金流端拒絕。系統會依 `payout_retry_limit`
自動重試，逾次數才轉 `failed` 等待財務手動重送。

> **冪等**：對已 `paid` 的撥款單重送會直接回既有結果，不會出第二次款。

---

# 調整單

**任何金額更正一律走這裡**，不改原始付款、退款或撥款紀錄（§14.6）。

## 11. 調整單列表

```
GET platform/adjustments
```

需要 `payout.view`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending` \| `approved` \| `settled` \| `cancelled` |
| `shop_id` | integer | 限定店面 |

> **`status` 用的是調整單自己的語彙，不是撥款的。** 兩者只有 `pending` 重疊 ——
> 送 `paid`（撥款的狀態）會得到 422。

### 回應 200

```json
{
  "success": true,
  "message": "取得調整單列表。",
  "data": [
    {
      "id": 15,
      "adjustment_no": "AD20260818000002",
      "shop_id": 21,
      "type": "chargeback",
      "amount": -1340,
      "reason": "信用卡拒付追扣",
      "status": "pending",
      "reference_type": "chargeback",
      "reference_id": "9",
      "created_by_admin_id": 4,
      "approved_by_admin_id": null,
      "approved_at": null,
      "settled_payout_id": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 5, "last_page": 1 } }
}
```

| 欄位 | 值 |
| --- | --- |
| `type` | `refund_after_payout`（撥款後退款）\| `chargeback`（拒付追扣）\| `reconciliation`（對帳更正）\| `manual`（人工調整） |
| `status` | `pending`（待核准）\| `approved`（已核准待結算）\| `settled`（已在某期撥款結清）\| `cancelled`（已撤銷） |
| `amount` | **可正可負**，負數表示從賣家撥款中扣回 |
| `settled_payout_id` | 於哪一張撥款單結清 |

---

## 12. 建立調整單

```
POST platform/adjustments
```

需要 **`adjustment.create`**（財務）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 須存在 |
| `type` | string | ✅ | `refund_after_payout` \| `chargeback` \| `reconciliation` \| `manual` |
| `amount` | integer | ✅ | **不可為 0**，可為負 |
| `reason` | string | ✅ | 最長 500 |
| `reference_type` | string\|null | — | 最長 40 |
| `reference_id` | string\|null | — | 最長 40 |

**回應 201**，`data` 為調整單，`message`：`調整單已建立，待核准。`

---

## 13. 核准調整單

```
POST platform/adjustments/{adjustment}/approve
```

需要 **`adjustment.approve`**（平台管理者）。無請求 body。

回應 200，`data` 為調整單（`status` 為 `approved`），`message`：`調整單已核准。`

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 沒有 `adjustment.approve` |
| 422 | **建立者核准自己開的單**（§15.3 職責分離） |
| 422 | 調整單不在 `pending` |

> 前端請不要把「建立」與「核准」做成同一顆按鈕或同一個表單，會必然踩到 422。

---

# 拒付

拒付是平台**無法拒絕的外部強制**，不走退貨退款流程。登錄端點因此**沒有
「要不要接受」的選項**。

## 14. 拒付列表

```
GET platform/chargebacks
```

需要 `chargeback.manage`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending` \| `responded` \| `won` \| `lost`。**可逗號分隔** |
> **`status` 接受逗號分隔的多個值**（例如 `status=open,watching`）。
> [營運總覽](02-overview.md#links--每一個數字自己的去處)的風險數字是**兩個狀態的
> 聯集**，而總覽的 `links` 帶的正是這種寫法 —— 一次只吃一個值的話，
> 那個數字會「數得出來卻列不出來」。每一段都必須是允許值，否則整個請求回 422
> 並列出允許值。

| `shop_id` | integer | 限定店面。與退款一樣走子訂單 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

> **`status` 用的是拒付自己的語彙，不是撥款的。** 兩者只有 `pending` 重疊：
> 送 `paid`（撥款的狀態）會得到 422。

### 回應 200

```json
{
  "success": true,
  "message": "取得拒付列表。",
  "data": [
    {
      "id": 9,
      "order_sub_no": "SO20260810000127",
      "amount": 1340,
      "status": "pending",
      "notified_at": "2026-08-18 09:00:00",
      "response_deadline_at": "2026-09-01 09:00:00",
      "responded_at": null,
      "resolution": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 2, "last_page": 1 } }
}
```

`status`：`pending`（待抗辯）\| `responded`（已送出抗辯）\| `won`（抗辯成立）\| `lost`（抗辯失敗）。
`response_deadline_at` 依 `chargeback_response_days`（**工作天**）推算。

---

## 15. 登錄拒付

```
POST platform/chargebacks
```

需要 `chargeback.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `order_sub_no` | string | ✅ | 最長 40，子訂單編號 |
| `amount` | integer | ✅ | ≥ 1 |

**回應 201**，`data` 為拒付單，`message`：`拒付通知已登錄。`

登錄時的自動處理：

- **尚未撥款**：該筆子訂單自動被撥款資格條件擋下，不會出款。
- **已經撥款**：**同時建立一張負向調整單**追扣。兩件事必須一起做，
  否則那筆已經出去的錢就沒有任何地方記載該追回。

---

## 16. 拒付定案

```
POST platform/chargebacks/{chargeback}/resolve
```

需要 `chargeback.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `won` | boolean | ✅ | `true` 抗辯成立、`false` 抗辯失敗 |
| `resolution` | string | ✅ | 最長 500 |

回應 200，`data` 為拒付單，`message`：`拒付案件已定案。`

| `won` | 結果 |
| --- | --- |
| `true` | 狀態 `won`，**連帶撤銷先前建立的追扣調整單** —— 那筆錢不該留在賣家帳上 |
| `false` | 狀態 `lost`，追扣調整單維持有效，於下期撥款扣抵 |
