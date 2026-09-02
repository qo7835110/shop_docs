# 02 審核（店面申請、商品）

對應 spec U-02、U-03、P-09.3、C-01、§15.3、§2.3。

工會分潤的對價，正是工會對店面與商品的審核與管理責任（§2.2.1）。這兩組端點是工會後台的核心。

## 兩條貫穿本頁的規則

1. **申請人與審核人不得為同一帳號**（§15.3）。工會管理者自己開的店，要由工會的另一個人審，
   或由平台審。這道檢查在服務層，CLI 與其他入口一樣受限。
2. **核准高風險分類的商品不等於「已通過」**。它只會轉入平台複審，尚未完成審核 ——
   工會端若顯示成「已通過」，賣家會誤以為可以上架。而且**送上去之後就收不回來**：
   對 `pending_platform_review` 的商品再次裁定會被擋成 403，那一段只有平台做得了主。

---

# 店面申請審核（U-02）

## 審核的對象是**申請單**，不是店面（§11.2）

`shop_applications` 與 `shops` 是兩張表、兩組狀態。**店面於核准的同一交易內才被建立**
—— 核准之前沒有店面可以指向，因此路由參數是 `{application}`。

```
申請單 shop_applications
  draft ──賣家送審──> pending_review ──┬── approve ────> approved   ← 終態
                                       ├── needs_info ─> needs_info ──賣家補件重送──> pending_review
                                       └── reject ─────> rejected   ← 終態

                                          approve 的同一交易內建立 ↓
店面 shops
  approved ──(賣家自行開業)──> open
```

工會能做的只有中間那一步（`decision`）。**核准之後不會自動開業** ——
何時營業是賣家自己的決定（`approved → open` 沒有工會端的端點）。

法人機構（`corporation`）的申請由平台審核，工會端對它呼叫 `decision` 會得到 403。

---

## 1. 店面申請清單

```
GET union/shop-applications
```

需要 `shop.review` @ 本工會。**分頁**。範圍固定為操作者自己的工會。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `draft`／`pending_review`／`needs_info`／`approved`／`rejected` —— **申請單**的狀態 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

