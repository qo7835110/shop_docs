# 10 個資權利

對應 spec X-04、X-04.2、X-04.3、§2.3。

## 三種申請

| `type` | 你要的東西 | 平台做什麼 |
| --- | --- | --- |
| `export` | 匯出我的個人資料 | 產製檔案供你下載 |
| `correction` | 更正錯誤的資料 | 轉會務端查證後更正 |
| `delete` | 刪除我的資料 | **去識別化**，見下 |

## 申請的對象是「這一筆會籍」，不是整個人

四種申請都以**登入的那筆會籍**為範圍（`data_request.member_id`）。同一個人
在兩個工會各有一筆會籍時，刪除其中一筆，另一筆完全不受影響 —— 訂單、評價、
資料都還在。

這與平台其他以會籍為單位的規則一致（會員價、開店資格、未取貨額度）。
要刪掉全部，就是逐筆會籍各申請一次。最後一筆完成後，登入帳號才會一併停用。

## 「刪除」執行的是去識別化，而且不可還原

你的姓名、聯絡方式與收件資訊會以**不可還原**的方式清除，但
**訂單編號、金額與狀態依法定保存義務保留**（§X-04.2）。

這不是打折的刪除 —— 交易紀錄的保存年限是 `personal_data_retention_years`
（預設 7 年），那是法規要求，平台沒有裁量空間。

---

## 1. 我的申請

```
GET buyer/data-requests
```

**分頁**。

```json
{
  "id": 6,
  "type": "delete",
  "status": "pending",
  "reason": "不再使用本平台",
  "blocked_reasons": null,
  "estimated_available_at": null,
  "decision_reason": null,
  "handled_at": null,
  "executed_at": null,
  "download_available": false,
  "download_expires_at": null,
  "pending_days": 3,
  "next_review_at": null,
  "created_at": "2026-08-17T11:00:00+08:00"
}
```

| 欄位 | 說明 |
| --- | --- |
| `status` | `pending`（待審核）\| `blocked`（有未結案事項）\| `approved`（已受理待執行）\| `rejected`（未受理）\| `completed`（已執行） |
| `blocked_reasons` | `blocked` 時列出**具體阻礙**（見第 2 條的形狀） |
| `estimated_available_at` | 預估可受理的時間或說明 |
| `decision_reason` | 平台的裁定理由 —— 不受理時這裡就是原因 |
| `download_available` | **僅 `export` 且已完成時為 `true`**，此時可打第 4 條 |
| `download_expires_at` | **這份檔案什麼時候會被清掉**（`data_export_retention_days`，預設 30 日曆天，自執行日起算）。沒有檔案時為 `null` |
| `pending_days` | 已經等了幾天。已結案（`completed` / `rejected`）為 `null` |
| `next_review_at` | **系統下一次重新評估的時點**，只有 `blocked` 有值 |

### 三個時間欄位補的是同一件事

這條流程有法定意義，卻原本**完全沒有時間軸**：不知道等了多久、
不知道系統什麼時候會再看一次、不知道檔案什麼時候消失。

> **`pending_days` 不是「還要多久」。** spec 沒有給平台側的處理期限參數，
> 系統因此不發明一個 —— 它只告訴你「已經多久了」。

---

## 2. 刪除申請的可受理性

```
GET buyer/data-requests/eligibility
```

**送出刪除申請之前先打這一支** —— 它會直接告訴你現在能不能受理，
而不是讓你送出後才被擋。

