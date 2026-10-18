# 01 開店申請

對應 spec S-01、S-10、U-02、§11.2。

## 你建立的是**申請單**，不是店面

`shop_applications` 與 `shops` 是兩張表、兩組狀態。**店面於核准的同一交易內才被建立** ——
在那之前沒有店面可以指向，因此這一章的路由參數一律是 `{application}`。

```
draft ──submit──> pending_review ──┬── approve ────> approved   ← 終態，同時建立店面
   ↑                               ├── needs_info ─> needs_info ──submit──> pending_review
   └───── PATCH 修改 ──────────────┘                     ↑
                                   └── reject ─────> rejected   ← 終態，改不了也送不了
```

三件事值得先記住：

1. **`draft` 與 `needs_info` 才改得動、送得出。** `pending_review` 送審中不得補件 ——
   審核者看到的必須是他點開時的那一份。
2. **`rejected` 是終態。** 不可就地改資料重送；重新申請必為**新的單號**，
   而舊單的退回理由不被覆蓋 —— 那是「上次為什麼沒過」的唯一紀錄（§12.3-71）。
   前端在 `rejected` 上請提供「重新申請」，不是「修改後重送」。
3. **核准不自動開業。** 何時營業是賣家自己的決定，見 [02 §3 開始營業](02-shop.md#3-開始營業)。

## 同時只能有一張進行中的申請

`draft`／`needs_info`／`pending_review` 任一狀態存在時，再建立會得到 **422**（§12.3-73）。
已核准（已經有店面）或已退回的舊單不算進行中。

---

## 1. 我的申請紀錄

```
GET seller/shop-applications
```

**非分頁**，回傳本人的全部申請單（含終態的舊單）。

### 回應 200

```json
{
  "success": true,
  "message": "取得開店申請紀錄。",
  "data": [
    {
      "id": 1,
      "application_no": "SA20260824ddf9d0",
      "organization_id": 2,
      "applicant_member_id": 3,
      "name": "示範待審核申請",
      "slug": "pending-seller",
      "status": "pending_review",
      "created_by_type": "member",
      "is_terminal": false,
      "shop_id": null,
      "submitted_at": "2026-08-24 05:01:12",
      "decided_at": null,
      "decision_reason": null
    }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `is_terminal` | `approved` / `rejected` 為 `true`。前端據此決定按鈕，不要自己列舉狀態。 |
| `created_by_type` | 這張申請單是誰開的：`member`（你自己）\| `platform`（平台代建，§P-01）。代建時你沒有申請過任何東西，卻在賣家中心看到一張以你為店主的申請單 —— 這一欄就是那句「這是平台代你建的」。**擁有者仍是你**，代建只改變誰按了第一個按鈕；不回操作者是誰（平台人員的身分不屬於申請人該看到的資料，§2.3）。 |
| `shop_id` | **核准後才有值**，指向同一交易內建立的店面。 |
| `slug` | 由店名產生，核准後成為前台網址的一部分。**建立後不可變更**。 |

> 清單用的是 `summary` 形狀，**不含** `description` 等內文欄位與 `approval_checklist`。
> 要那些請打第 3 條。

---

## 2. 建立申請草稿

```
POST seller/shop-applications
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | ✅ | 最長 50 |
| `description` | string\|null | — | 最長 2000 |
| `customer_service_contact` | string\|null | — | 最長 100 |
| `pickup_note` | string\|null | — | 最長 1000 |
| `shipping_note` | string\|null | — | 最長 1000 |
| `return_note` | string\|null | — | 最長 1000 |
| `invoice_capability` | string | — | `triplicate` \| `duplicate_only` \| `none` |

```json
{ "name": "職人手作坊", "description": "手工皮件", "invoice_capability": "duplicate_only" }
```

**沒有 `organization_id`**：申請單一律掛在你這組帳號所屬的組織下（§X-05.1）。
在多個工會有會籍的人要換一個工會開店，就用那個工會核發的帳密登入 ——
帳號決定組織，沒有可傳的參數，也沒有切換端點。

**回應 201**，`data` 為申請詳情（含 `approval_checklist`、
`approval_checklist_satisfied` 與 `pending_actions`），
`message`：`開店申請草稿已建立。`

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | **不符開店資格**：會籍非 `active`、商城側被停權、會費未繳、所屬組織不得開店（平台預設組織）、組織本身停權。訊息即具體理由，可直接顯示 |
| 422 | 已有一張進行中的申請 |

> 資格判定在後端，且**與平台代建走同一套** —— 平台代你按按鈕不會讓資格變寬（§S-01）。

---

## 3. 申請詳情

```
GET seller/shop-applications/{application}
```

比清單多三個區塊：`approval_checklist`、`pending_actions`、`review_history`。

> **`review_history` 是新到舊**，最新的一筆在 `[0]`；同秒紀錄按流水號由大到小排序。倒過來讀會拿到最舊的
> 「送審」那一筆，而送審不必填理由 —— 於是 `reason` 是 `null`，
> 看起來像是「理由沒有存進歷程」。那兩件事在畫面上長得一模一樣。

### 回應 200

```json
{
  "success": true,
  "message": "取得開店申請。",
  "data": {
    "id": 1,
    "application_no": "SA20260824ddf9d0",
    "organization_id": 2,
    "applicant_member_id": 3,
    "name": "示範待審核申請",
    "slug": "pending-seller",
    "status": "pending_review",
    "is_terminal": false,
    "submitted_at": "2026-08-24 05:01:12",
    "description": "開發用示範資料。",
    "customer_service_contact": "0912345678",
    "invoice_capability": "none",
    "approval_checklist": [
      { "code": "active_member", "label": "有效會員", "satisfied": true, "detail": null, "actionable_by": "organization" },
      { "code": "membership_paid", "label": "會費已繳", "satisfied": true, "detail": null, "actionable_by": "organization" },
      { "code": "bank_account_verified", "label": "銀行戶名驗證通過", "satisfied": false, "detail": null, "actionable_by": "platform" },
      { "code": "documents_complete", "label": "附件齊全", "satisfied": false, "detail": "尚缺：存摺封面、身分證明文件", "actionable_by": "applicant" }
    ],
    "approval_checklist_satisfied": false,
    "pending_actions": ["審核中，請等候審核結果。"],
    "review_history": []
  },
  "errors": null,
  "meta": null
}
```

> **值為 `null` 的欄位會被省略**，不是回 `null`（上例沒有 `pickup_note`、`decided_at`）。
> 取值一律用 `?? null`，不要假設鍵存在。

### `approval_checklist` —— 四項固定檢核，**三項你自己動不了**

順序固定，可直接照著渲染。每一項都帶 `actionable_by`，說明誰能處理它：

| `code` | 條件 | `actionable_by` | 誰能處理 |
| --- | --- | --- | --- |
| `active_member` | 會籍與商城側狀態**皆為** `active` | `organization` | 工會／平台 |
| `membership_paid` | 會費已繳 | `organization` | 工會／平台（會務端事實） |
| `bank_account_verified` | 撥款帳戶戶名驗證通過 | `platform` | **只有平台**（第 6 條） |
| `documents_complete` | 必要附件齊全，`detail` 列出還缺哪些 | `applicant` | **你自己**（第 5 條） |

**四項全滿足才核准得了**，否則審核者按核准會得到 422。因此畫面上遇到
`bank_account_verified` 為 `false` 時，正確的提示是「等待平台完成戶名驗證」，
不是「請補件」—— 而 `actionable_by` 就是讓你分辨這件事的欄位。

`approval_checklist_satisfied` 是那四項的**合取**，由後端算好。前端做得到自己
`every()` 一次，但那會讓同一條規則有兩個實作，而漂移的症狀是
「按鈕是亮的，按下去 422」。

### `pending_actions` —— 後端算好的下一步

**每一個回傳申請單的端點都帶它**（建立、修改、送審、詳情），不只詳情 ——
建立成功之後畫面該顯示什麼，不必再打一次。

可直接顯示的中文字串陣列，依狀態而定：

| 狀態 | 內容 |
| --- | --- |
| `draft` / `needs_info` | 見下方 |
| `pending_review` | `審核中，請等候審核結果。` |
| `approved` | `審核已通過，可於店面點選「開始營業」正式營運。` |
| `rejected` | `本次審核已結案；如仍希望開店，請重新提出申請。` |

#### `draft` / `needs_info`：分成「輪到你」與「輪到別人」

未滿足的檢核依 `actionable_by` 分組 —— **你自己的先講，你在等別人的後講**：

| 情況 | 內容 |
| --- | --- |
| 四項全滿足 | `資料已齊備，可送出審核。` |
| 附件還缺 | `請先完成「附件齊全」（尚缺：存摺封面）後送出審核。` |
| 只剩戶名驗證 | `「銀行戶名驗證通過」尚未完成，將由平台完成戶名驗證；此項非由您處理，送審前請先等候。` |
| 會籍或會費 | `…請洽所屬工會確認會籍與會費；此項非由您處理…` |

這個區分是必要的：四項裡三項不是你能處理的，而一句籠統的「請先補齊核准條件」
會讓一個什麼都補齊了的人一直找自己還缺什麼。

---

## 4. 修改草稿

```
PATCH seller/shop-applications/{application}
```

欄位與第 2 條相同，全部選填。回應 200，形狀同第 3 條。

**只有 `draft` 與 `needs_info` 改得動**；`pending_review` 與兩個終態回 422。

`slug` 與 `organization_id` 不在可改欄位裡：前者是核准後的前台網址，
後者決定由哪個工會審核與適用哪一份費率。

---

## 5. 附件

```
GET  seller/shop-applications/{application}/documents
POST seller/shop-applications/{application}/documents
```

### 查詢回應 200

```json
{
  "success": true,
  "message": "取得申請附件。",
  "data": {
    "required": ["bank_passbook", "identity"],
    "types": {
      "identity": "身分證明文件",
      "business_registration": "營業登記證明",
      "bank_passbook": "存摺封面",
      "other": "其他補充文件"
    },
    "documents": [],
    "history": []
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `required` | **這個組織要求的附件類型**，由組織類型決定 —— 法人機構要的與自然人不同。清單由後端給，前端不要寫死。 |
| `types` | 全部類型的代碼與中文標籤，供下拉選單。 |
| `documents` | 目前**有效**的附件，同一類型只保留最新一份。 |
| `history` | 換件歷程。舊檔不刪除，因為審核者可能已經看過它。 |

### 上傳

`POST` 為 `multipart/form-data`：

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `document_type` | string | ✅ | 見 `types` 的鍵 |
| `file` | file | ✅ | `pdf` / `jpg` / `jpeg` / `png`，最大 **5120 KB** |

**回應 201**，`data` 為 `{ required, documents }`，`message`：`附件已上傳。`

> **同類型重複上傳是「換掉」，不是「多一份」** —— 新的成為有效附件，舊的移入 `history`。
> 這也表示補件不必先刪除舊檔。

---

## 6. 撥款帳戶

```
GET seller/shop-applications/{application}/bank-account
PUT seller/shop-applications/{application}/bank-account
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `bank_code` | string | ✅ | 三碼數字 |
| `branch_code` | string\|null | — | 1～7 碼數字 |
| `account_name` | string | ✅ | 最長 60 |
| `account_number` | string | ✅ | 5～20 碼，只接受數字與 `-` |

### 回應 200

```json
{
  "success": true,
  "message": "撥款帳戶已更新。",
  "data": {
    "bank_code": "822",
    "branch_code": "0016",
    "account_name": "王小明",
    "account_number_masked": "******0123",
    "verify_status": "pending",
    "verified_at": null,
    "verify_failed_reason": null,
    "payout_ready": false
  },
  "errors": null,
  "meta": null
}
```

尚未設定時 `data` 為 **`null`**（不是空物件）。

| 欄位 | 說明 |
| --- | --- |
| `account_number_masked` | **永遠只有遮罩值**。帳號在資料庫是加密儲存的，**沒有解密端點** —— 連你自己也讀不回完整號碼（§S-10）。 |
| `verify_status` | `pending` \| `verified` \| `failed`。**只有平台改得動它。** |
| `verify_failed_reason` | 驗證失敗的原因，可直接顯示並引導更正。 |
| `payout_ready` | 此本地銀行帳戶是否可作為銀行收款帳戶；PAYUNi grant 另驗原商家／UNi 受款及適用能力，不能以本欄位取代（§14.9.1 條件 5）。 |

> **改帳號會讓驗證失效。** 驗證的是那一組帳號，不是這個人 —— 改完要重新等平台驗證，
> 而未通過驗證的店面不進撥款候選。因此**不要在有待撥款項時換帳號**。

---

## 7. 送出審核

```
POST seller/shop-applications/{application}/submit
```

無請求 body。`draft` / `needs_info` → `pending_review`。
回應 200，`data` 為申請詳情（`pending_actions` 會變成
`審核中，請等候審核結果。`），`message`：`已送出審核。`

### 檢核未滿足**也送得出去**

送審**不檢查核准檢核清單** —— 那道閘門在**核准端**：審核者按核准時若四項未全滿足，
他會得到 422（§U-02）。

因此 `pending_actions` 的「請先補齊核准條件後送出審核。」是**建議，不是限制**。
前端可以把送審按鈕變灰，但那是體貼，不是後端在擋 —— 直接打 API 會成功。

實際送出去的後果是被退回或要求補件，而**退回是終態**：重新申請必為新單號。
所以照著 `pending_actions` 做仍然划算，只是理由是「省一輪」而不是「不然會失敗」。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 狀態不是 `draft` 或 `needs_info`（送審中不得重送；兩個終態送不了） |
| 422 | 所屬組織已停權或關閉 —— 送審是「往新交易靠近一步」（§P-01） |

### 送出之後

球在工會手上（法人機構則在平台，§C-01）。你能做的只有等，或在收到 `needs_info`
之後補件重送。審核結果會出現在 `status` 與 `review_history` 上；被退回時
`decision_reason` 就是原因。

核准的那一刻會同時發生三件事：建立店面、把 `seller` 角色授予你（範圍為該店面）、
`shop_id` 出現在申請單上。接下來走 [02 店面](02-shop.md)。
