# 04 店面管理

對應 spec P-01、C-01、S-01、S-03、S-10、P-07.2、§11.2。

## 申請單與店面是**兩張表、兩組狀態**（§11.2）

審核期的狀態在**申請單** `shop_applications` 上，店面 `shops` 只承載營業狀態。
**店面於申請核准的同一交易內才被建立** —— 核准之前沒有店面可以指向，
因此代建、代送審與審核決議的對象一律是 `{application}`，不是 `{shop}`。

```
申請單 shop_applications
  draft ──submit──> pending_review ──┬── approve ────> approved   ← 終態
                                     ├── needs_info ─> needs_info ──submit──> pending_review
                                     └── reject ─────> rejected   ← 終態

                                        approve 的同一交易內建立 ↓

店面 shops
  approved ──(賣家自行開業)──> open ──suspend──> suspended ──resume──> open
                                    open / suspended ──close──> closed
```

### `approved` 與 `rejected` 都是**終態**（§12.3-71）

被退回的申請單**不可就地改資料重送**，重新申請必為**新的單號**，
而舊單的退回理由不被覆蓋 —— 那是「為什麼上次沒過」的唯一紀錄。
前端在 `rejected` 的申請單上不要提供「修改後重送」，應提供「重新申請」。

`needs_info`（補件）不同：那不是終態，同一張單補完可再送審。

### 三件事平台**做不到**，前端不要留按鈕

1. **代為開業**（`approved → open`）：何時營業是賣家自己的決定。
2. **賣家自行解除平台停權**：越級解除以 403 拒絕並留稽核。
3. **刪除店面**：關閉取代刪除。

## 誰審核誰（§C-01）

| 店面所屬組織 | 審核者 |
| --- | --- |
| `union`／`organization` | 該工會的管理者（平台亦可介入） |
| `corporation`（法人機構） | **只有平台** |

---

## 1. 店面清單

```
GET platform/shops
```