```json
{
  "success": true,
  "message": "取得刪除申請的可受理性。",
  "data": {
    "can_request_deletion": false,
    "blocked_reasons": [
      {
        "code": "orders_in_progress",
        "detail": "尚有 14 筆訂單未完成履約或退貨退款 (spec §X-04.3)。",
        "link": "buyer/orders"
      },
      {
        "code": "open_disputes",
        "detail": "尚有未結案的爭議或申訴：退款 5 件、未取貨申訴 0 件、違規申訴 0 件 (spec §X-04.3)。",
        "link": "buyer/refunds"
      }
    ],
    "estimated_available_at": "待上列事項全部結案後即可受理；各項的處理期限依系統參數與該筆單據的狀態而定，您可於買家中心追蹤各筆的進度。",
    "notice": "刪除申請經受理後執行的是**去識別化**：您的姓名、聯絡方式與收件資訊將以不可還原的方式清除，但訂單編號、金額與狀態會依法定保存義務保留 (spec §X-04.2)。此操作不可還原。"
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `can_request_deletion` | 現在能不能受理 |
| `blocked_reasons[].code` | 阻礙的類型代碼，供前端做導向 |
| `blocked_reasons[].detail` | **可直接顯示的完整句子**，含筆數與 spec 依據 |
| `blocked_reasons[].link` | 該去看哪一份清單。「尚有 14 筆訂單未完成履約」的下一個動作是「**哪 14 筆**」，而那份「code → 去哪裡」的對照表放在前端就會漂移 |
| `estimated_available_at` | 一段說明，不是時間戳 —— 各項期限依單據狀態而定，給不出確切日期 |
| `notice` | **請照實顯示**。這是唯一一次告知「不可還原」的機會 |

> **`blocked_reasons` 是必須向你說明的**（§X-04.3）：不能只回一個「不可申請」，
> 那不算說明。前端請把 `detail` 逐條列出，並連到 `link`。

四種 `code` 與各自的去處：

| `code` | `link` |
| --- | --- |
| `orders_in_progress` | `buyer/orders` |
| `open_disputes` | `buyer/refunds`（未取貨申訴與違規申訴在各自的清單上，`detail` 已逐項列出筆數） |
| `open_financials` | `seller/payouts?shop_id={shop}` |
| `shop_not_closed` | `seller/shops/{shop}` |

---

## 3. 提出申請

```
POST buyer/data-requests
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `type` | string | ✅ | `export` \| `correction` \| `delete` |
| `reason` | string\|null | **`correction` 時必填** | 最長 1000 |

**回應 201**，`data` 為申請單。**`message` 有兩種**：

| 情況 | `message` |
| --- | --- |
| 可受理 | `申請已送出，由平台審核。` |
| 有未結案事項 | `申請已記錄，但目前尚有未結案事項，須先結案才能受理 (spec §X-04.3)。` |

**第二種情況仍然回 201，申請也真的被記錄了** —— 它不是失敗。
`blocked_reasons` 會填上阻礙，你不必重送。

### 失敗

| 狀態 | 情況 | `message` |
| --- | --- | --- |
| 422 | 同一會籍已有**同類型**且尚未結案（非 `completed`／`rejected`）的申請 | `您已有一件同類型的申請正在處理中，請待處理完成後再提出。` |
| 422 | 此會籍已完成去識別化 | `此會籍已完成去識別化，無法再提出申請。` |
| 422 | `correction` 未填 `reason`、`type` 不在列舉內 | 驗證錯誤 |

遇到第一種時請引導使用者到第 1 條的清單查看那一件的進度，不要讓他重送。

**「再審」是自動的**：`privacy:reevaluate-blocked` 每日重新評估暫緩中的申請，
阻礙消失即送回**待審**佇列（不是自動受理 —— 受理仍是平台的明確動作）。
`next_review_at` 就是下一次評估的時點，請把它顯示出來：
那四條是**暫緩，不是拒絕**（§X-04.1），而一件永遠不會被再看一次的申請
與被拒絕沒有分別。

阻礙仍在時，`blocked_reasons` 的筆數會跟著更新 —— 你追蹤的是那幾個數字在變小。

`correction` 的 `reason` 必填是因為**會務端要據以查證** ——
只寫「資料有誤」查不了。

---

## 4. 下載匯出檔

```
GET buyer/data-requests/{dataRequest}/download
```

回應是**檔案串流**（`personal-data.json`），不是 JSON 信封。

只有 `type = export` 且 `status = completed`（`download_available` 為 `true`）
的申請下載得了。

### 回 404 的兩種情形

| 情況 | 為什麼是 404 |
| --- | --- |
| 別人的申請編號 | 那筆申請對你而言不存在；回 403 等於告訴你那個編號有效（§2.3） |
| 檔案已過期或清理 | 那是檔案生命週期，不是權限問題 —— 見 `download_expires_at` |

兩者回應相同是刻意的 —— 呼叫端不需要、也不應該分辨得出來。

> 檔案路徑**不會**出現在任何回應裡：路徑是伺服器內部的儲存位置，
> 交出去等於交出一條繞過授權的下載路徑。

### 檔案會過期，請在期限內下載

保留期是 `data_export_retention_days`（預設 **30 日曆天**），
由 `privacy:prune-exports` 每日清理。清掉的是**檔案**，申請紀錄留著（§2.3）——
因此 `status` 仍是 `completed`，而 `download_available` 會翻成 `false`。

匯出檔是一份完整的個資副本（姓名、電話、地址、訂單、金額）。
永久留存等於在儲存層長期多放一份你的全部資料，與 §X-04 整節的目的相反。
