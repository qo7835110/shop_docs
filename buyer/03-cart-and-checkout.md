# 03 購物車與結帳

對應 spec F-06、F-06.2、B-02、§11.3、§11.4、§13.5.2、§13.2.2。

## 購物車依店面分組

一次結帳會產生**一張主訂單**與**每家店一張子訂單**（§11.5），因此購物車從一開始
就按店面分組 —— 運費、配送方式與發票都是**逐店**決定的。

## 購物車不預留庫存

加入購物車什麼都不會發生：不佔庫存、不鎖價格。**下單那一刻才預留**（§11.4）。
因此 `purchasable_quantity` 是「此刻還買得到幾件」，隨時可能變。

---

## 1. 購物車內容

```
GET buyer/cart
```

**非分頁**，依店面分組。

```json
{
  "success": true,
  "message": "取得購物車內容。",
  "data": [
    {
      "shop": {
        "id": 1,
        "name": "示範開業店面",
        "slug": "open-seller",
        "is_buyable": true,
        "invoice_capability": "none"
      },
      "items": [
        {
          "id": 1,
          "product_id": 1,
          "product_sku_id": 1,
          "product_name": "示範包裝食品（多規格）",
          "spec_name": "小包裝 250 公克",
          "quantity": 2,
          "is_purchasable": true,
          "unavailable_reason": null,
          "purchasable_quantity": 10,
          "added_at": "2026-08-24 10:35:20"
        }
      ]
    }
  ],
  "errors": null,
  "meta": null
}
```

### 每一項都帶「此刻買不買得到」

| 欄位 | 說明 |
| --- | --- |
| `is_purchasable` | **後端算出的可購買性**，涵蓋五個條件：店面已開業、商品已上架、可售庫存 > 0、會員限定資格符合、你本人可交易。 |
| `unavailable_reason` | 買不到的**具體原因**，可直接顯示。`is_purchasable` 為 `true` 時是 `null`。 |
| `purchasable_quantity` | 此刻最多買得到幾件。**不是庫存數字** —— 它已經扣掉別人的預留了。 |
| `shop.is_buyable` | 這家店此刻是否可交易（僅 `open` 為 `true`）。 |

> **買不到的品項不會自動移除。** 留著是為了讓你知道它變了 ——
> 悄悄消失的購物車比顯示「已售完」更難解釋。

---

## 2. 加入購物車

```
POST buyer/cart
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `product_sku_id` | integer | ✅ | ≥ 1 |
| `quantity` | integer | ✅ | ≥ 1 |

**回應 201**，`data` 為**整台購物車**（不是剛加的那一項），
`message`：`已加入購物車。`

> 回傳整台是刻意的：加入一項可能改變同店其他項的運費與可購買性，
> 只回一筆的話前端得自己合併，而合併錯的那一次會讓運費顯示成舊的。

同一個 SKU 重複加入是**累加數量**，不會產生第二列。

---

## 3. 調整數量

```
PATCH buyer/cart/items/{item}
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `quantity` | integer | ✅ | ≥ 1 |

回應 200，`data` 為整台購物車，`message`：`已更新數量。`

**要移除請用第 4 條**，不是把數量設成 0（`min:1` 會擋下）。

---

## 4. 移除品項

```
DELETE buyer/cart/items/{item}
```

回應 200，`data` 為整台購物車，`message`：`已移除品項。`

別人的購物車品項編號回 **404**。

---

## 5. 結帳試算

```
POST buyer/checkout/preview
```

**不建立訂單、不預留庫存。**

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `items` | array | — | **省略時以購物車全部品項結帳**；提供時為「直接購買」（§F-06.2） |
| `items[].product_sku_id` | integer | 有 `items` 時 ✅ | —— |
| `items[].quantity` | integer | 有 `items` 時 ✅ | ≥ 1 |
| `shipments` | array | ✅ | **逐店**的配送與發票資訊，至少一筆 |
| `shipments[].shop_id` | integer | ✅ | —— |
| `shipments[].shipping_method` | string | ✅ | `home` \| `convenience_store` \| `mail` \| `union_pickup` \| `self_pickup` |
| `shipments[].recipient_name` | string | ✅ | 最長 60 |
| `shipments[].recipient_phone` | string | ✅ | 最長 30 |
| `shipments[].postal_code` | string\|null | — | 最長 10 |
| `shipments[].shipping_address` | string\|null | — | 最長 255 |
| `shipments[].pickup_store_code` | string\|null | — | 最長 32，超商取貨用 |
| `shipments[].pickup_store_name` | string\|null | — | 最長 120 |
| `shipments[].invoice_type` | string\|null | — | `duplicate` \| `triplicate` |
| `shipments[].buyer_tax_id` | string\|null | — | 恰 8 碼 |
| `shipments[].invoice_title` | string\|null | — | 最長 120 |
| `shipments[].carrier` | string\|null | — | 最長 64 |
| `shipments[].donation_code` | string\|null | — | 最長 16 |

