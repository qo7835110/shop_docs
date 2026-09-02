# 05 商品複審與分類

對應 spec P-09、P-09.1、P-09.3、§11.2。

## 商品狀態機（§11.2）

```
draft ──submit──> under_review ──┬── approve ──> approved ──(賣家上架)──> listed
                                 ├── needs_info ──> needs_info
                                 └── reject ─────> rejected

高風險分類：under_review ──工會核准──> pending_platform_review ──平台複審──> approved

listed ──賣家下架──> delisted
listed ──平台違規下架──> violation_delisted
listed ──(庫存歸零)──> sold_out（衍生狀態）
```

**平台只處理兩件事**：`pending_platform_review` 的複審，以及違規下架。
一般風險的商品由工會核准即完成審核，平台不必也不該逐件過目。

> `sold_out` 是**由即時可售庫存衍生**的顯示狀態，不是資料庫真值，
> 也絕不可拿來判斷「能不能下單」。

---

## 1. 平台複審佇列

```
GET platform/product-reviews
```

需要 `product.review`。**分頁**。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | 只接受審核中的狀態：`under_review`、`needs_info`、`pending_platform_review`。**省略時預設只列 `pending_platform_review`** |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得平台複審佇列。",
  "data": [
    {
      "id": 3301,
      "shop_id": 21,
      "name": "手工皮革刀具組",
      "status": "pending_platform_review",
      "display_status": "pending_platform_review",
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
      "last_reviewed_at": "2026-08-16 11:03:00"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 2, "last_page": 1 } }
}
```

> 審核者看得到 SKU 的售價與會員價，**看不到成本**（`cost` 只在賣家自己的視角出現）。

| 欄位 | 說明 |
| --- | --- |
| `display_status` | 給畫面用的狀態代碼。未上架時等於 `status`；已上架時依即時可售庫存回傳 `listed` 或 `sold_out`。 |
| `member_only_summary.level` | `none`（無會員限定 SKU）\| `partial`（部分規格）\| `all`（全部）。`label` 是可直接顯示的中文標籤，`none` 時為 `null`。 |
| `member_price_scope` | 會員價適用範圍，`own_organization`（限本工會會員）、`any_organization`（任一工會會員）、`none`（無會員價）。 |

---

## 2. 複審決議

```
POST platform/product-reviews/{product}/decision
```

需要 `product.review`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `decision` | string | ✅ | `approve` \| `needs_info` \| `reject` |
| `reason` | string\|null | `approve` 以外必填 | 最長 500 |

```json
{ "decision": "reject", "reason": "刀具屬管制品項，未提供合格證明" }
```

### 回應 200

`data` 為商品形狀（同佇列單筆）再加上 `review_history`：

```json
{
  "review_history": [
    {
      "decision": "approve",
      "from_status": "under_review",
      "to_status": "pending_platform_review",
      "reason": null,
      "reviewer_type": "member",
      "reviewed_at": "2026-08-16 11:03:00"
    },
    {
      "decision": "reject",
      "from_status": "pending_platform_review",
      "to_status": "rejected",
      "reason": "刀具屬管制品項，未提供合格證明",
      "reviewer_type": "admin",
      "reviewed_at": "2026-08-18 10:22:00"
    }
  ]
}
```

`message`：`複審結果已記錄，並已通知賣家。`

`review_history` 完整保留工會與平台兩段審核，`reviewer_type` 區分是誰做的決定。

---

## 3. 違規下架

```
POST platform/products/{product}/violation-delist
```

需要 **`violation.manage`**（不是 `product.review`）—— 這是處置，不是審核。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data` 為商品形狀（`status` 為 `violation_delisted`），
`message`：`商品已違規下架，並已通知賣家；賣家得依申訴程序提出異議。`

> 違規下架與賣家自行下架（`delisted`）是**不同狀態**，不可互相轉換掩蓋。
> 賣家不能自己把 `violation_delisted` 改回上架。

---

# 商品分類主檔

分類是平台主檔（§P-09.1），**只停用、不刪除**：既有商品都引用分類 ID。

## 4. 分類樹

```
GET platform/categories
```

