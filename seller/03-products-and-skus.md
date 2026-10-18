# 03 商品與規格

對應 spec S-04、P-09.3、§11.2、§11.6、§13.5.2。

## 商品狀態機

```
draft ──submit──> under_review ──┬── approve ──> approved ──list──> listed
  ↑                              ├── needs_info ──> needs_info ──submit──> under_review
  └── PATCH 修改 ────────────────┴── reject ─────> rejected ──submit──> under_review

高風險分類：under_review ──工會 approve──> pending_platform_review ──平台複審──> approved

listed ──delist──> delisted ──list──> listed
listed ──平台違規下架──> violation_delisted   ← 你改不回來
```

九個狀態：`draft`、`under_review`、`needs_info`、`rejected`、`pending_platform_review`、
`approved`、`listed`、`delisted`、`violation_delisted`。

三件容易誤解的事：

1. **`sold_out` 不在上面。** 售完是**由即時可售庫存衍生**的顯示狀態，不是資料庫真值 ——
   它只出現在 `display_status`，而且**絕不可拿來判斷能不能下單**（§11.2）。
2. **高風險分類的「核准」不等於通過。** 工會核准後轉入 `pending_platform_review`，
   尚未完成審核，賣家還不能上架。去向由分類的風險等級決定，不是審核者選的（§P-09.3）。
