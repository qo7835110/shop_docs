# 07 收藏

對應 spec B-05、§12.3-18。

## 收藏會記住你當時看到的價格

加入收藏時會快照 `price_at_favorited`，之後與 `current_price` 比對就得到
`price_dropped` —— 那是降價通知的依據（§B-05）。

通知能不能收由[通知設定](01-profile-and-notifications.md#4-通知設定)決定，
但**收藏清單本身永遠算得出 `price_dropped`**，與通知設定無關。

---

## 1. 收藏清單

```
GET buyer/favorites
```

**分頁**（分頁資訊在 `meta.pagination`）。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `type` | string | `product` \| `shop` |
| `filter` | string | `all`（預設）\| `price_dropped` \| `sold_out` \| `delisted` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "id": 1,
  "type": "product",
  "target_id": 1,
  "state": "available",
  "is_purchasable": true,
  "price_at_favorited": 180,
  "current_price": 180,
  "price_dropped": false,
  "target": {
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
    "price_from": 160,
    "member_price_available": true,
    "member_only_summary": { "level": "none", "label": null, "matches_member_only_filter": false },
    "rating_avg": 0,
    "rating_count": 0,
    "favorite_count": 1,
    "listed_at": "2026-08-24 10:34:44"
  },
  "favorited_at": "2026-08-24 10:37:56"
}
```

| 欄位 | 說明 |
| --- | --- |
| `state` | `available` / `sold_out` / `delisted` —— 對應 `filter` 的三種值。 |
| `is_purchasable` | 此刻買不買得到（涵蓋店面、上架、庫存、會員資格）。 |
| `price_at_favorited` | **收藏當下的價格快照**。 |
| `current_price` | 現在的價格。 |
| `price_dropped` | `current_price < price_at_favorited`。 |
| `target.price_from` | 該商品**最低的**規格售價（會員價若適用則反映在這裡）。 |
| `target.member_price_available` | 這個 token 所屬的會籍是否享有會員價 —— 以另一個工會的帳號登入時會不同。 |

### 已下架與售完的**不會消失**

**後台列表不過濾可購買性，前台才過濾。** 你必須看得到「這個收藏已經買不到了」，
才知道要不要移除它（§12.3-18）。用 `filter=sold_out` 或 `filter=delisted`
就能只看那些。

---

## 2. 切換收藏

```
POST buyer/favorites
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `type` | string | ✅ | `product` \| `shop` |
| `id` | integer | ✅ | 商品或店面的 ID |

### 回應 200

```json
{
  "success": true,
  "message": "已加入收藏。",
  "data": { "favorited": true, "favorite_count": 1 },
  "errors": null,
  "meta": null
}
```

**這是 toggle**：已收藏時再打一次會取消，`message` 變成 `已取消收藏。`、
`favorited` 變 `false`。

`favorite_count` 是**該商品／店面目前的總收藏數**，可直接更新畫面上的數字。

> ⚠️ **請求欄位叫 `id`，回應的收藏紀錄欄位叫 `target_id`** —— 兩者是同一個東西。
> 送 `target_id` 會得到 422（`id` 為必填）。

---

## 3. 取消收藏

```
DELETE buyer/favorites
```

欄位與第 2 條相同。回應 200，
`data` 為 `{ "favorited": false, "favorite_count": N }`，`message`：`已取消收藏。`

與 toggle 的差別：**這一支是冪等的** —— 本來就沒收藏也回 200，
不會反過來幫你加入收藏。畫面上的「移除」按鈕請用這一支，不要用 toggle，
否則連點兩下會變成重新收藏。