需要 `category.manage`。**非分頁**，回傳完整兩層樹狀結構。

### 回應 200

```json
{
  "success": true,
  "message": "取得商品分類樹。",
  "data": [
    {
      "id": 10,
      "parent_id": null,
      "code": "handmade",
      "name": "手工藝品",
      "risk_level": "normal",
      "is_prohibited": false,
      "status": "active",
      "sort_order": 1,
      "requires_platform_review": false,
      "selectable_by_products": true,
      "product_count": 0,
      "children": [
        {
          "id": 88,
          "parent_id": 10,
          "code": "handmade_tools",
          "name": "手作工具",
          "risk_level": "high",
          "is_prohibited": false,
          "status": "active",
          "sort_order": 2,
          "requires_platform_review": true,
          "selectable_by_products": true,
          "product_count": 37
        }
      ]
    }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `risk_level` | `normal` \| `high`。`high` 會讓該分類的商品在工會核准後轉入 `pending_platform_review`。 |
| `is_prohibited` | 禁售分類。商品不得選用。 |
| `status` | `active` \| `inactive`。停用後不可再被新商品選用。 |
| `requires_platform_review` | **後端算出的衍生值**，前端不要自行以 `risk_level` 推導。 |
| `selectable_by_products` | 商品建立／編輯時是否可選用此分類（同時考慮 `status` 與 `is_prohibited`）。 |
| `product_count` | 掛在這個分類底下的商品數（含各種狀態）。**停用前唯一要問的問題是「這會影響幾件」**——停用是「新商品不能再選」，既有商品仍掛在上面。商品必須指定子分類，因此頂層分類通常是 0。 |

> 子分類只有一層，`children` 內的節點不再有 `children` 欄位。

---

## 5. 建立分類

```
POST platform/categories
```

需要 `category.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `parent_id` | integer\|null | — | 須為既有分類；`null` 為頂層 |
| `code` | string | ✅ | 最長 40、`^[a-z][a-z0-9_]*$`、**全站唯一** |
| `name` | string | ✅ | 最長 50 |
| `risk_level` | string | — | `normal`（預設）\| `high` |
| `is_prohibited` | boolean | — | 預設 `false` |
| `sort_order` | integer | — | ≥ 0，預設 0 |

**回應 201**，`data` 為單一分類（**不含** `children`），`status` 預設 `active`。

---

## 6. 更新分類

```
PATCH platform/categories/{category}
```

需要 `category.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | — | 最長 50 |
| `risk_level` | string | — | `normal` \| `high` |
| `is_prohibited` | boolean | — | — |
| `sort_order` | integer | — | ≥ 0 |
| `reason` | string\|null | — | 最長 255，寫入稽核 |

`code` 與 `parent_id` **不可變更**。回應 200，`data` 為單一分類，
**外加 `affected_product_count`**：這次調整把幾件商品退回了平台複審。

```json
{ "…": "分類欄位", "affected_product_count": 12 }
```

調升風險等級是一個**有連帶效果**的動作，而按下去的人原本只拿到分類本身 ——
「有 12 件商品被退回複審」得自己去複審佇列數，而佇列裡還混著別的分類的商品。
對照組是違規記錄：那支端點的回應直接帶回重新判定後的處置。

沒有連帶效果時為 `0`（例如調降風險、只改名稱）。**`0` 與「沒有這個欄位」
對操作者是兩件事**，因此它恆在。

> **把 `risk_level` 從 `normal` 調升為 `high` 有連帶效果**：該分類下**已核准但尚未上架**的商品
> 會被退回 `pending_platform_review`，出現在複審佇列裡。**已上架的商品不會自動退回** ——
> 那要由平台依 §P-09.3 決定是否違規下架。調降風險則沒有連帶效果。

---

## 7. 啟用／停用分類

```
PATCH platform/categories/{category}/status
```

需要 `category.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `active` \| `inactive` |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data` 為單一分類。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 停用父分類但其子分類仍為 `active` —— 必須由下往上停用 |

> **停用不影響既有商品**：已經在該分類下的商品不會自動下架，只是新商品不能再選。
> 需要下架請走違規下架或通知賣家自行處理。