3. **`violation_delisted` 與 `delisted` 是兩個狀態**，不可互相轉換掩蓋。
   平台違規下架的商品，你按 `list` 也上不回去 —— 那要走[申訴](10-violations.md#3-提出申訴)。

---

## 0. 可選商品分類

```
GET seller/categories
```

只需會員 token。分類是**平台主檔**，每個賣家看到的是同一份，沒有租戶維度。
**非分頁**：`data` 是陣列，`meta` 為 `null`。

建立商品的 `category_id` 是必填，而這一支是它唯一的合法來源——
不必借平台身分，也不必去資料庫撈 ID。

### 回傳的是選項，不是分類樹

只列**現在真的能選的葉節點**，三個條件缺一不可：

* 是子分類（商品必須指定子分類，主分類選了視為未完成，§P-09.1）
* 自己 `status = active` 且非禁售
* **主分類也 `active`** —— 主分類停用代表整條路徑不再開放

回一棵完整的樹會把「哪一層可選」的判斷推給前端，而那個判斷一旦與後端不同，
賣家就會看到一個選了會被拒絕的選項。要看完整兩層樹（含停用與禁售、帶商品數）
是平台維護的事，在 `GET platform/categories`——賣家 token 打那一支回 401。

### 回應 200

```json
{
  "success": true,
  "message": "取得可選商品分類。",
  "data": [
    {
      "id": 12,
      "code": "apparel-tops",
      "name": "上衣",
      "path": "服飾 / 上衣",
      "parent": { "id": 3, "name": "服飾" },
      "requires_platform_review": false
    }
  ],
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `path` | `主分類 / 子分類`。子分類名稱單獨看常常不夠——「配件」可能同時屬於服飾與 3C |
| `requires_platform_review` | 高風險分類（§P-09.3）。**不是不能選**，是工會核准後還要轉平台複審才會 `approved`。賣家該在選之前就知道，而不是等商品卡在一個沒人解釋過的狀態 |

> **清單過濾不是驗證。** 建立與修改商品時後端各自再驗一次分類：偽造的
> `category_id` 從來不會經過這份清單。送審時的檢核表是第三道——草稿建立之後、
> 送審之前，平台可能才把該分類停用。三道守的是三個時點。

---

## 1. 商品列表

```
GET seller/products
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | 上列九個狀態之一 |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

`shop_id` **必填**：授權模型是「角色綁店面」，端點不從 token 推店面（見
[README §兩種閘門](README.md#兩種閘門缺一不可)）。

### 回應 200

```json
{
  "success": true,
  "message": "取得商品列表。",
  "data": [
    {
      "id": 1,
      "shop_id": 1,
      "name": "示範包裝食品（多規格）",
      "status": "listed",
      "display_status": "listed",
      "category_id": 4,
      "condition": "new",
      "is_handmade": false,
      "condition_disclosure": null,
      "description": "開發用示範資料。",
      "images": [],
      "member_only_summary": { "level": "none", "label": null, "matches_member_only_filter": false },
      "listed_at": "2026-08-24 05:01:13",
      "rating_avg": 0,
      "rating_count": 0,
      "delist_reason": null,
      "last_reviewed_at": null,
      "skus": [
        {
          "id": 1,
          "sku_code": "DEMO-FOOD-S",
          "spec_name": "小包裝 250 公克",
          "price": 180,
          "member_price": 160,
          "member_price_scope": "own_organization",
          "member_only": false,
          "cost": 108,
          "weight_grams": 300,
          "volume": null,
          "is_active": true
        }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 12, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `display_status` | 給畫面用的狀態。未上架時等於 `status`；**已上架時依即時可售庫存回傳 `listed` 或 `sold_out`**。 |
| `skus[].cost` | **只有賣家視角看得到成本**。工會與平台的審核畫面沒有這一欄。 |
| `member_only_summary` | `level`：`none` / `partial`（部分規格會員限定）/ `all`。`label` 可直接顯示，`none` 時為 `null`。 |
| `delist_reason` | 下架原因。違規下架時是平台填的。 |
| `last_reviewed_at` | 最後一次審核裁定的時間。**審核歷程**在第 3 條。 |

---

## 2. 建立商品草稿

```
POST seller/products
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 須存在且為你的店 |
| `name` | string | ✅ | 最長 100 |
| `category_id` | integer | ✅ | 須為既有分類 —— **且送審時必須是子分類**（見第 5 條） |
| `condition` | string | ✅ | `new` \| `used` \| `refurbished` |
| `is_handmade` | boolean | — | 預設 `false` |
| `description` | string\|null | — | 最長 5000 |
| `condition_disclosure` | string\|null | — | 最長 2000 |

**回應 201**，`data` 為商品（`status` 為 `draft`），`message`：`商品草稿已建立。`

> **建立商品時不能帶圖。** 圖片是**真實上傳**，走
> [第 12 條 商品圖片](#12-商品圖片)的 `POST seller/products/{product}/images`（multipart）。
> 這裡送 `images` 陣列不會報錯，會被**靜默忽略** —— 然後你得到一件沒有圖的商品。
> 先建草稿拿到 `product.id`，再逐張上傳。

> **`condition`、`is_handmade` 與分類是三個獨立維度**（§11.6）：手工的二手農產品是合法組合。
> 前端不要把它們做成互斥的單選。

---

## 3. 商品詳情與送審檢核

```
GET seller/products/{product}
```

比列表多兩個區塊：

```json
{
  "…": "同列表單筆",
  "submission_checklist": [
    { "code": "child_category", "label": "已指定子分類", "satisfied": true, "detail": null },
    { "code": "category_selectable", "label": "分類可供選用", "satisfied": true, "detail": null },
    { "code": "has_active_sku", "label": "至少一個啟用中的規格", "satisfied": true, "detail": null },
    { "code": "member_price_scope_consistent", "label": "所有規格的會員價資格範圍一致", "satisfied": true, "detail": null },
    { "code": "member_only_has_scope", "label": "會員限定規格具備適用資格範圍", "satisfied": true, "detail": null },
    { "code": "condition_disclosed", "label": "二手與福利品已揭露瑕疵狀況", "satisfied": true, "detail": null }
  ],
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
}
```

### 送審檢核清單 —— 六項

| `code` | 條件 |
| --- | --- |
| `child_category` | 已指定**子分類**（主分類不可直接掛商品） |
| `category_selectable` | 該分類仍可選用（未停用、非禁售） |
| `has_active_sku` | 至少一個 `is_active` 的規格 |
| `member_price_scope_consistent` | 所有規格的 `member_price_scope` 一致 |
| `member_only_has_scope` | 會員限定的規格必須有適用資格範圍 |
| `condition_disclosed` | `used` / `refurbished` 必須填 `condition_disclosure` |

第四、五項是**同一個坑的兩面**：會員價的資格範圍若在同一件商品裡不一致，
買家在商品頁看到的價格會依他點進去的規格而變 —— 而他不會知道為什麼（§13.5.2）。

`review_history` 完整保留工會與平台**兩段**審核，`reviewer_type` 區分是誰做的決定。

---

## 4. 修改商品

```
PATCH seller/products/{product}
```

欄位同第 2 條（不含 `shop_id`），全部選填。回應 200。

**可改的狀態**：`draft`、`needs_info`、`rejected`、`delisted`。
審核中（`under_review` / `pending_platform_review`）與 `listed` 改不動 ——
前者是審核者正在看的那一份，後者要先下架。

---

## 5. 送出審核

```
POST seller/products/{product}/submit
```

無請求 body。回應 200，`message`：`商品已送出審核。`

**可送審的起點**：`draft`、`needs_info`、`rejected`。

> 商品的 `rejected` **與開店申請的 `rejected` 不同** —— 商品退回後改一改可以再送同一件，
> 申請單退回則是終態必須另開一張。兩者不要共用同一套前端邏輯。

檢核未全部滿足時回 422，訊息指出未滿足的項目。

---

## 6. 上架

```
POST seller/products/{product}/list
```

無請求 body。`approved` / `delisted` → `listed`。回應 200，`message`：`商品已上架。`

### 上架不等於買得到

可購買性是**五個條件的交集**（§11.3）：店面已核准且開業、商品已核准且上架、
SKU 可售庫存 > 0、會員限定 SKU 的買家資格符合、配送方式交集非空。
`display_status` 只反映其中的庫存那一項。

`pending_platform_review` 上不了架 —— 那還沒審完。

---

## 7. 下架

```
POST seller/products/{product}/delist
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`message`：`商品已下架。`

**下架不影響既有訂單** —— 已成立的訂單照走完出貨與售後。

---

## 8. 規格列表

```
GET seller/products/{product}/skus
```

`data` 是**規格陣列**（即商品詳情裡的 `skus`），不是包在物件裡。非分頁。

---

## 9. 新增規格

```
POST seller/products/{product}/skus
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `sku_code` | string | ✅ | 最長 40，**同一店面內唯一** |
| `spec_name` | string | ✅ | 最長 60 |
| `price` | integer | ✅ | ≥ 1，整數 TWD（**含稅價**） |
| `member_price` | integer\|null | — | ≥ 1 |
| `member_price_scope` | string | — | `own_organization` \| `any_organization` \| `none` |
| `member_only` | boolean | — | 預設 `false` |
| `cost` | integer\|null | — | ≥ 0，供毛利報表用 |
| `weight_grams` | integer | — | ≥ 0，**按重量計費的運費要用它** |
| `volume` | integer\|null | — | ≥ 0 |
| `sort_order` | integer | — | ≥ 0 |

**回應 201**，`data` 為**該商品的完整規格陣列**（不是剛建立的那一筆），
`message`：`規格已新增。`

> 回傳整組而不是單筆是刻意的：`sort_order` 與會員價一致性都是**跨規格**的性質，
> 只回一筆的話前端得自己合併，而合併錯的那一次會讓排序看起來是隨機的。

| 欄位 | 說明 |
| --- | --- |
| `price` | **含稅價**。稅額由後端反推，不要自己加（§13.4）。 |
| `member_price_scope` | `own_organization` 限本工會會員；`any_organization` 任一工會會員皆可；`none` 無會員價。 |
| `member_only` | 會員限定規格：不符資格者**看得到但買不了**，不是隱藏。 |
| `weight_grams` | 沒填就是 0，而按重量計費的運費會因此算成最低級距 —— 這是最常見的運費短收原因。 |

---

## 10. 修改規格

```
PATCH seller/skus/{sku}
```

欄位同第 9 條，全部選填。回應 200，`data` 同樣是完整規格陣列。

> **改價不影響既有訂單**：商品、規格、價格與庫存在下單當下就以快照寫進訂單（§3.5）。
> 前端不要提示「此變更將影響未出貨訂單」——那是錯的。

---

## 11. 停用規格

```
PATCH seller/skus/{sku}/deactivate
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data` 為完整規格陣列（該筆 `is_active` 為 `false`），
`message`：`規格已停用。`

**只有停用，沒有刪除**：既有訂單、庫存異動與退款都引用 SKU ID，
刪除會讓歷史交易失去指向。

> 停用最後一個啟用中的規格會讓 `has_active_sku` 不再滿足 ——
> 商品仍是 `listed`，但買家買不到任何規格。要整件收掉請用[下架](#7-下架)。

---

## 12. 商品圖片

```
GET seller/products/{product}/images
```

**非分頁**，依 `position` 排序。

```json
{
  "success": true,
  "message": "取得商品圖片。",
  "data": [
    { "id": 7, "position": 0, "is_main": true,  "url": "http://…/storage/product-images/1/xxx.jpg" },
    { "id": 8, "position": 1, "is_main": false, "url": "http://…/storage/product-images/1/yyy.jpg" }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `position` | 排序，**0 即主圖**（§S-04）。永遠是連續的 `0..n-1`。 |
| `is_main` | 由後端算好，不必自己判斷 `position` 是不是 0。 |
| `url` | 可直接顯示的網址。**回應不含儲存路徑** —— 那是實作細節，換磁碟或加 CDN 都不該要求前端改程式。 |

### 圖片不是關鍵欄位

換圖**不會**讓已核准的商品退回重審，也**不影響上架狀態** —— 與售價、庫存同級（§S-04）。
因此已上架的商品也改得動圖片。

> 這是規格上的明示取捨：賣家可以送審一張合規的圖、通過、再換掉它。
> 接受它的前提是圖片改為真實上傳後不會再被外部替換（§13.7），
> 且違規下架與 §P-07 的處置路徑完整。

---

## 13. 上傳圖片

```
POST seller/products/{product}/images
```

**`multipart/form-data`**，不是 JSON。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `file` | file | ✅ | `jpg` / `jpeg` / `png`，最大 `product_image_max_kb`（預設 **5120 KB＝5 MB**） |

**回應 201**，`data` 為該張圖片，`message`：`圖片已上傳。`

* **第一張自動成為主圖**（`position = 0`），其餘接在後面。
* 上限 **10 張**；達上限再上傳回 422。

### 超過大小或格式不符是 422，不是靜默壓縮

§13.7 明訂「拒絕並提示重傳，**不自動壓縮、不靜默截斷**」。
被拒絕的檔案不會留下任何紀錄，因此重傳不需要先刪除什麼。

```json
{
  "success": false,
  "message": "送出的資料未通過驗證。",
  "data": null,
  "errors": { "file": ["The file field must not be greater than 5120 kilobytes."] },
  "meta": null
}
```

---

## 14. 圖片排序

```
PUT seller/products/{product}/images/order
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `image_ids` | integer[] | ✅ | 本商品**全部**圖片的一個排列，不得重複 |

```json
{ "image_ids": [8, 7, 9] }
```

回應 200，`data` 為重排後的完整清單，`message`：`圖片順序已更新，第一張為主圖。`

### 為什麼要完整清單，而不是「把某一張設為主圖」

後者看似簡單，但它沒有表達其餘圖片該落在哪裡 —— 於是實作得自己決定，
而那個決定不會寫在任何地方。清單不完整時回 422。

---

## 15. 刪除圖片

```
DELETE seller/products/{product}/images/{image}
```

回應 200，`message`：`圖片已刪除。`

**刪除後其餘圖片會自動遞補**，`position` 維持 `0..n-1` 連續 ——
因此刪掉主圖之後，原本的第二張成為新的主圖。少了這一步，
商品會處在「有圖片但沒有主圖」的狀態，而主圖是送審的必填欄位（§S-04）。

別家店的圖片 id 回 **404** 而不是 403：那個 id 沒有對外流通的理由，
回 403 等於確認它存在（§2.3）。

**違規下架的商品不得變更圖片**（422）—— 那是平台的處置，
換一張圖再上架等於自行解除處置（§P-07.2）。
