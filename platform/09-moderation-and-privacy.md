# 09 內容處置與個資

對應 spec B-04.1、B-04.4、X-03.4、X-03.5、X-03.6、X-04。

---

# 評價處置

**只有隱藏，沒有刪除，也沒有取消隱藏。**

- 不刪除：隱藏採狀態變更，原文須保留供爭議追溯。
- 沒有取消隱藏：那等同推翻自己的判定，應該是申訴流程的結果，不是一個對稱的按鈕。

隱藏權限是 `review.moderate`，**預設只有平台管理者有，客服沒有** ——
隱藏評價會直接改變店面評分與商品頁，那是營運判斷，不是第一線爭議處理。

## 1. 全站評價列表

```
GET platform/reviews
```

需要 `review.moderate`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `shop_id` | integer | 限定店面。可與 `status` 並存 |
| `status` | string | `visible` \| `hidden` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得評價列表。",
  "data": [
    {
      "id": 771,
      "rating": 2,
      "content": "包裝破損，客服態度不佳",
      "images": ["reviews/771/1.jpg"],
      "reviewer_name": "王＊明",
      "created_at": "2026-08-14 20:11:00",
      "fully_refunded": true,
      "fully_refunded_notice": "此筆交易已全額退款。",
      "append": null,
      "reply": { "content": "已與買家聯繫並致歉", "created_at": "2026-08-15 09:00:00" },
      "status": "visible",
      "hidden_reason": null,
      "product_id": 3301,
      "reported": true,
      "can_reply": false,
      "member_id": 501,
      "order_item_id": 8801,
      "hidden_by_admin_id": null,
      "hidden_at": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 88, "last_page": 5 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `reviewer_name` | **遮罩後**的評價者姓名。 |
| `fully_refunded` | 「先評價、後全額退款」的標註。**評價保留、仍計入統計**，只是標註 —— 不自動隱藏，否則賣家就有動機以退款換掉負評。買賣雙方皆無法編修或關閉此標註。 |
| `append` | 買家追評（`content`、`images`、`created_at`），沒有則為 `null`。 |
| `reply` | 賣家回覆，沒有則為 `null`。 |
| `reported` | 是否有賣家檢舉。 |

---

## 2. 隱藏評價

```
POST platform/reviews/{review}/hide
```

需要 `review.moderate`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

回應 200，`data` 為評價（`status` 為 `hidden`，`hidden_by_admin_id`、`hidden_at` 有值），
`message`：`評價已隱藏，原文保留。`

> 隱藏是**不可逆**的操作（沒有反向端點），且會影響店面評分。前端請做二次確認。

---

# 對話介入與訊息處置

## 客服要讀對話，得先加入；加入會留痕

**沒有「列出全部對話」的端點。** 訊息裡有瑕疵照片、收件資訊與協商內容，
全站可讀等於全站個資可讀。平台人員讀對話的唯一途徑是：因爭議加入該對話。

`leave` 之後就**再也讀不到** —— 否則退出只是一個時間戳，不是一個界線。

流程：`join` → `messages` / `send` → `leave`。

---

## 3. 加入對話

```
POST platform/conversations/{conversation}/join
```

需要 `refund.dispute`（客服角色）。無請求 body。

### 回應 200

```json
{
  "success": true,
  "message": "已加入對話，加入紀錄已留存。",
  "data": {
    "id": 8812,
    "status": "open",
    "order_sub_id": 3312,
    "last_message_at": "2026-08-17 21:40:00",
    "cs_present": true,
    "cs_joined_at": "2026-08-18 10:30:00",
    "cs_left_at": null
  },
  "errors": null,
  "meta": null
}
```

`status`：`open` \| `closed`。`cs_present` 為後端算出的衍生值（已加入且尚未退出）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 沒有 `refund.dispute` |
| 422 | 該對話沒有進行中的爭議（是否有爭議由服務層判斷，不是授權問題） |

---

## 4. 退出對話

```
POST platform/conversations/{conversation}/leave
```

需要 `refund.dispute`。無請求 body。
回應 200，`data` 同上（`cs_present` 為 `false`、`cs_left_at` 有值），
`message`：`已退出對話，退出紀錄已留存。`

> 退出後再呼叫 `messages` 會得到 403，訊息是
> `平台人員須因爭議加入對話後才能檢視內容 (spec §X-03.5)。`

---

## 5. 讀取對話訊息

```
GET platform/conversations/{conversation}/messages
```

需要 **本人已加入且尚未退出**該對話（別的客服加入的對話你也讀不到）。**分頁**。

### 回應 200

```json
{
  "success": true,
  "message": "取得訊息列表。",
  "data": [
    {
      "id": 33,
      "conversation_id": 8812,
      "sender_type": "member",
      "sender_id": 501,
      "content": "可以直接匯款給我，我給你打折",
      "attachments": [],
      "status": "visible",
      "hidden_reason": null,
      "created_at": "2026-08-17 21:40:00",
      "flagged": true,
      "flagged_reasons": ["站外交易用語", "銀行帳號"],
      "hidden_by_admin_id": null,
      "shop": { "id": 21, "name": "職人手作坊" }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 41, "last_page": 3 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `content` / `attachments` | **平台視角看得到原文**，即使該訊息已被隱藏（買賣雙方看到的是「此訊息已由平台隱藏。」）。 |
| `flagged` / `flagged_reasons` | 敏感內容偵測結果。`flagged_reasons` 已轉成中文標籤可直接顯示：身分證號、銀行帳號、外部連結、LINE 帳號、通訊軟體帳號、電子郵件、電話號碼、站外交易用語。 |
| `sender_type` | `member` \| `admin` |
| `status` | `visible` \| `hidden` |
| `shop` | **這則訊息屬於哪一家店**。查證成立的下一步是 `POST platform/violations`，而那支端點要 `shop_id` —— 清單只給 `sender_id` 與 `conversation_id` 的話，`sender_type: member` 可能是買家也可能是賣家，於是**待查清單認不出被查的對象**。對話恆為「一位買家 × 一家店面」（§X-03.1），因此店面是對話的固有屬性。 |

---

## 6. 客服發送訊息

```
POST platform/conversations/{conversation}/messages
```

需要本人已加入該對話。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `content` | string | ✅ | 最長 2000 |
| `attachments` | string[] | — | 每項最長 500 |

**回應 201**，`data` 為訊息單筆，`message`：`訊息已送出。`

---

## 7. 待查訊息清單

```
GET platform/flagged-messages
```

需要 `message.moderate`（平台管理者與客服都有）。**分頁**。

命中敏感內容偵測的訊息不會消失，會出現在這裡。`data` 形狀同第 5 條的訊息。

> 這是**唯一**不需先加入對話就能看到訊息內容的端點，範圍限於命中偵測的訊息本身。

---

## 8. 隱藏訊息

```
POST platform/messages/{message}/hide
```

需要 `message.moderate`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

回應 200，`data` 為訊息（`status` 為 `hidden`），`message`：`訊息已隱藏，原文保留。`

買賣雙方之後看到的內容會變成 `此訊息已由平台隱藏。`，附件一併隱藏；
**平台視角仍看得到原文**。訊息只能由平台隱藏，買賣雙方都不能刪除訊息。

---

# 個資當事人權利申請（X-04）

## 裁定與執行是兩個端點

去識別化**不可還原**，而「已受理」與「已執行」是兩個不同的事實：
受理之後、執行之前，當事人可能又下了一張訂單 —— 執行端點會**再跑一次**檢查，
那一刻才是最終防線。

三個端點都需要 `audit.view`（合規人員）。

## 9. 申請列表

```
GET platform/data-requests
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `type` | string | `delete`（刪除／去識別化）\| `export`（匯出）\| `correction`（更正） |
| `status` | string | `pending` \| `blocked` \| `approved` \| `rejected` \| `completed` |

### 回應 200

```json
{
  "success": true,
  "message": "取得個資權利申請。",
  "data": [
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
      "pending_days": 4,
      "next_review_at": "2026-08-22T02:40:00+08:00",
      "created_at": "2026-08-17T11:00:00+08:00",
      "member": { "id": 501, "member_number": "000031", "organization_id": 3 }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 2, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `status = blocked` | 目前存在不得受理的情形（進行中訂單、未結爭議等），`blocked_reasons` 列出原因。 |
| `estimated_available_at` | 預估可受理的時間（例如最後一筆訂單走完保存義務之後）。 |
| `download_available` | 僅 `export` 且已完成時為 `true`，表示當事人可自行下載。 |
| `download_expires_at` | 匯出檔的清理時點（`data_export_retention_days`，預設 30 日曆天）。沒有檔案時為 `null`。 |
| `pending_days` | 這一件已經等了幾天。已結案為 `null`。 |
| `next_review_at` | 僅 `blocked`：`privacy:reevaluate-blocked` 下一次重新評估的時點。 |

### 佇列的順序：等最久的先做

spec §X-04 沒有給裁定側的處理期限（違規申訴與未取貨申訴也都沒有），
因此這裡不發明一個 —— 但佇列仍然需要「先做哪一件」的依據。
**未結案的排在前面，其中最舊的先做**；已結案的排在後面（那是查詢用途，不是待辦）。

### 暫緩中的申請會自動重新評估

§X-04.3 的「須先結案才能受理」隱含了「結案之後有人會再看一次」。
`privacy:reevaluate-blocked` 每日執行：阻礙消失時把申請送回 **`pending`**
（不是 `approved` —— 受理仍然是你的一個明確動作），阻礙仍在時把
`blocked_reasons` 的筆數更新成最新的。

---

## 10. 裁定申請

```
POST platform/data-requests/{dataRequest}/decision
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `approved` | boolean | ✅ | `true` 受理、`false` 不受理 |
| `reason` | string | ✅ | 5 ～ 500 字 |

回應 200，`data` 為申請單筆。

| `approved` | `message` |
| --- | --- |
| `true` | `申請已受理，待執行。` |
| `false` | `申請未受理，已記錄理由並通知當事人。` |

---

## 11. 執行申請

```
POST platform/data-requests/{dataRequest}/execute
```

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `confirm` | boolean | **`delete` 類型必填且須為 `true`** | 二次確認 |

```json
{ "confirm": true }
```

> **二次確認做成必要欄位而不是前端的對話框**：對話框攔不住直接打 API 的人，
> 而這個動作沒有第二次機會。前端仍應顯示確認對話框，但那是輔助，不是保證。

### 回應 200

| 類型 | `message` |
| --- | --- |
| `delete` | `已完成去識別化。此操作不可還原；訂單編號、金額與狀態依法定保存義務保留 (spec §X-04.2)。` |
| `export` / `correction` | `已完成，當事人可自行下載。` |

`data` 為申請單筆（`status` 為 `completed`、`executed_at` 有值）。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 422 | 申請不在 `approved` | `只有已受理的申請可以執行。` |
| 422 | `delete` 但未帶 `confirm = true` | `去識別化不可還原，執行前必須二次確認 (spec §X-04.2)。請帶 confirm = true 再送出。` |
| 422 | 受理後又出現不得受理的情形 | `受理後又出現不得受理的情形，已停止執行：…` |

> 最後一種 422 **是正常的防線生效**，不是系統錯誤。請把訊息原樣顯示給操作者，
> 並提示重新確認當事人的訂單狀態。
