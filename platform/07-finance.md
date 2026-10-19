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
| `processed_by_admin_id` / `provider_refund_id` | 執行退款的財務人員與退款識別；目前 demo 模擬入口存入的是模擬標記，不是外部成功證據。 |
| `payment_method` | 原付款管道。退款一律退回原管道，因此**執行退款前**就需要它 —— `provider_refund_id` 是退款**之後**才有值的那一個。只有管道別，沒有卡號或帳號（§2.3）。 |
| `order_sub_status` | 對應的子訂單狀態。 |
| `return_request` | 關聯的退貨申請；免退貨路徑為 `null`。**裁定爭議要用的三個說法就在這裡**：買家的 `dispute_reason`（在退款單上）、賣家的 `inspection_note`、雙方的 `attachments`。少了它們，客服只能改用資料庫或另外開票去問，而 `dispute_resolution_days` 就燒在那上面。 |
| `pending_actions` | 平台的待辦，**分屬兩種身分**（§15.3）：`disputed` → 待客服裁定（`refund.dispute`）；`processing` → 待財務執行退款（`refund.process`）。物流退回尚待檢時另含等待賣家驗收／缺行事曆提示，即使財務已退款亦保留。 |
| `returned_shipment` | 物流退回實體紀錄或 `null`；含 `received_at`、`inspection_deadline_at`、`inspected_at`、`inspection_result`、`inspection_note`、`auto_resolved`，三方讀取同一份事實，完整語意見[買家文件](../buyer/05-returns-and-refunds.md#物流退回的實體收件驗收1441)。不是財務退款狀態，也不包含操作者身分。 |

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
| `false` | 退款單 → `closed`；子訂單依 §14.4.0 回復：原已完成則維持，原已送達則依原期限決定已送達或已完成，不重新起算期限 |

裁定以鎖定後的最新狀態判斷；已裁定或已結案的舊請求回 422，不覆寫原結果、不新增裁定稽核。
裁定、狀態、子訂單回復與通知建立同交易提交，失敗全部回滾。

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

目前此入口僅提供 **local／testing 的 demo 退款模擬**（§14.4.3），不向金流送出退款。
原成功付款必須為 `demo`，主訂單為 `paid`／`completed`，且退款沒有任何外部執行紀錄。
PAYUNi／其他供應商、production／staging、缺成功付款或已有 G1 執行紀錄皆拒絕；
即使 G1 通用執行紀錄為成功，也不能由手填識別碼代替本筆退款核實。
真實 PAYUNi 送出與完成證據接線仍待 Plan31-4／D5，不能將本端點當作正式退款 fallback。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `provider_refund_id` | string\|null | — | 最長 64；本入口只作本地模擬標記，非外部退款成功證據 |

### 回應 200

`data` 為退款單（`status` 為 `processed`），`message`：`本地模擬退款已完成。`

> **冪等**：在上述條件下重送同一筆已 `processed` 的模擬退款會回既有結果，不覆寫原識別或新增稽核／通知。
> 結案、子訂單／評價／銷量副作用、稽核與通知建立同交易提交，任一步失敗皆回滾。

### 回應 422

| 情況 | message |
| --- | --- |
| 非 local／testing | `退款模擬僅允許 local／testing 環境 (spec §14.4.3)。` |
| 非 demo 原成功付款、主訂單未付款或已有外部執行 | `尚未取得可核實的外部退款成功證據，不能將退款標為完成 (spec §14.4.3)。` |

拒絕不變更退款／付款／庫存／稽核／通知，也不外呼；付款成功後的退款不改原始應付金額。
財務 `processing` 待辦顯示 `待財務執行退款；本地模擬入口不適用真實付款，須核實外部結果。`。
買家／賣家看到的 demo 結案與通知會明示「本地模擬，未向金流送出退款」。

副作用（前端可預期畫面上會跟著變）：

- 子訂單結算，撥款金額扣抵。
- 若為全額退款，該筆交易的評價會被標註「此筆交易已全額退款」。
  **評價本身保留、仍計入統計**，不會被隱藏 —— 否則賣家就有動機以退款換掉負評。

---

## 3a. PAYUNi 原退款查核歷史

```
GET platform/refunds/{refundNo}/payuni-query
```

需要目前有效的 **`refund.process` + `settlement.reconcile`** 雙權限，僅限原 sandbox 代理商付款。
會員 token 在認證層拒絕；客服與一般平台管理員不因此取得財務權限。
不接受查詢參數；只讀原退款持久占用的最新查核，不外呼、不回退一般付款或舊 matched 觀察。
此處沒有占用便回 422，不補造來源；有占用但尚無退款查核時 `data.query=null`。

回應 200，`Cache-Control: private, no-store`。`data`：

```json
{
  "query": {
    "query_id": 1,
    "started_at": "2026-10-07T09:00:00+00:00",
    "observation": {
      "id": 1,
      "outcome": "matched",
      "failure": null,
      "recorded_at": "2026-10-07T09:00:01+00:00"
    }
  },
  "completion": null,
  "refund_completion_verified": false
}
```

已建立工作但尚未保存觀察時 `query.observation=null`；matched 只代表原付款交易對上，**不是退款完成**。
不公開工作鍵、商家／交易外部識別、金鑰、原始封包或私有路徑。停止收款或撤銷原憑證不阻止有權者讀歷史。

`completion` 為原退款的持久完成摘要或 null，與當次讀取的查詢觀察分開判定。
已完成時包含 `completion_id`（整數）、`refund_status=processed`、`execution_status=succeeded`、`refund_completion_verified=true`；外層同名旗標亦為 true。
若經 3g 人工同額核對，另含 `completion_basis=provider_portal_review`、`manual_review_id`（整數）；這是人工核對依據，不冒充供應商獨立退款序號。
若此退款在原交易 UNi 入帳核實後送出，完成摘要另含 `merchant_settlement`：`adjustment_recorded=true`、`refund_amount`（TWD 整數）、`provider_debit_verified=false`、`recovery_authorized=false`。前兩者表示已同交易保存不可變應計調整；後兩者明確表示尚未核實商家 UNi 實際扣回，也未授權再追扣。未有此類調整的舊案／撥款前退款不含此物件，不能以欄位缺少推論已扣回。原 grant／調整的內部 ID 不公開。
GET 永遠不採用結果，即使自動採用開關開啟；已完成的歷史不因關閉開關或觀察過期變回未完成。

## 3b. PAYUNi 原退款單次查回

```
POST platform/refunds/{refundNo}/payuni-query
```

與 3a 相同雙權限及原占用範圍；只對 PAYUNi M3 查詢，不送 M4 退款。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `job_key` | string | 是 | 正規小寫 UUID，原值比對；不可前後空白。不接受其他欄位 |

新鍵先提交原占用／查詢／嘗試與稽核，再在交易外單次查詢，回應形狀同 3a。
**同鍵只讀原查詢工作，不再次外查**，即使尚無結果也不重查；不能挪用其他退款、付款、通知或恢復工作的鍵。
原工作結果不明時先以原鍵讀回，不清除占用或重新退款。只有明確要再查原交易時才用新鍵建立另一筆唯讀查詢。
原憑證不可用、查無或遠端不明只保存 unknown，不代表失敗或可重送。回覆在途撤權仍保存觀察，但拒絕向已撤權者回傳。
收款及退款送出開關不影響此唯讀查回；此入口不啟用任何開關。
只有環境為 sandbox 且獨立 `PAYUNI_REFUND_AUTO_COMPLETION_ENABLED=true`（預設 false）時，POST 查回後會嘗試自動結案。
結案交易重新鎖後核對原受理回條、最新成功退款狀態／本單金額、sandbox UTC+8 發動時間窗及可退餘額，不採信 caller 摘要或僅父交易 matched。
完整證據、退款／G1 終態、業務副作用及解除占用原子提交；不足或採用回滾仍回 `completion=null`，不表示可以重送退款。
原查詢已保存但採用前中斷，可沿原 `job_key` 再 POST 恢復採用，仍不再次 M3／M4；採用始終重驗最新事實，不挑舊成功。
提交後回覆遺失會重新讀回持久完成事實；目前授權失效仍拒絕。HTTP 200 本身不等於完成，須檢查 completion 與完成旗標。
與既有財務付款查核相同，POST 套用每分鐘 10 次限制，超過回 429；限流不重設原工作鍵或退款占用。

## 3c. PAYUNi 退款本地操作鏈核對

```
GET platform/refunds/{refundNo}/payuni-correlation
```

與 3a 相同雙權限；不接受參數，不外呼、不採用結果。只核對原占用、基線、首次受理回條與最新 M3，
回條缺漏、查詢未完成／過期、不符、爭議或同額多次無法歸屬時 422，不退回舊成功。

回應 200，`Cache-Control: private, no-store`；`data` 包含 `submission_id`、`receipt_id`、`baseline_observation_id`、
`query_id`、`observation_id`（皆整數）、`amount`、`remaining_refund_amount`（TWD 整數）、`fresh_until`（ISO 8601），以及：

- `local_correlation_consistent=true`：本地操作鏈條件相符。
- `provider_initiation_time_verified=true`：依業主批准的 sandbox 實測 UTC+8 對應，RefundDay 位於原占用至受理回條時間範圍（含兩端）；無效日期或超出範圍回 422，不加時間寬限、不延伸 production。
- `provider_initiated_at_utc`（ISO 8601）及 `provider_time_basis`：正規化發動時間及判定依據；RefundDay 以台灣時間（UTC+8）解讀，sandbox 為 `sandbox_observed_utc_plus_8_v1`（實測政策），正式為 `payuni_taiwan_time_utc_plus_8_v1`（業主 2026-10-18）。
- `refund_completion_verified=false`：**沒有結案**，退款／G1／庫存／付款／占用均不改。

此摘要不是可傳回伺服器的結案憑證，也不授予退款重送權。既有 `/process` 仍為 demo-only。
共通失敗：401 未登入／錯誤主體、403 缺任一權限或非代理商憑證、404 退款不存在、422 輸入或來源不符（包含非 sandbox）。

---

## 3d. PAYUNi 原退款來源準備

```
POST platform/refunds/{refundNo}/payuni-origin
```

需要目前有效的 **`refund.process`**，僅平台財務；不接受 body 欄位。
前置為原商城 PAYUNi sandbox 信用卡已成功付款、退款在 processing，原 session 最新 M3 必須已核實全額請款與一次付清。
可先透過既有原付款 `payuni-query` 明示查回（該查詢另需 `settlement.reconcile`），不得拿 Lab run、後台截圖或手填交易識別替代。

此操作只原子建立／讀回既有 G1 及原付款來源，不取得送出權、不外呼。
重播不換商家、金額、憑證或證據；既有無來源 G1 不可補貼。新鮮度、保守期限、可退額度及互斥仍須在送出時重驗。
回應 200 `data` 為 `origin_id`、`execution_id`（整數）、`created_at`（ISO 8601）及 `refund_completion_verified=false`，no-store。

## 3e. PAYUNi 原占用／首次回條讀回

```
GET platform/refunds/{refundNo}/payuni-submission
```

需要目前有效的 **`refund.process`**，不接受任何參數、不外呼。原來源不存在回 422；已有原來源而尚未占用時 `data=null`。
送出開關關閉或原憑證撤銷不阻止有權者讀回；仍沿原主幹加鎖核對，不能以 caller 指定別筆占用。
回應 200、no-store：

```json
{
  "submission_id": 1,
  "execution_status": "unknown",
  "receipt_id": 1,
  "receipt_classification": "accepted",
  "refund_completion_verified": false
}
```

上述為 `data` 的形狀。尚未保存回條時 receipt_id／receipt_classification 為 null，可能是送出前中斷或已送出但結果遺失，**不能重送**。
accepted 僅代表受理，不是退款已完成；unknown 同樣不可推定失敗。回應不包含外部交易、原封包或憑證。

## 3f. PAYUNi 明示單次退款送出

```
POST platform/refunds/{refundNo}/payuni-submission
```

需要目前有效的 **`refund.process`**，部署環境須為 sandbox 且 `PAYUNI_REFUND_SUBMISSION_ENABLED=true`。
**預設 false，目前整段驗收未完成，不因文件有此入口便啟用或在封測發退款。** 此為未來逐筆操作入口，不是可整批執行的探測。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `confirm_refund_no` | string | 是 | `R` 加 16 碼數字，原值須與路徑原退款編號完全一致，不 trim |
| `exclusive_operation_confirmed` | boolean | 是 | 必須是 JSON true；確認從發動至查核結束僅從商城操作，不並行在 PAYUNi 後台退款／取消退款 |

拒絕多餘欄位，不接受金額、商家、TradeNo、占用主鍵、acquired 或假完成回條。金額及來源全部沿原退款主幹由伺服器取得。
只有本次新取得的持久占用才可在再次查核、完整提交後鎖外送一次 M4；先保存首份回條再授權讀回。單次 HTTP 不重試。
既有占用一律讀原紀錄，即使沒有回條也不能恢復送出權；但送出開關關閉時 POST 拒絕，應改用 3e 的 GET 讀回。

原交易已有 grant 占用時，只有該筆原 UNi 入帳核實完成、且新的 M3 工作不早於核實時間，才可依原額度／時效／權限規則送出退款。
伺服器永久保存原 `grant_completion_id`，但請求不接受此欄位，回條亦不公開為解鎖 token。
不必等待整張 payout 的其他交易，亦不因手動標 `paid` 而放行。未知 grant、缺證據或挪用別筆證據均拒絕。
退款處理不改原 paid 撥款、付款額或再次異動庫存；買家退款成功與商家 UNi 扣回分開核實，不自動另建待追扣。
送出前資格、憑證、期限或基線變動即停止，已建立占用則保留查核，不能手動刪除重做。回覆途中撤權仍保存結果，但不向撤權者回傳。

回應 200 `data` 同 3e，**固定 `refund_completion_verified=false`，不改退款 processed、不解除占用、不通知退款完成**。
失聯或 5xx 先 GET 3e，後續 M3 使用 3b；不要因回應失敗而重新退款。每分鐘 10 次限制，超過回 429 不重設占用。
401 主體／登入錯誤、403 無財務權、404 原退款不存在、422 未明示確認、輸入／來源不符、開關關閉或送出條件未成立。

---

# 金流對帳（僅財務）

匯入與比對是**兩個權限**，因為後果不同等級：匯入只是把外部檔案放進系統，
比對則會改變付款的可出款狀態。

## 3g. PAYUNi 同額退款人工核對結案

```
POST platform/refunds/{refundNo}/payuni-manual-completion
```

P-02／§14.4.3，僅 sandbox，須目前有效的 `refund.process` + `settlement.reconcile`。
用途是已受理的第二次同額退款：財務已在 PAYUNi 後台逐筆確認原交易及兩次不同時間的退款成功，補足自動查核無法區分同額退款的歸屬。不是手動指定退款成功，也不是退款送出入口。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `submission_id` | integer | 是 | 本次原退款占用 ID，JSON 正整數 |
| `query_id` | integer | 是 | 原交易最新退款查詢 ID，JSON 正整數；觀察須仍在原 TTL 內 |
| `merchant_trade_no` | string | 是 | 後台商店訂單編號，須與伺服器原交易完全一致，最多 100 字元 |
| `trade_no` | string | 是 | 後台 UNI 序號，須與伺服器原交易完全一致，最多 100 字元 |
| `previous_refund_at` | string | 是 | 前一筆成功退款的台北時間，格式 `YYYY-MM-DD HH:mm:ss`，須與已完成退款證據一致 |
| `refund_at` | string | 是 | 本次成功退款的台北時間，同格式；須晚於前筆、等於最新觀察，且位於原占用至回條時間窗 |
| `provider_history_confirmed` | boolean | 是 | 必須為 JSON `true`，確認已逐筆核對後台原交易退款歷程 |
| `reason` | string | 是 | 非空核對理由，最多 500 字元，永久稽核保存 |

拒絕額外欄位，尤其金額、狀態、外部收據摘要與成功旗標。伺服器沿原鎖序重讀目前授權、退款／付款／受理回條、最新成功觀察、請款金額、退款餘額與時間；缺資料、爭議、非同額、前筆未有完成證據、過期或錯單均回 422（缺權限為 403）。不回退舊成功，不以後台截圖取代最新 API 觀察。

200 回應 `data` 為完成摘要：`completion_id`、`refund_status=processed`、`execution_status=succeeded`、`refund_completion_verified=true`、`completion_basis=provider_portal_review`、`manual_review_id`。附 `Cache-Control: private, no-store`，不回傳外部交易識別碼。若有撥款後應計調整，另含 3a 所列 `merchant_settlement`。

人工核對紀錄、完成證據、退款／G1 終態、通知、稽核及解除占用同交易提交。相同內容重播回傳原完成，即使觀察已過期；不同內容不覆寫原證據，仍需目前授權。提交後不明先 GET 3a 讀回，不重新 M4。此入口不外呼、不改開關，不依賴自動結案開關；每分鐘 10 次限制。

此結案不表示銀行入帳、UNi 實際扣回或整體驗收完成。一般同額退款的自動採用限制維持不變。
路徑 A 依已完成退款累計判定子單終態：累計全額為 `refunded`，部分為 `completed`；未出貨取消仍維持 `cancelled`。原付款及已付款主單不取消，不再次異動庫存。

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

目前通用匯入／比對僅供 `local`／`testing` 的 `provider=demo` 模擬，其他環境或供應商回 422。
PAYUNi 必須另接具商家、環境、來源種類、完整日期範圍及分頁證據的資料來源；
手填交易序號、金額或缺費用補 0 不能成為付款／費用已核實的證明。
既有檔案仍可查閱，但未核實的真實供應商檔案不能透過本端點執行比對。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `provider` | string | ✅ | 最長 40；目前 Service 僅允許本地／測試環境的 `demo` |
| `period` | string | ✅ | 最長 20，結算期別 |
| `original_name` | string\|null | — | 最長 255，原始檔名 |
| `declared_count` | integer | — | 對方宣稱的筆數 |
| `declared_amount` | integer | — | 對方宣稱的總額（整數 TWD） |
| `lines` | array | ✅ | 至少 1 筆 |
| `lines[].provider_transaction_id` | string | ✅ | 最長 64 |
| `lines[].amount` | integer | ✅ | 整數 TWD |
| `lines[].fee` | integer | — | 手續費 |
| `lines[].settled_at` | date\|null | — | 撥付日（台北時間，見 [conventions](../conventions.md#時間)）；只給日期＝台北當天 00:00 |

```json
{
  "provider": "demo",
  "period": "2026-08-W2",
  "original_name": "demo_20260816.csv",
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
    "provider": "demo",
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
> 為 `false` 時，比對會回 422，保留匯入資料供調查，不改付款、不建立差異或放行撥款。
> 為 `true` 只表示總數與明細相符，不表示來源真實或資料範圍完整。

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

目前僅允許 `local`／`testing` 的 `demo` 檔案。Service 在交易內鎖定檔案與明細，
以保存的供應商及總數檢查，不採用呼叫端修改過的模型。
未核實供應商、空明細或總筆數／總金額不符均回 422；重播相同拒絕也不改變任何對帳事實。
這不是 PAYUNi 正式對帳接入，不能用模擬成功回應證明金流閉環。

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

## 6a. PAYUNi 原付款核對採用

```
POST platform/payments/{paymentNo}/payuni-reconciliation
```

需有效平台財務管理員及 `settlement.reconcile`，僅限 sandbox 代理商原付款。
先從原付款 `payuni-query` 歷史取得觀察 ID，再明確提出採用；GET 查核歷史本身不會採用。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `observation_id` | integer | ✅ | 正整數，必須屬於原付款的最新完整觀察；已採用原觀察可重播 |

不接受額外欄位（包括金額、費用、外部交易號與相符旗標）。伺服器在原主幹鎖內
重驗目前權限、原憑證、最新查核、時效、凍結原額、全額請款、權威費用、已處理退款及剩餘可退額。
本操作不外送 PAYUNi，不新增付款、退款或撥款，也不更改原付款金額及費用。

回應 200 的 `data`：

```json
{
  "outcome": "matched",
  "reconciliation_id": 1,
  "discrepancy_ids": [],
  "scope": "known_payment",
  "payout_ready": false
}
```

`matched` 保存不可變採用證據並更新付款對帳投影；費用或可退餘額不符時
`outcome=mismatched`、`reconciliation_id=null`，`discrepancy_ids` 列出逐項建立的差異案件。
**HTTP 200 不表示一定相符。** 缺欄位／未知／過期／錯配或尚有未結案差異時回 422，不補零、不放行。
原觀察重播返回原證據或案件 ID，不重複寫入，也不覆寫其後改變的對帳狀態或重開案件。
即使重播 `outcome=matched`，也只能解讀為該觀察當時的採用結果，不能當成目前可撥款。
回應 `Cache-Control: private, no-store`；401 未認證／錯 guard、403 缺財務權、404 查無付款。
此為單筆已知交易範圍，不是全期間缺帳判定、UNi／銀行入帳或 31-7 全部驗收；撥款另驗完整條件。

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

PAYUNi 原交易核對產生的差異也沿用本入口。`context.component` 分別以
`payment_fee`／`remaining_refund_amount` 標示費用或剩餘可退額差異，
`provider_value`／`local_value` 為當次保存的兩側數值，不是新的應收或退款金額。
原觀察來源與差異事實不可更改／刪除。處理時鎖定原付款主幹並重驗有效財務權限，
狀態、說明及稽核一起提交；同狀態、同說明的重播不重寫結案時間及稽核。
`open`／`watching` 都會阻止新的相符採用；改為 `resolved` 本身也不會把付款標成相符，
仍須另有最新完整查核並通過第 6a 節採用條件，且不代表可撥款；本結案端點不代為採用。

---

# 撥款

```
撥款金額 = 商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整
```

回應中的 `formula` 欄位就是這句話，可直接顯示在明細頁。
**退貨運費不進這條算式** —— 賣家自行與物流業者簽約，那筆錢不經過平台。

**沒有「建立撥款單」的端點**：撥款由排程產生候選，不由賣家申請、不由單筆訂單即時觸發。
少一條路，就少一個繞過候選條件的入口。

PAYUNi 候選另需權威付款對帳，以及原商家／UNi 受款與 grant 能力核實。
付款對帳條件已接第 6a 節的不可變採用證據：鎖原主幹後重讀最新查核、現行時效、
原額／費用、已處理退款與未結案差異；新查核尚未採用、憑證已撤銷或證據過期均須重新核實。
單有 `reconcile_status=matched` 不能通過。原商家受款與 grant 核實已接入候選條件；五項條件均通過才可建候選，並非已有對帳就能撥款。
付款成功／來源封存、A3 可收款或本地銀行帳戶已驗證不能代替其餘條件；
未通過的原因不會提示賣家以補銀行帳戶解鎖 grant。
開發資料中的 `PAYUNI-PAYOUT-GUARD` 是明示注入的歷史錯誤候選，只供拒絕假結案測試，並非成功案例。

原交易正向 UNi 證據全部採用後，可由受控 `payuni:complete-payout` 指令核實整單（[操作手冊](../../ops/payuni-payment-inspection.md)）。
此為本地證據採用，不再次外送撥款。列表與賣家詳情均提供 `net_amount_basis: "accrual"` 及
`uni_settlement`（未整單核實為 `null`）；完整欄位見[賣家撥款](../seller/07-payouts.md)。
`paid` 只表示所有原交易 UNi 入帳已核實，不代表平台費收取、工會分潤或銀行到帳。

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

目前僅 local／testing 可執行 **demo 模擬**，不向金流送出指令（§14.4.3／§14.9.3）。
全部原成功付款須為 demo、主訂單已付款／完成、來源明細屬同店且沒有任何 G1 執行紀錄。
正額缺明細、混合非 demo 付款、缺成功付款、既有 G1（即使 succeeded）或來源集合在等待間改變皆拒絕。
production／staging 不接受模擬；PAYUNi 實際釋款／費用／銀行入帳仍待 31-6 接線及核實，不能 fallback。

### 回應 422 —— 模擬邊界拒絕

| 情況 | message |
| --- | --- |
| 非 local／testing | `撥款模擬僅允許 local／testing 環境 (spec §14.9)。` |
| 原付款／明細／外部指令不符 | `尚未取得可核實的逐交易撥款結果，不能由本地模擬入口處理 (spec §14.9.3)。` |

拒絕不改財務單據、稽核或完成通知。`payouts:execute` 同樣拒絕，逐筆保留原狀並繼續其他候選，
失敗摘要及作業告警供營運追查；不把拒絕改成可自動重送的真實 `failed`。

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

通過模擬邊界後回應 200，**仍須用 `data.status` 判斷結果**：

| `status` | `message` | 意義 |
| --- | --- | --- |
| `paid` | `本地模擬撥款已完成。` | 本地模擬完成，`paid_at` 有值，不代表 PAYUNi／銀行入帳 |
| `aborted` | `出款前再驗證未通過，已中止並記錄原因。` | 未通過再驗證，原因在 `aborted_reason` |
| `deferred` | `金額未達門檻或為負，遞延下期繼續抵扣。` | `net_amount` ≤ 0 或未達最低金額 |

> `aborted` 與 `deferred` **都是正常的營運結果，不是錯誤**。前端不要顯示成紅色錯誤訊息，
> 也不要自動重試。
>
> 負數餘額**結轉下期繼續抵扣**，既不截斷為 0，也不向賣家追收現金。

現有失敗入口只供 demo 模擬，不能把任意原因字串當成銀行拒絕證據；同原因重播不累加重試。
真實逾時／查無是未知，不能推定失敗重送。已 paid 不可改為 failed，須另建調整紀錄。

所有撥款讀回新增 `execution_notice`：demo 模擬、外部結果待核實或零／負額遞延提示。
`expected_paid_at` 只在 local／testing、正額 pending、原付款全 demo 且無 G1 時顯示模擬排程時刻，其他為 null。
結案、稽核與通知建立同交易，任何一步失敗全部回滾；完成重播不覆寫時間或重複通知。

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

回應另包含 `payuni_recovery_hold`（布林）與 `recovery_notice`（字串或 null）。旗標 true 表示原 PAYUNi 拒付扣回尚未核實，金額僅應計，禁止一般核准或納入撥款抵扣；即使升級前已核准亦會保留原狀態但停止抵扣。旗標與來源由伺服器／DB 決定，不接受 caller 解除。false 不代表外部扣回已核實。抗辯成立可取消尚未抵扣的原調整，但不表示金流已將扣款退回。

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
- **已經釋款**：同交易保留負向應計調整。PAYUNi 原交易 UNi 已核實即可，不必等待整張 payout 標 paid；其調整固定帶 `payuni_recovery_hold=true`，在外部扣回未知時禁止一般核准及再次追扣，不把拒付通知當作商家已扣回的證明。其他供應商保留既有雙人核准流程。
- 拒付、調整、稽核及通知同交易；故障整筆回滾。原 PAYUNi 拒付只留一筆原調整，重複建立回 422，不改原付款／撥款。

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
| `false` | 狀態 `lost`；PAYUNi 調整仍待核實，不自動追扣，原額撥款仍被阻擋；其他供應商沿既有核准調整流程 |

定案與賣家抗辯共用原訂單鎖並重讀案件。同一結果／理由重播只讀回；矛盾定案或已定案後再抗辯回 422。PAYUNi 敗訴不是外部扣回或剩餘可撥額的證據，不因新的交易查詢相符而放行原額。
