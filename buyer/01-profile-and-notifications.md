# 01 個人資料、會籍與通知

對應 spec X-05、B-01、X-01、X-02、§2.2、§2.3。

---

## 1. 個人資料

```
GET buyer/profile
```

```json
{
  "success": true,
  "message": "取得個人資料。",
  "data": {
    "account": "DEMO-1a258c",
    "phone": "0912345678",
    "email": "demo-buyer@example.test",
    "must_change_password": false,
    "membership": {
      "id": 8,
      "member_number": "1a258cdd",
      "name": "示範買家",
      "identity_card": null,
      "membership_status": "active",
      "marketplace_status": "active",
      "is_tradable": true,
      "membership_paid": true,
      "organization": { "id": 2, "code": "DEMO", "name": "示範職業工會", "type": "union" },
      "can_open_shop": true,
      "eligible_for_member_price": true,
      "phone": "0912345678",
      "email": "demo-buyer@example.test",
      "required_consents_granted": true,
      "shops": [
        { "id": 1, "name": "示範工會商店", "slug": "demo-shop", "status": "open" }
      ]
    }
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `account` | **Phase 1 唯一的登入識別碼**，全站唯一、不可變更，由**所屬組織**核發。電話與 Email 僅作聯絡用途，不能用來登入。 |
| `phone` / `email`（外層） | 這個會籍登記的聯絡方式，與 `membership.phone` / `membership.email` 同一份。 |
| `must_change_password` | 還在用組織方核發的**一次性初始密碼**（§X-05.1）。為 `true` 時請顯示提示並導向[變更密碼](../storefront/01-auth.md#5-變更密碼)。登入回應也帶同一格，但登入只發生一次 —— 使用者關掉那個提示、隔天再進來就沒有任何地方會再提醒他，而那組密碼經線下管道交付，看過它的人不只他一個。 |
| `membership` | 這組帳號所屬的會籍。**不會是 `null`** —— 帳號就在會籍上。 |
| `identity_card` | **一律遮罩**；未設定時為 `null`。買家端**沒有**取得完整身分證號的端點。 |
| `membership.required_consents_granted` | 必要的交易與隱私條款（`transaction_terms`、`privacy_policy`）是否已同意 (§B-01)。`false` 時送出訂單與開店申請會被擋；用第 3 條 `PUT buyer/consents` 同意即可。 |
| `membership.shops` | 這個會籍名下的店面。沒有店面時是**空陣列**，不是 `null`。 |

### `shops`：賣家後台的入口

賣家側每一個列表都要 `?shop_id=`（訂單、退款、庫存、評價、對話、撥款），
而**這是唯一回答得了「我有哪些店」的地方**（登入回應也有同一份）。

店面掛在 **member** 上：一個人可以在多個工會有會籍，而店面屬於其中某一個工會
（§2.2）。以另一個工會的帳號登入，看到的是那個工會的店面清單。

`status` 一起給，因為 `approved` 與 `open` 能做的事不同 ——
「開始營業」是賣家自己的動作（§11.2）。

> **只有本人看得到。** 工會與平台檢視會員時走的是遮罩視角，那裡沒有這一格（§2.3）。

> **本人看自己的聯絡方式不遮罩**，但身分證仍然遮罩（§X-05.4）。

### 會員沒有單一的 `status`

`membership_status`（會籍事實，會務端同步）與 `marketplace_status`（商城處置）
是兩個來源不同的欄位；`is_tradable` 是後端算出的「兩者皆為 `active`」。

**要判斷自己能不能下單，讀 `is_tradable`。** 不能下單時它是 `false`，
而原因分別要循不同管道處理 —— 會籍問題找工會，商城處置循 §P-07.3 申訴。

**這一支沒有修改端點。** 姓名、會員編號與會籍由會務端指派，商城不得改寫（§B-01）；
資料有誤請通報所屬工會。

---

## 2. 一個工會一組帳號

**沒有會籍清單，也沒有切換端點。**

`account` 由所屬組織核發，一個人加入兩個工會就有兩組帳密（§X-05.1）。登入哪一組
就是哪一個工會的身分，而**一個 token 的身分永遠不變**。

這是 2026-08-28 的改動。在那之前作用中會籍記在 `users.active_member_id` 上 ——
一個**帳號層級、跨裝置、持續生效**的狀態。它造成的問題不在資料，在位置：

- 在 A 分頁切換，B 分頁與手機上的下一個請求就換了身分；
- 購物車、訂單、退貨、未取貨紀錄都掛在會籍上，切換後再打舊資料得到的是
  **403**（`您不是這筆交易的當事人`），不是空清單 —— 對使用者而言就是
  「我的訂單不見了」。

改成一組織一帳號之後這條問題從結構上消失，因此前端不需要：

1. 在切換後重新載入所有依賴身分的畫面；
2. 在畫面上常駐顯示「目前作用中的工會」；
3. 防範「另一個分頁偷偷切換了身分」。

要以另一個工會的身分行動，就用那個工會的帳密登入，拿到的是**另一張 token**。
兩張 token 同時有效，互不影響 —— 前端要同時呈現兩個身分也做得到。

> 系統仍然知道這兩組帳號是同一個自然人（平台代建時以身分證比對建立的關聯，
> §X-05.4），但**那條關聯不對外輸出**，買家端沒有任何端點看得到它。

---

## 3. 同意設定

```
GET buyer/consents
PUT buyer/consents
```

### 查詢回應

```json
[
  {
    "consent_type": "terms",
    "scope": null,
    "is_required": true,
    "revocable": false,
    "version": "2026-01",
    "granted": true,
    "action": "granted",
    "acted_at": "2026-08-01T09:00:00+08:00"
  }
]
```

| 欄位 | 說明 |
| --- | --- |
| `is_required` | 必要條款。**未同意即不得使用本服務**（§B-01）。 |
| `revocable` | `is_required` 的反面 —— **必要條款撤不掉**。前端據此決定開關能不能關。 |
| `version` | 條款版本。改版後需要重新取得同意。 |
| `granted` / `action` / `acted_at` | 目前狀態、最後一次動作與時間。 |

### 更新

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `consents` | object | ✅ | `{ "consent_type": boolean }`，至少一項 |
| `scope` | string\|null | — | 最長 60 —— **跨平台串流的同意須記錄用途與欄位範圍**（§X-01） |

回應 200，`data` 為 `[{ consent_type, granted }]`，
`message`：`同意設定已更新，立即生效；既有訂單的履約不受影響 (spec §X-01)。`

**請照實顯示那句話**：撤回同意不會讓已成立的訂單停止履約，
也不會取消你已經買的東西。

---

## 4. 通知設定

```
GET buyer/notification-preferences
PUT buyer/notification-preferences
```

### 查詢回應

```json
{
  "categories": {
    "order": "訂單",
    "payment": "付款",
    "shipment": "出貨與物流",
    "refund": "退貨退款",
    "account_security": "帳號安全",
    "violation": "違規與停權",
    "shop": "店面審核與營業",
    "message": "訊息與評價",
    "marketing": "行銷推播"
  },
  "preferences": [
    { "category": "order", "channel": "in_app", "enabled": true, "mandatory": true },
    { "category": "marketing", "channel": "email", "enabled": true, "mandatory": false }
  ]
}
```

九個分類 × 三個管道（`in_app` / `email` / `sms`）= 27 組設定。

### `mandatory` 決定開關能不能關

| 情況 | `mandatory` | 說明 |
| --- | --- | --- |
| 除 `marketing` 外的八個分類 | `true` | **交易與安全類通知不可關閉**（§X-02） |
| 任何分類的 `in_app` 管道 | `true` | 站內通知一律保留 —— 那是完整內容的唯一去處 |
| `marketing` 的 `email` / `sms` | `false` | 唯一關得掉的兩格 |

前端請用 `mandatory` 把開關變灰，不要讓使用者按了才發現 422。

### 更新

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `preferences` | array | ✅ | 至少一項 |
| `preferences[].category` | string | ✅ | 上列九個之一 |
| `preferences[].channel` | string | ✅ | `in_app` \| `email` \| `sms` |
| `preferences[].enabled` | boolean | ✅ | —— |

回應 200，形狀同查詢。**送出關閉必要通知的組合會被拒絕**（§12.3 的必要通知不可關閉）。

---

## 5. 通知列表

```
GET buyer/notifications
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `unread` | boolean | 只列未讀 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