### 回應 200

```json
{
  "success": true,
  "message": "取得結帳試算。",
  "data": {
    "shops": [
      {
        "shop": { "id": 1, "name": "示範開業店面", "slug": "open-seller", "invoice_capability": "none" },
        "items": [
          {
            "product_id": 1,
            "product_sku_id": 1,
            "product_name": "示範包裝食品（多規格）",
            "spec_name": "小包裝 250 公克",
            "condition": "new",
            "unit_price": 160,
            "price_type": "member",
            "price_basis_organization_id": 2,
            "quantity": 2,
            "subtotal": 320
          }
        ],
        "goods_amount": 320,
        "shipping_fee": 100,
        "discount_amount": 0,
        "payable_amount": 420,
        "tax_amount": 15.2381,
        "shipping_method": "home"
      }
    ],
    "payable_amount": 420,
    "tax_amount": 15.2381
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `unit_price` | **最終成交價**。`price_type` 說明它是原價（`base`）還是會員價（`member`）。 |
| `price_basis_organization_id` | 會員價**憑哪一個組織的會籍**成立 —— 以另一個工會的帳號登入時這個值會不同。 |
| `payable_amount`（外層） | 各店應付的加總。**稅額不參與加總**（§11.3、§13.4）。 |
| `tax_amount` | 由商品小計反推，四位小數。**它不是要加上去的**，售價已含稅。 |
| `discount_amount` | 第一階段**恆為 0**。 |

> **試算與正式送出跑同一個計算器** —— §13.5.2 要求「結帳頁必須在送出前明確揭示
> 最終成交價與適用依據，不得於付款後才變動」。兩套算法遲早會不一致，所以只有一套。
>
> **成本不會出現在這裡。** 那是賣家的營業資料（§2.3）。

### 配送方式的可用性是**交集**

平台 → 工會 → 店面 → 商品限制，取交集（§13.2.2）。下層開不了上層已停用的方式；
交集為空時該店的商品**不得結帳**（§12.3-62），不是拿平台預設運費硬算一個出來。

---

## 6. 送出訂單

```
POST buyer/checkout
```

欄位與第 5 條相同，多一個：

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `expected_payable_amount` | integer | — | ≥ 0 —— **試算時看到的金額** |

**回應 201**，`data` 為主訂單（見 [04 訂單](04-orders-and-payment.md#1-訂單列表)），
`message`：`訂單已成立，請於期限內完成付款。`

### `expected_payable_amount` 請一定要送

它**只用於比對，絕不作為入帳依據**（§11.3）。不一致時回 422，訊息會同時給出
兩個數字：

```
應付金額與結帳頁揭示的不一致（結帳頁 420 元，目前 460 元），請重新確認後再送出 (spec §13.5.2)。
```

那通常代表你停在結帳頁的期間價格或運費變了。不送這個欄位不會報錯，
但你就失去了這道保護。

### 送出的那一刻發生什麼

1. **依 SKU 編號排序後**逐一鎖定庫存（排序是為了避免跨店訂單互相等待）。
2. 以**鎖定後的**庫存重新判定可購買性 —— 試算到送出之間可能隔了幾分鐘，
   商品可能已經下架、售完或被停權。**這是唯一算數的一道判定。**
3. 解析成交價與費率並寫入快照。
4. 建立主訂單、逐店子訂單、逐品項。
5. 建立**待付款階段**的庫存預留。
6. 發出訂單通知（不可關閉）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 金額與 `expected_payable_amount` 不符 |
| 422 | 某項此刻買不到（下架、售完、店面停權、會員限定資格不符） |
| 422 | 你本人不可交易（`is_tradable` 為 `false`） |
| 422 | 選了超商取貨但**未取貨次數已達上限**（§14.7.3，見 [09](09-pickup-failures.md)） |
| 422 | 該店的配送方式交集為空 |

送出成功後**購物車中被結帳的品項會被清掉**；用 `items` 直接購買則不動購物車。