需要 `shop.review`。**分頁**。跨組織查詢。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `approved` \| `open` \| `suspended` \| `closed` —— **店面狀態只有這四種** |
| `organization_id` | integer | 限定單一組織 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得店面清單。",
  "data": [
    {
      "id": 21,
      "slug": "zhi-ren-shou-zuo-fang",
      "name": "職人手作坊",
      "description": "手工皮件",
      "organization_id": 3,
      "status": "open",
      "is_buyable": true,
      "invoice_capability": "duplicate_only",
      "customer_service_contact": "service@example.com",
      "pickup_note": "週三公休",
      "shipping_note": "下單後 3 個工作天內出貨",
      "return_note": "商品到貨後 7 日內可退",
      "favorite_count": 128,
      "rating_avg": 4.7,
      "rating_count": 63,
      "opened_at": "2026-03-01 10:00:00",
      "owner_member_id": 501,
      "approved_at": "2026-02-27 14:20:00",
      "suspended_reason": null,
      "suspended_by_type": null,
      "closed_reason": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 48, "last_page": 3 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `is_buyable` | **後端算出的可購買性**（僅 `open` 為 `true`）。前端不要自行以 `status` 判斷。 |
| `invoice_capability` | `triplicate`（三聯式）\| `duplicate_only`（僅二聯式）\| `none` |
| `approved_at` | 申請核准（＝店面被建立）的時間。**審核歷程不在店面上**，在申請單上（見第 2 條）。 |
| `suspended_reason` / `closed_reason` | 停權／關閉時填寫的原因，供後台顯示。 |
| `suspended_by_type` | **這筆暫停是誰下的**：`seller`／`union`／`platform`（**不是 `admin`**）。決定誰解得掉 —— 賣家只解得掉自己下的休假，工會解不掉平台停權（§11.2、§P-07.4）。未暫停時為 `null`。 |

---

## 2. 店面詳情

```
GET platform/shops/{shop}
```

需要 `shop.review`。比清單多**一個區塊**：`application`（這間店的來源申請單）。

```json
{
  "success": true,
  "message": "取得店面資料。",
  "data": {
    "…": "同清單單筆",
    "application": {
      "id": 7,
      "application_no": "SA20260227000012",
      "organization_id": 3,
      "applicant_member_id": 501,
      "name": "職人手作坊",
      "slug": "zhi-ren-shou-zuo-fang",
      "status": "approved",
      "is_terminal": true,
      "shop_id": 21,
      "submitted_at": "2026-02-25 09:10:00",
      "decided_at": "2026-02-27 14:20:00",
      "description": "手工皮件",
      "customer_service_contact": "service@example.com",
      "pickup_note": "週三公休",
      "shipping_note": "下單後 3 個工作天內出貨",
      "return_note": "商品到貨後 7 日內可退",
      "invoice_capability": "duplicate_only",
      "review_history": [
        {
          "decision": "approve",
          "from_status": "pending_review",
          "to_status": "approved",
          "reason": null,
          "reviewer_type": "member",
          "reviewed_at": "2026-02-27 14:20:00"
        }
      ]
    }
  },
  "errors": null,
  "meta": null
}
```

> **`review_history` 在 `application` 裡面，不在頂層** —— 審核歷程屬於申請單（§11.2）。
> 沒有來源申請單的店面（例如由部署程序建立的示範資料）`application` 為 `null`。
>
> **這一支不帶 `approval_checklist`**：檢核清單是核准前的判斷依據，而店面存在就代表
> 已經核准過了。要看檢核清單請打申請單的端點（第 3、4、6 條的回應，或第 10 條的
> 申請單版戶名驗證）。

> **`application` 物件裡值為 `null` 的欄位會被省略**，不是回 `null`。
> 上例沒有 `decision_reason`，是因為這張申請單是核准的（核准不必填理由）。
> 前端一律用 `application.decision_reason ?? null` 取值，不要假設鍵存在。

### 核准檢核清單

`approval_checklist` 由**申請單**的端點回傳（第 3、4、6、10 條），
是**四項固定檢核**，順序固定，供審核畫面直接渲染：

```json
[
  { "code": "active_member", "label": "有效會員", "satisfied": true, "detail": null },
  { "code": "membership_paid", "label": "會費已繳", "satisfied": true, "detail": null },
  { "code": "bank_account_verified", "label": "銀行戶名驗證通過", "satisfied": false, "detail": "戶名與申請人姓名不符" },
  { "code": "documents_complete", "label": "附件齊全", "satisfied": false, "detail": "尚缺：身分證明文件、存摺封面" }
]
```


| `code` | 條件 |
| --- | --- |
| `active_member` | 店主會籍狀態為 `active` |
| `membership_paid` | 店主會費已繳 |
| `bank_account_verified` | 撥款帳戶戶名驗證通過（見第 10 條） |
| `documents_complete` | 必要附件齊全，`detail` 會列出還缺哪些 |

四項全部 `satisfied: true` 才能核准。核准端點會再檢查一次，未通過回 422。

`review_history.reviewer_type` 為 `member`（工會管理者）或 `admin`（平台人員）。

---

## 3. 代為建立開店申請

```
POST platform/shop-applications
```

需要 `shop.review`。

**走的是與賣家端完全相同的服務**：資格檢查、發票能力判定與初始狀態一模一樣，
差別只在誰按了按鈕。建出來的是一張 `draft` 的**申請單**（不是店面），
仍須備齊附件與撥款帳戶後送審，仍須經工會審核（§U-02）。

> 同一會員**同時至多一張進行中的申請單**（§12.3-73）。已有進行中的申請時回 422。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `owner_member_id` | integer | ✅ | 須為既有會員 |
| `name` | string | ✅ | 最長 50 |
| `description` | string\|null | — | 最長 2000 |
| `customer_service_contact` | string\|null | — | 最長 100 |
| `pickup_note` | string\|null | — | 最長 1000 |
| `shipping_note` | string\|null | — | 最長 1000 |
| `return_note` | string\|null | — | 最長 1000 |
| `invoice_capability` | string | — | `triplicate` \| `duplicate_only` \| `none` |

### 回應 201

`data` 為**申請單**詳情，含 `approval_checklist`；`review_history` 為空陣列
（草稿還沒有任何審核動作）。
`message`：`已代為建立開店申請草稿；仍須備齊附件與撥款帳戶後送審。`

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | **被代建的那位會員不符開店資格**（會籍非 active、會費未繳、所屬組織不得開店等）。訊息即具體理由，可直接顯示 |
| 422 | 會員不存在 |

> 平台的權限比賣家大，但「誰有資格開店」不因為換人按按鈕而改變。

---

## 4. 代為送審

```
POST platform/shop-applications/{application}/submit
```

需要 `shop.review`。無請求 body。`draft` / `needs_info` → `pending_review`。

**`rejected` 送不了** —— 它是終態，重新申請請建立新的申請單（見第 3 節）。

回應 200，`data` 為申請單詳情，**含 `approval_checklist` 與 `review_history`**
（送審那一筆已經進歷程了），
`message`：`已代為送出審核；本次送審者不得再擔任審核者。`

> **送審者記為平台人員**，之後同一人對這一件呼叫 `decision` 會被服務層擋下（§15.3）。
> 前端請把這句訊息顯示出來，避免操作者按完送審又想按核准。

---

## 5. 更新店面資料

```
PATCH platform/shops/{shop}
```

需要 `shop.review`。平台的跨組織更正，與賣家自行維護是不同的授權路徑。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | — | 最長 50 |
| `description` | string\|null | — | 最長 2000 |
| `customer_service_contact` | string\|null | — | 最長 100 |
| `pickup_note` / `shipping_note` / `return_note` | string\|null | — | 各最長 1000 |
| `invoice_capability` | string | — | `triplicate` \| `duplicate_only` \| `none` |
| **`reason`** | string | ✅ | 最長 200 —— **平台代改一律要理由** |

回應 200，`data` 為店面詳情（不含附加區塊）。

---

## 6. 審核決議

```
POST platform/shop-applications/{application}/decision
```

需要 `shop.review`。適用法人機構申請，以及平台對工會申請的介入。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `decision` | string | ✅ | `approve` \| `needs_info` \| `reject` |
| `reason` | string\|null | `approve` 以外必填 | 最長 500 |

```json
{ "decision": "needs_info", "reason": "請補上營業登記證明的清晰照片" }
```

### 回應 200

`data` 為申請單詳情，**含 `approval_checklist` 與 `review_history`**。
`approve` 時 `shop_id` 會有值 —— **店面在核准的同一交易內建立**（§12.3-73）。
**回應不帶完整的 `shop` 物件**，要拿店面資料請以該 `shop_id` 打第 2 條。
`message`：`審核結果已記錄，並已通知申請人。`

### 狀態轉移

| decision | 申請單結果狀態 | 後續 |
| --- | --- | --- |
| `approve` | `approved`（**終態**） | 同一交易內建立店面，狀態 `approved`；賣家自行開業才會變 `open` |
| `needs_info` | `needs_info` | 賣家補件後可重新送審**同一張單** |
| `reject` | `rejected`（**終態**） | 不可就地改資料重送；重新申請必為**新單號**，舊單的退回理由不被覆蓋 |

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 沒有 `shop.review`；或本人就是送審者（自己送審自己核准） |
| 422 | 申請單不在 `pending_review`；核准時檢核清單未全部滿足；會務同步逾期而無法判定會籍（§U-05.3） |

> **非 `pending_review` 的申請單一律核准不了**（§12.3-72），且未繳費／會籍無效／
> 銀行未驗證／附件缺漏者亦然。阻擋發生在後端，不是只有前端按鈕變灰。

---

> **店面詳情（`GET platform/shops/{shop}`）另外帶 `disposition`**，形狀同
> [08 風控 §6](08-risk.md#6-店面處置現況)的 `disposition` 物件。
>
> 這一欄存在的理由是一個**合法但反直覺**的組合：平台主動撤銷違規記錄會讓等級
> 退回警告，卻**不解除停權**——「須即時解除」只綁在申訴成立那條路上（§P-07.3）。
> 於是這一頁會出現「等級 `warned`、狀態 `suspended`」，而它是對的。
> 少了這一欄，下一位管理者只看得到 `suspended`，會以為系統算錯，
> 然後去找「調整等級」的按鈕 —— 而那個按鈕**刻意不存在**（§P-07）。

---

## 7. 停權

```
PATCH platform/shops/{shop}/suspend
```

需要 **`violation.manage`**（不是 `shop.review`）—— 停權是處置手段。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data` 為店面詳情，
`message`：`店面已停權，既有訂單仍須依約履行。`

**停權 ≠ 免除履約**：既有訂單仍要出貨、處理退貨與撥款，只是不能有新交易。
前端請顯示這句話。

---

## 8. 恢復營業

```
PATCH platform/shops/{shop}/resume
```

需要 `violation.manage`。請求同上（`reason` 必填）。
回應 200，`message`：`店面已恢復營業。`

---

## 9. 關閉店面

```
PATCH platform/shops/{shop}/close
```

需要 `violation.manage`（關閉比停權更不可逆，§11.2 未定義 `closed` 的回頭路，
因此不會比停權更容易取得）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | **尚有進行中的訂單**。必須先讓進行中的訂單走完（出貨、退貨、撥款）才能關閉 |

---

## 10. 撥款帳戶戶名驗證

**兩支端點，兩個對象** —— 核准之前帳戶掛在申請單上，核准之後掛在店面上：

```
PATCH platform/shop-applications/{application}/bank-account/verify   ← 審核期
PATCH platform/shops/{shop}/bank-account/verify                      ← 已開店
```

兩者的**請求**形狀相同，需要 `shop.review`。**回應不同**：申請單那一支多帶
`approval_checklist`（見下）。
**只有平台可以驗證** —— 賣家自行標記驗證通過等於自己核准自己。
申請單那一支的驗證結果會餵進[核准檢核清單](#核准檢核清單)的
`bank_account_verified`；店面那一支則影響撥款候選的第五項條件（§14.9.1）。

> **終態的申請單驗證不了**：`approved` / `rejected` 的申請單呼叫這支回 422
> —— 帳戶要改請改店面那一支（§12.3-71）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `verified` | boolean | ✅ | `true` 標記通過、`false` 標記失敗 |
| `reason` | string\|null | `verified = false` 時必填 | 最長 255 |

```json
{ "verified": false, "reason": "戶名與申請人姓名不符" }
```

### 回應 200

**申請單版**（`shop-applications/{application}/bank-account/verify`）：

```json
{
  "success": true,
  "message": "撥款帳戶驗證狀態已更新。",
  "data": {
    "bank_account": {
      "bank_code": "822",
      "branch_code": "0031",
      "account_name": "王小明",
      "account_number_masked": "******1234",
      "verify_status": "failed",
      "verified_at": null,
      "verify_failed_reason": "戶名與申請人姓名不符",
      "payout_ready": false
    },
    "approval_checklist": [ "…四項檢核，見上…" ]
  },
  "errors": null,
  "meta": null
}
```

**店面版**（`shops/{shop}/bank-account/verify`）：`data` **只有 `bank_account`**，
沒有 `approval_checklist` —— 店面已經核准過了，檢核清單在這個階段沒有意義。

```json
{
  "success": true,
  "message": "撥款帳戶驗證狀態已更新。",
  "data": {
    "bank_account": { "…": "同上" }
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `account_number_masked` | **永遠只有遮罩值**。帳號在資料庫是加密儲存的，沒有解密端點；這個值由末四碼組出，全程不解密。 |
| `verify_status` | `pending` \| `verified` \| `failed` |
| `payout_ready` | 是否可作為撥款收款帳戶。 |

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 422 | 該店面尚未設定撥款帳戶 | `此店面尚未設定撥款帳戶 (spec §S-10)。` |
| 422 | `verified` 為 `false` 但沒填 `reason` | 驗證錯誤 |