```json
{
  "id": 83,
  "category": "refund",
  "event_type": "refund.progress",
  "title": "退貨申請已同意",
  "body": "訂單 M2026082400019761-01 的退貨申請已獲賣家同意，請於 2026-08-31 23:59:59 前寄回並登錄物流單號，逾期將自動結案不予退款。退貨運費由賣家負擔。",
  "link": "/buyer/return-requests/5",
  "read_at": null,
  "created_at": "2026-08-24 10:34:47"
}
```

| 欄位 | 說明 |
| --- | --- |
| `category` | 上列九個分類，可用於畫面分群。 |
| `event_type` | 更細的事件代碼，供前端決定圖示或導向邏輯。 |
| `body` | **完整內容只在站內呈現**；Email 與簡訊拿到的是事件提示，不含明細（§X-02）。 |
| `link` | 站內連結，可直接導向。 |

> `body` 常常已經把期限與後果寫進去了（如上例）。前端請直接顯示，
> 不要自己重組句子 —— 那些句子是後端依當下狀態算出來的。

---

## 6. 標記已讀

```
PATCH buyer/notifications/{notification}/read
```

無請求 body。回應 200，`data` 為該筆通知（`read_at` 有值），
`message`：`通知已標記為已讀。`

**沒有「全部已讀」端點**，也沒有刪除通知的端點 —— 通知是事件紀錄。
別人的通知編號回 **404**（不是 403）：那筆通知對你而言不存在（§2.3）。
