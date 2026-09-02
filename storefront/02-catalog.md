# 02 商品與店面

對應 spec F-01、F-03、F-06、§11.2、§11.3、§11.6、§13.2.2、§13.5.2。

---

## 1. 商品搜尋

```
GET storefront/products
```

**分頁**。token 選填 —— 帶了就用你的會籍算價格。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `keyword` | string | 最長 100 |
| `category_id` | integer | 分類 |
| `organization_id` | integer | 限定某個工會的商城 |
| `condition` | string | `new` \| `used` \| `refurbished` |
| `is_handmade` | boolean | 手工製作 |
| `min_price` / `max_price` | integer | 價格區間（整數 TWD） |
| `member_only` | boolean | 只看／排除會員限定商品 |
| `shipping_method` | string | `home` \| `convenience_store` \| `mail` \| `union_pickup` \| `self_pickup` |
| `sort` | string | 見下表，預設 `relevance` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 六種排序

| `sort` | 意義 |
| --- | --- |
| `relevance` | 相關性（預設） |
| `price_asc` / `price_desc` | 價格低→高 / 高→低 |
| `listed_at` | 最新上架 |
| `rating` | 評分 |
| `sales` | 銷量 |

> **價格區間以「售價」為準，不是會員價**：`min_price` / `max_price` 篩的是原價，
> 否則同一件商品會因為看的人不同而落在不同區間，而分頁結果會跟著跳動。

> **`condition` 與 `is_handmade` 是兩個獨立維度**（§11.6）：手工的二手商品是合法組合，
> 前端不要做成互斥單選。

### 回應 200