> 省略 `status` 會列出本工會**所有狀態**的申請單，不只待審的。
> 做待辦清單請明確帶 `status=pending_review`。
>
> 要查已開業店面的營業狀態（`open`／`suspended`／`closed`）請用
> [`GET union/shops`](04-shop-and-member-management.md#1-旗下店面清單) —— 那是另一張表。

### 回應 200

```json
{
  "success": true,
  "message": "取得本工會的店面申請。",
  "data": [
    {
      "id": 21,
      "application_no": "SA20260815000007",
      "organization_id": 3,
      "applicant_member_id": 501,
      "name": "職人手作坊",
      "slug": "zhi-ren-shou-zuo-fang",
      "status": "pending_review",
      "is_terminal": false,
      "submitted_at": "2026-08-15 09:10:00",
      "description": "手工皮件",
      "customer_service_contact": "service@example.com",
      "pickup_note": "週三公休",
      "shipping_note": "下單後 3 個工作天內出貨",
      "return_note": "商品到貨後 7 日內可退",
      "invoice_capability": "duplicate_only",
      "review_history": [],
      "approval_checklist": [
        { "code": "active_member", "label": "有效會員", "satisfied": true, "detail": null, "actionable_by": "organization" },
        { "code": "membership_paid", "label": "會費已繳", "satisfied": true, "detail": null, "actionable_by": "organization" },
        { "code": "bank_account_verified", "label": "銀行戶名驗證通過", "satisfied": false, "detail": null, "actionable_by": "platform" },
        { "code": "documents_complete", "label": "附件齊全", "satisfied": false, "detail": "尚缺：存摺封面", "actionable_by": "applicant" }
      ],
      "approval_checklist_satisfied": false
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 2, "last_page": 1 } }
}
```

### 核准檢核清單

`approval_checklist` 是**四項固定檢核**，順序固定，直接照著渲染即可：

| `code` | 條件 | `actionable_by` | 誰能處理 |
| --- | --- | --- | --- |
| `active_member` | 店主會籍狀態為 `active` | `organization` | 工會／平台 |
| `membership_paid` | 店主會費已繳 | `organization` | 工會／平台 |
| `bank_account_verified` | 撥款帳戶戶名驗證通過 | `platform` | **平台**（工會不能驗證） |
| `documents_complete` | 必要附件齊全，`detail` 列出還缺哪些 | `applicant` | 賣家補件 |

**四項全部 `satisfied: true` 才核准得了**，否則 `decision` 會回 422。
其中兩項工會自己動不了 —— 遇到 `bank_account_verified` 為 `false` 時，
畫面應該引導「請平台完成戶名驗證」，而不是讓審核者按了才發現失敗。

### `approval_checklist_satisfied` —— 這張現在核准得了嗎

那四項的**合取**，由後端算好。清單上有四項明細，因此前端做得到自己
`every()` 一次 —— 但核准端的規則就是這個合取，讓它有兩個實作遲早會漂移，
而漂移的症狀是「按鈕是亮的，按下去 422」。

審核佇列一次看二十張，逐張自己算等於逐張自己記規則。**直接讀這一格。**

### 這是**申請單**的欄位，不是店面的

清單回的是 `shop_applications` 的列，**沒有** `is_buyable`、`favorite_count`、
`rating_avg`、`opened_at`、`suspended_reason` 這些店面欄位 —— 那些要到核准
建立店面之後才存在（[04 §1](04-shop-and-member-management.md#1-旗下店面清單)）。

| 欄位 | 說明 |
| --- | --- |
| `application_no` | 申請單號。**退回後重新申請必為新單號**，舊單的理由不被覆蓋。 |
| `applicant_member_id` | 申請人（未來的店主）。**他不得同時是審核者**（§15.3）。 |
| `is_terminal` | `approved` / `rejected` 為 `true` —— 終態的單裁定不了，前端直接把按鈕收掉。 |
| `shop_id` | **核准後才有值**，指向同一交易內建立的店面。未核准時這個鍵不出現。 |
| `submitted_at` / `decided_at` / `decision_reason` | 送審、裁定的時間與理由。**值為 `null` 的鍵會被省略**，取值請用 `?? null`。 |
| `review_history` | 完整審核歷程，草稿階段為空陣列。 |

---

## 2. 店面審核決議

```
POST union/shop-applications/{application}/decision
```

需要 `shop.review` @ 該店面所屬工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `decision` | string | ✅ | `approve` \| `needs_info` \| `reject` |
| `reason` | string\|null | **`approve` 以外必填** | 最長 500 |

```json
{ "decision": "needs_info", "reason": "請補上存摺封面影本" }
```

### 回應 200

```json
{
  "success": true,
  "message": "審核結果已記錄，並已通知申請人。",
  "data": {
    "…": "同清單單筆，**含 approval_checklist**",
    "status": "needs_info",
    "decided_at": "2026-08-18 14:30:00",
    "decision_reason": "請補上存摺封面影本",
    "review_history": [
      {
        "decision": "needs_info",
        "from_status": "pending_review",
        "to_status": "needs_info",
        "reason": "請補上存摺封面影本",
        "reviewer_type": "member",
        "reviewed_at": "2026-08-18 14:30:00"
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

`review_history` 是完整的審核歷程，`reviewer_type` 區分 `member`（工會管理者）
與 `admin`（平台人員）。

回應與清單單筆是**同一個形狀**（`ShopApplicationPresenter::detailed`），
一樣帶 `approval_checklist` —— 裁定完最想知道的下一件事就是「現在還差什麼」。
`approve` 時 `shop_id` 會有值：店面在核准的同一交易內建立。

### 三種決議的差別

| decision | 申請單結果狀態 | 賣家後續 |
| --- | --- | --- |
| `approve` | `approved`（**終態**） | 同一交易內建立店面；賣家自行開業才會變 `open` |
| `needs_info` | `needs_info` | 補件後可重新送審**同一張單** |
| `reject` | `rejected`（**終態**） | **不可就地改資料重送**；重新申請必為新單號 |

> **`approve` 與 `reject` 都是終態，`needs_info` 不是** —— 兩者的差別不是語氣，
> 是能不能回頭（§12.3-71）。
>
> `reject` 表示「這個申請本身不該通過」，此後這張單就定案了：賣家要再開店必須
> 建立**新的申請單**，而舊單的退回理由**不被覆蓋** —— 那是「上次為什麼沒過」的
> 唯一紀錄，也是下一輪審核時的依據。
>
> `needs_info` 表示「資料不齊」，同一張單補完再送。前端在 `rejected` 的單上
> 不要提供「修改後重送」，應提供「重新申請」。

### 失敗

| 狀態 | 情況 | message 重點 |
| --- | --- | --- |
| 403 | 跨工會審核 | `您只能審核本工會的店面申請，不得跨工會存取 (spec §2.3)。` |
| 403 | 沒有 `shop.review` | `您沒有審核店面的授權 (spec §U-02)。` |
| 403 | 對法人機構的申請 | `法人機構的開店申請由平台審核 (spec §C-01)。` |
| 422 | **店主就是審核者**，或**送審者就是審核者**（平台代送審的情況） | `店面審核：同一帳號不得同時擔任兩方 (spec §15.3)。` |
| 422 | 申請單不在 `pending_review` | `只有待審核的申請可以裁定 (spec §11.2)。`（`needs_info` 要等賣家重新送審；`approved`／`rejected` 是終態，永遠裁定不了） |
| 422 | 會務同步逾期 | 核准的檢核含「有效會員」「已繳費」，同步不可信時不得核准（§U-05.3）—— 是 422 不是 403 |
| 422 | 核准時檢核清單未全部滿足 | 指出未滿足的項目 |

---

# 商品審核（U-03）

## 商品狀態機（§11.2、§P-09.3）

```
draft ──賣家送審──> under_review ──┬── approve ──> approved ──(賣家上架)──> listed
                                   ├── needs_info ──> needs_info
                                   └── reject ─────> rejected

高風險分類：under_review ──工會 approve──> pending_platform_review ──平台複審──> approved
```

**同一個 `approve` 動作，結果可能是兩種狀態**，取決於商品分類的風險等級。

---

## 3. 商品審核佇列

```
GET union/product-reviews
```

需要 `product.review` @ 本工會。**分頁**。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | 只接受審核中的三種狀態：`under_review`、`needs_info`、`pending_platform_review` |
| `page` / `per_page` | integer | 見共通約定 |

**省略 `status` 時三種狀態都會列出**，包含已送平台複審的商品 —— 它們留在清單裡是為了讓工會
追蹤下落，但**球已經在平台那邊，不是工會的待辦**。做待辦清單請明確帶 `status=under_review`。

> 這也是為什麼[總覽](01-overview-and-queries.md#1-工會商城總覽)的
> `pending_reviews.product_reviews` 會小於本清單的 `total`：那個數字只算 `under_review`。

> **已送複審的商品，工會裁定不了**：對它呼叫 `decision` 會得到 403
> `商品已轉入平台複審，改由平台裁定 (spec §P-09.3)。`
> 前端請只對 `under_review` / `needs_info` 的項目顯示審核按鈕。

### 回應 200

```json
{
  "success": true,
  "message": "取得本工會的商品審核佇列。",
  "data": [
    {
      "id": 3301,
      "shop_id": 21,
      "name": "手工皮革刀具組",
      "status": "under_review",
      "display_status": "under_review",
      "category_id": 88,
      "condition": "new",
      "is_handmade": true,
      "condition_disclosure": null,
      "description": "……",
      "images": ["https://cdn.example.com/p/3301-1.jpg"],
      "member_only_summary": { "level": "none", "label": null, "matches_member_only_filter": false },
      "listed_at": null,
      "rating_avg": 0,
      "rating_count": 0,
      "shop": { "id": 21, "name": "職人手作坊", "organization_id": 3 },
      "skus": [
        {
          "id": 9001,
          "sku_code": "PL-001-M",
          "spec_name": "中型",
          "price": 1280,
          "member_price": 1180,
          "member_price_scope": "own_organization",
          "member_only": false,
          "weight_grams": 800,
          "volume": null,
          "is_active": true
        }
      ],
      "last_reviewed_at": null,
      "requires_platform_review": false
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 5, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `skus[].cost` | **不存在**。審核者看得到售價與會員價，看不到成本 —— 那只在賣家自己的視角出現。 |
| `member_price_scope` | `own_organization`（限本工會會員）\| `any_organization`（任一工會會員）\| `none` |
| `member_only_summary.level` | `none` \| `partial`（部分規格會員限定）\| `all`。`label` 可直接顯示。 |
| `condition` / `condition_disclosure` | 商品新舊狀況與揭露說明，審核重點之一。 |
| `requires_platform_review` | **按下核准之後這件商品會去哪裡**（§P-09.3）。`true` 代表分類為高風險，工會核准後轉入 `pending_platform_review` 而不是 `approved` —— 兩種都是成功，但畫面上要說的話不同，而它也是要不要多看兩眼的判準。 |

> `requires_platform_review` 由後端依分類的 `risk_level` 算出。
> 分類是平台改得動的資料 —— 前端寫死一份高風險清單，改了就錯。

---

## 4. 商品審核決議

```
POST union/product-reviews/{product}/decision
```

需要 `product.review` @ 該商品所屬工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `decision` | string | ✅ | `approve` \| `needs_info` \| `reject` |
| `reason` | string\|null | **`approve` 以外必填** | 最長 500 |

### 回應 200 —— 兩種 message，一定要照實顯示

`data` 為商品形狀（同佇列單筆）再加上 `review_history`。**`message` 依結果不同**：

| 核准後的 `status` | `message` |
| --- | --- |
| `approved` | `審核結果已記錄，並已通知賣家。` |
| `pending_platform_review` | `工會審核已通過，因分類為高風險，商品已送平台複審，尚未完成審核。` |

```json
{
  "success": true,
  "message": "工會審核已通過，因分類為高風險，商品已送平台複審，尚未完成審核。",
  "data": {
    "id": 3301,
    "status": "pending_platform_review",
    "…": "…",
    "review_history": [
      {
        "decision": "approve",
        "from_status": "under_review",
        "to_status": "pending_platform_review",
        "reason": null,
        "reviewer_type": "member",
        "reviewed_at": "2026-08-18 11:03:00"
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

> **前端請直接顯示回應的 `message`，不要自己組「審核通過」四個字。**
> 判斷式若要自己寫，依據是 `data.status`：`pending_platform_review` 代表**尚未完成審核**，
> 賣家還不能上架。這是 §U-03 明列的規範重點。

### 失敗

| 狀態 | 情況 | message 重點 |
| --- | --- | --- |
| 403 | 跨工會審核 | `您只能審核本工會的商品，不得跨工會存取 (spec §2.3)。` |
| 403 | 沒有 `product.review` | `您沒有審核商品的授權 (spec §U-03)。` |
| 403 | **商品已在 `pending_platform_review`** | `商品已轉入平台複審，改由平台裁定 (spec §P-09.3)。` |
| 422 | 商品不在審核中狀態 | `只有審核中的商品可以裁定 (spec §11.2)。` |
| 422 | **店主就是審核者** | `商品審核：同一帳號不得同時擔任兩方 (spec §15.3)。` |
