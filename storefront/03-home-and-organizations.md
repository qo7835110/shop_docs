# 03 首頁與工會商城

對應 spec F-02、U-04、§2.2.1、§12.3-18、§12.3-51。

---

## 1. 首頁

```
GET storefront/home
```

token 選填。**非分頁**。

```json
{
  "success": true,
  "message": "取得首頁內容。",
  "data": {
    "featured_products": [ { "…": "商品摘要，形狀同商品搜尋的單筆" } ],
    "organizations": [
      { "id": 3, "code": "DEMO2", "name": "示範第二職業工會", "type": "union" },
      { "id": 2, "code": "DEMO", "name": "示範職業工會", "type": "union" }
    ]
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 上限 | 說明 |
| --- | :---: | --- |
| `featured_products` | 12 | 精選商品，**已經過完整的可購買性判定** |
| `organizations` | 12 | 工會商城入口 |

### 精選商品的筆數可能少於 12

候選是先多取一些，再逐件跑可購買性（可售庫存、會員限定資格）——
那兩項在 SQL 之外判定。因此 **12 是上限不是保證**，而且**帶會員 token 時筆數可能不同**：
會員限定商品對符合資格的人才算可購買。

### `organizations` 排除平台預設組織

平台預設組織不承載店面、沒有工會管理者、也沒有商城頁（§2.2.1）——
它出現在入口清單只會導向一個永遠是空的頁面（§12.3-51）。

這裡的單筆只有四個欄位（`id`／`code`／`name`／`type`），要看內容請打第 3 條。

---

## 2. 工會商城列表

```
GET storefront/organizations
```

**不需 token。** 非分頁（受 `per_page` 限制筆數）。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `keyword` | string | 最長 50，比對代碼與名稱 |
| `per_page` | integer | 筆數上限，見[共通約定](../conventions.md#分頁) |

`data` 形狀同首頁的 `organizations`。

同樣**排除平台預設組織**，也排除已關閉的組織。

---

## 3. 工會商城頁

```
GET storefront/organizations/{code}
```

`{code}` 是**組織代碼**（如 `DEMO`），不是數字 ID。token 選填。

```json
{
  "success": true,
  "message": "取得工會商城內容。",
  "data": {
    "id": 2,
    "code": "DEMO",
    "name": "示範職業工會",
    "type": "union",
    "announcements": [
      {
        "id": 1,
        "title": "示範公告：連假出貨說明",
        "content": "連假期間物流暫停收件，訂單將於收假後依序出貨。",
        "audience": "public",
        "is_pinned": true,
        "published_from": "2026-08-21 18:37:53",
        "published_until": null
      }
    ],
    "featured": [
      {
        "type": "product",
        "position": 1,
        "target": { "…": "商品摘要或店面摘要" }
      }
    ],
    "shops": [
      {
        "id": 1,
        "slug": "open-seller",
        "name": "示範開業店面",
        "organization_id": 2,
        "status": "open",
        "is_buyable": true,
        "rating_avg": 0,
        "rating_count": 0,
        "favorite_count": 0
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

`storefront/organizations/PF`（平台預設組織）回 **404**。

### `announcements` —— 會員專屬公告只給本工會會員

最多 10 則。可見性由三件事共同決定（§U-04）：

| 條件 | 說明 |
| --- | --- |
| 生效期間 | `published_from` ≤ 現在 ≤ `published_until`（後者可為 `null` = 無限期） |
| `audience` | `public` 所有人可見；**`members_only` 只給本工會的會籍** |
| 排序 | `is_pinned` 的在前 |

**訪客看不到 `members_only` 的公告**，而且是「不存在」不是「被遮蔽」——
回應裡根本沒有那一筆。

> 在另一個工會有會籍的人**也看不到**這裡的 `members_only` 公告：判定看的是
> 判準是**這個 token 所屬的會籍是否屬於這個工會**，不是「這個人有沒有任何會籍」。

### `featured` —— 推薦版位也要過可購買性閘門

| 欄位 | 說明 |
| --- | --- |
| `type` | `product` \| `shop` |
| `position` | 排序，數字小的在前 |
| `target` | 商品摘要或店面摘要（形狀同商品搜尋／店面清單的單筆） |

**推薦版位的判定與商品搜尋完全相同**（§U-04）—— 工會推薦了一件已售完或已下架的
商品，它不會出現在這裡。

> 這與工會後台的推薦版位清單刻意相反：後台**不過濾**，並用 `is_currently_visible`
> 標出哪一格已經失效，好讓管理者知道要換掉它（§12.3-18）。前台只顯示還買得到的。

### `shops` —— 只列可交易的店面

`buyableForOrganization` 篩的是 `open` 的店面。已暫停、已關閉、尚未開業的
都不在這份清單裡 —— 但**直接打[店面頁](02-catalog.md#3-店面頁)仍然看得到**
已暫停與已關閉的（買家可能循訂單連過來）。

兩者的差別是「要不要主動推薦」與「連過去看不看得到」，不是同一個問題。