```json
{
  "success": true,
  "message": "取得商品列表。",
  "data": [
    {
      "id": 1,
      "name": "示範包裝食品（多規格）",
      "images": [],
      "category_id": 2,
      "condition": "new",
      "is_handmade": false,
      "shop": {
        "id": 1,
        "slug": "open-seller",
        "name": "示範開業店面",
        "organization_id": 2,
        "status": "open",
        "is_buyable": true,
        "rating_avg": 0,
        "rating_count": 0,
        "favorite_count": 0
      },
      "display_status": "listed",
      "is_purchasable": true,
      "price_from": 180,
      "member_price_available": true,
      "member_only_summary": { "level": "none", "label": null, "matches_member_only_filter": false },
      "rating_avg": 0,
      "rating_count": 0,
      "favorite_count": 0,
      "listed_at": "2026-08-24 18:37:53"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 6, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `price_from` | **這件商品最低的一個規格價**。帶會員 token 時反映會員價 —— 訪客看 180，DEMO 工會的會員看 160。 |
| `member_price_available` | 這件商品**有沒有**會員價（不代表你適用）。 |
| `is_purchasable` | 此刻買不買得到，涵蓋五個條件（見 [README](README.md#前台看得到的東西是一層一層篩過的)）。 |
| `display_status` | `listed` 或 `sold_out` —— **售完是由即時可售庫存衍生的顯示狀態**，不是資料庫真值。 |
| `member_only_summary` | `level`：`none` / `partial`（部分規格會員限定）/ `all`；`label` 可直接顯示。 |

> **會員限定商品對不符資格者仍會出現在結果裡**，`is_purchasable` 為 `false`。
> 隱藏它們會讓「加入工會有什麼好處」變成看不見的東西。

---

## 2. 商品詳情

```
GET storefront/products/{product}
```

token 選填。**只找得到已上架的商品** —— 草稿、審核中、已下架與違規下架一律
404（§F-01）：回 404 而不是 403，避免洩漏它曾經存在。

> **店面暫停營業不屬於這一類。** 那時商品頁回 **200**，
> `is_purchasable` 為 `false`、SKU 的 `unavailable_reason` 說「店面目前未營業。」
>
> 理由與 §F-03 為店面頁做的取捨相同：買家可能循訂單或收藏連過來，
> 直接 404 會讓他以為店家跑了 —— 而**收藏夾、訂單明細、降價通知裡的連結
> 全都指向商品頁**。兩個入口對同一個事實給兩種答案，前端無從分辨
> 「商品被刪了」與「店家暫停營業」。
>
> 放寬的只有「店面此刻是否可購買」這一項：商品仍須是 `listed`、
> 店面仍須開業過（`opened_at` 有值），而**可購買性的判定一條都沒有動**。

```json
{
  "success": true,
  "message": "取得商品資料。",
  "data": {
    "id": 1,
    "name": "示範包裝食品（多規格）",
    "description": "…",
    "images": [],
    "category_id": 2,
    "condition": "new",
    "is_handmade": false,
    "condition_disclosure": null,
    "display_status": "listed",
    "is_purchasable": true,
    "price_from": 180,
    "member_price_available": true,
    "member_only_summary": { "level": "none", "label": null, "matches_member_only_filter": false },
    "rating_avg": 0,
    "rating_count": 0,
    "favorite_count": 0,
    "listed_at": "2026-08-24 18:37:53",
    "shop": {
      "id": 1,
      "slug": "open-seller",
      "name": "示範開業店面",
      "organization_id": 2,
      "status": "open",
      "is_buyable": true,
      "rating_avg": 0,
      "rating_count": 0,
      "favorite_count": 0,
      "invoice_capability": "none",
      "return_note": null,
      "shipping_note": null,
      "pickup_note": null
    },
    "shipping_methods": ["home", "convenience_store", "mail", "union_pickup", "self_pickup"],
    "skus": [
      {
        "id": 1,
        "sku_code": "DEMO-FOOD-S",
        "spec_name": "小包裝 250 公克",
        "price": 180,
        "price_type": "normal",
        "member_price": null,
        "member_price_available": true,
        "member_only": false,
        "purchasable_quantity": 10,
        "is_purchasable": true,
        "unavailable_reason": null,
        "membership_info_organization_id": null,
        "weight_grams": 500
      }
    ],
    "reviews": []
  },
  "errors": null,
  "meta": null
}
```

### `skus[]` 的價格欄位怎麼讀

| 欄位 | 訪客 | 適用會員價的會員 |
| --- | --- | --- |
| `price` | 原價 180 | **160**（已經是成交價） |
| `price_type` | `normal` | `member` |
| `member_price` | `null` | **160**（與 `price` 相同） |

**`price` 永遠是「你會付的那個數字」**，不需要自己在 `price` 與 `member_price`
之間挑。`price_type` 是給畫面標「會員價」用的。

`member_price` 只是把「這個數字是會員價」再說一次：符合資格時它等於 `price`，
不符資格時是 `null`。**它不是「原價旁邊那個比較小的數字」** —— 要顯示
「原價 180 → 會員價 160」的劃線效果，原價得由前端自己保留（例如先以訪客身分
取一次，或由商品列表的 `price_from` 對照），API 不在同一筆回應裡同時給兩個價格。

| 欄位 | 說明 |
| --- | --- |
| `purchasable_quantity` | 此刻最多買得到幾件 —— **已經扣掉別人的預留**，不是庫存數。 |
| `unavailable_reason` | 買不到的具體原因（會員限定、售完等），可直接顯示。 |
| `membership_info_organization_id` | 「加入哪個工會就能享有這個會員價」的提示 —— 給不符資格者看的。 |
| `weight_grams` | 供前端估算按重量計費的運費。 |

### `shipping_methods` 是**交集**的結果

平台 → 工會 → 店面 → 商品限制，取交集（§13.2.2）。這裡列出來的就是結帳時
**選得到**的方式；交集為空時該商品不得結帳（§12.3-62），
`is_purchasable` 會是 `false`。

### 店面的三段說明一起給

`shipping_note` / `pickup_note` / `return_note` 隨商品頁一起回傳 ——
買家在下單前就該看得到出貨時間、取貨規則與退貨條件，
而 `invoice_capability` 讓他知道能不能報帳（§13.6.2）。

---

## 3. 店面頁

```
GET storefront/shops/{slug}
```

`{slug}` 是店面代稱（由店名產生，建立後不變）。**不需 token。**

```json
{
  "success": true,
  "message": "取得店面資料。",
  "data": {
    "id": 1,
    "slug": "open-seller",
    "name": "示範開業店面",
    "organization_id": 2,
    "status": "open",
    "is_buyable": true,
    "rating_avg": 0,
    "rating_count": 0,
    "favorite_count": 0,
    "description": "開發用示範資料。",
    "customer_service_contact": "0912345678",
    "invoice_capability": "none",
    "logo_url": null,
    "cover_url": null,
    "service_hours": null,
    "shipping_note": null,
    "pickup_note": null,
    "return_note": null,
    "opened_at": "2026-08-24 18:37:52",
    "announcements": [
      {
        "id": 3,
        "title": "連假出貨說明",
        "content": "連假期間物流暫停收件，訂單將於收假後依序出貨。",
        "is_pinned": true,
        "published_from": "2026-08-24 00:00:00",
        "published_until": null
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

### 識別圖像與客服時間

| 欄位 | 說明 |
| --- | --- |
| `logo_url` / `cover_url` | 未設定為 `null` —— **請以預設樣式呈現，不要留空白區塊**（§F-03）。 |
| `service_hours` | 客服可回應的時間，自由文字。**純顯示** —— 在其所載時間之外仍可正常下單、結帳與付款（§S-03.2）。 |

> 這三個欄位**不經審核**，賣家改完即時生效（§S-03）。

### `announcements` —— 只有此刻生效中的

置頂者在前。**店面非 `open` 時一律是空陣列**，但**店面頁本身仍看得到**：
買家可能循訂單或收藏連過來，直接 404 會讓他以為店家跑了（§F-03）。

前台的公告**不帶 `status`** —— 買家不需要知道它是草稿還是已下架。

### 從未開業過的店面回 404

`approved` 但還沒點「開始營業」的店面，對前台而言不存在 —— 回 404 而不是一頁空的店面。

**已暫停或已關閉的店面仍然看得到**（`opened_at` 有值），只是 `is_buyable` 為 `false`：
買家可能是循訂單或收藏連過來的，直接 404 會讓他以為店家跑了。

### `unavailable_reason` —— 買不到的原因，可直接顯示

`is_buyable: false` 有兩種來源，而它們對訪客的意義完全不同：
暫停營業是「過幾天再來」，已關閉是「別等了」。一個布林值分不出這兩件事，
畫面只能寫一句含糊的「暫不可購買」。

| 店面狀態 | `unavailable_reason` |
| --- | --- |
| `open` | `null` |
| `suspended` | `此店面暫停營業中，稍後再回來看看。` |
| `closed` | `此店面已結束營業。` |

與 SKU 的 `unavailable_reason` 同一個位置、同一個理由：判斷在後端，句子也在後端。

> **不揭露處置事由**：停權的原因是店家與平台之間的事（§2.3）。
> 對訪客而言「暫停營業中」就是他需要知道的全部。

---

## 4. 店面商品列表

```
GET storefront/shops/{slug}/products
```

**分頁**。token 選填。參數同第 1 條（`sort` 與分頁；篩選條件在這裡意義不大，
因為範圍已經限定在這家店）。

`data` 為商品陣列，形狀同第 1 條的單筆。

### 未營業的店面回**空清單**，不是 404

```json
{
  "success": true,
  "message": "此店面目前未營業。",
  "data": [],
  "errors": null,
  "meta": {
    "pagination": { "current_page": 1, "per_page": 20, "total": 0, "last_page": 1 },
    "empty_reason": "此店面暫停營業中，稍後再回來看看。"
  }
}
```

店面頁看得到（第 3 條），商品清單是空的，並附一句可直接顯示的說明 ——
這比 404 誠實：店還在，只是現在不營業。

> 從未開業過的 slug 仍然是 **404**：兩者的差別是「暫時不做生意」與「不存在」。

### `meta.empty_reason` —— 空清單是哪一種空

一個空陣列有三種意思，而三種對訪客的意義完全不同：

| 情形 | `empty_reason` |
| --- | --- |
| 店面未營業 | `此店面暫停營業中，稍後再回來看看。` |
| 開業但還沒上架任何商品 | `這家店還沒有上架任何商品。` |
| 有商品 | **這一格不出現** |

放在 `meta` 而不是 `data`：`data` 只放領域資料（見[共通約定](../conventions.md)）——
塞進去會讓「一個空陣列」變成「一個帶著字串的陣列」，而前端的迴圈會踩到它。
