# 03 購物車與結帳

對應 spec F-06、F-06.1、F-06.2、B-02、§11.3、§11.4、§13.5.2、§13.2.2。

## PAYUNi 超商選店（sandbox，§S-12.2）

```
POST buyer/shops/{shop}/payuni-store-maps
GET buyer/payuni-store-maps/{storeMap}
```

POST 僅接受以下JSON欄位，不接受query參數或其他欄位；GET不接受body或query。

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `job_key` | string | 是 | 小寫UUID，同一次選店重試沿用原鍵，不重設期限 |
| `mobile` | boolean | 否 | 手機版呈現，預設false；不影響物流型態或地圖範圍 |

兩端點皆需會員token。管理員token為401；非原買家讀回或重用工作鍵為403。不存在的店面／選店ID為404。
首次選店須有有效買家、營業中店面、原sandbox商家綁定與有效憑證，以及明確開啟的選店設定。
地圖範圍由部署設定 `PAYUNI_STORE_MAP_TYPE` 明確指定1（本島）或2（含離島）；未設定為422，**不是預設開放離島**。
`PAYUNI_SANDBOX_STORE_MAP_ENABLED` 預設false；與收款開關獨立，不因此允許付款。

POST成功為200，data包含 `id`、`shop_id`、`status`、`created_at`、`expires_at`、`store`、`form`。
時間使用含時區ISO8601；期限從工作建立時計算，預設1800秒，採原資料表參數版本。
尚未選定為pending、store為null，form為 `{action, method: "POST", fields}`；**僅由瀏覽器提交至回應指定的PAYUNi地圖網址**。
不要用GET代替POST、不要將form或其中密文寫入日誌、環境檔、session或queue。
相同工作鍵、商店及mobile可重開原有效地圖；不新增工作或延長期限。更換商店或mobile須另開工作，不改寫原紀錄。

GET的data不含form。有效回傳驗證後為selected，store包含 `code`、`name`、`address`、`insular_area`（I本島／O離島）。
門市代碼是六碼字串，保留前導零。expired表示已過期；unavailable表示原綁定／商店／目前設定不可用。
已有selected、expired或unavailable的工作不再交付表單，POST回原狀態與form=null；歷史store可保留但不因此重新可用。
憑證失效、缺配置或非法輸入為422；交付提交不明為500，須查詢原工作或沿原鍵讀回，不更換鍵盲目建立。

瀏覽器回傳由表單內固定的MapReturnURL接收，前景頁只提示登入查詢原工作，不回顯門市或原封包。
即使頁面success=true，也不代表驗證通過；以原買家的GET結果為準。
所有回應含 `Cache-Control: private, no-store` 及 `Referrer-Policy: no-referrer`。

**選店 API 不再是結帳前置（Plan32-4，2026-10-15）。** PAYUNi 付款頁（UPP）沒有門市參數，
預選的門市無法帶入實際物流單，因此超商門市改由買家在 PAYUNi 付款頁選擇、於付款成立時凍結到子訂單。
本節 API 保留，供日後對已建立的物流單重選取件門市（地圖 Tag=4）；結帳帶 `store_map_id` 一律 422。

## 不同店面分開結帳

**一次結帳只包含一家店面**，產生**一張主訂單**與**一張子訂單**（§F-06、§11.3）。
購物車可以放多家店的商品，但每次明確選一家結帳、分別付款 —— 沒有任何一個請求
會自動拆成多筆扣款。因此購物車從一開始就按店面分組：運費、配送方式與發票都是
**逐店**決定的，各店也各自確認金額。

選哪一家店由 `shipments` 那**唯一一組**的 `shop_id` 指定（見第 5 條）。

> **既有的多子訂單仍讀得到。** 主／子分層沒有改變，訂單回應的 `sub_orders`
> 仍是陣列 —— 新訂單恆只有一個元素。分開結帳不回頭改寫、合併或刪除
> 已成立的訂單（§11.3）。

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

> **同一列剛好被你自己的結帳結掉時也是 404。** 結帳會鎖住該店面的購物車列、
> 成功後移除本次結帳的那幾列（第 6 條）；若你在另一個分頁同時送出調整數量或
> 移除，這個請求會等結帳的交易結束，然後得到 404 —— 那一列真的不在了。
> 回 200 會讓「已被結帳移除」與「移除成功」長得一模一樣。
>
> 同一個 SKU 被兩個請求同時加入則會**累加**成一列，件數不會掉（§S-06）。

---

## 5. 結帳試算

```
POST buyer/checkout/preview
```

**不建立訂單、不預留庫存。**

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `items` | array | — | **省略時以所選店面的購物車品項結帳**；提供時為「直接購買」（§F-06.2） |
| `items[].product_sku_id` | integer | 有 `items` 時 ✅ | —— |
| `items[].quantity` | integer | 有 `items` 時 ✅ | ≥ 1 |
| `shipments` | array | ✅ | 配送與發票資訊，**恰一組**（`size:1`）。多於一組一律 422，**同一家店的兩組也不例外** |
| `shipments[].shop_id` | integer | ✅ | **本次結帳的店面**，所有品項都必須屬於它 |
| `shipments[].shipping_method` | string | ✅ | `home` \| `convenience_store` \| `mail` \| `union_pickup` \| `self_pickup` |
| `shipments[].recipient_name` | string | ✅ | 最長 60；超商／宅配另須符合 PAYUNi 取件人規則：2–5 個中文字或 4–10 個英文字母 |
| `shipments[].recipient_phone` | string | ✅ | 最長 30；超商／宅配須為 09 開頭 10 碼手機 |
| `shipments[].postal_code` | string\|null | — | 最長 10；超商禁止提供，包含 null |
| `shipments[].shipping_address` | string\|null | — | 最長 255；宅配／郵寄必填，宅配最長 120；超商禁止提供 |
| `shipments[].pickup_store_code` | string\|null | — | 最長 32；僅限本地合成示範，正式路徑禁止提供 |
| `shipments[].pickup_store_name` | string\|null | — | 最長 120；僅限本地合成示範，正式路徑禁止提供 |
| `shipments[].invoice_type` | string\|null | — | `duplicate` \| `triplicate` |
| `shipments[].buyer_tax_id` | string\|null | — | 恰 8 碼 |
| `shipments[].invoice_title` | string\|null | — | 最長 120 |
| `shipments[].carrier` | string\|null | — | 最長 64 |
| `shipments[].donation_code` | string\|null | — | 最長 16 |

### 超商門市在付款頁選擇

超商取貨結帳時**不提供任何門市資料**：門市代碼、名稱、地址、郵遞區號與 `store_map_id` 皆 422
（含 null 也拒絕）。子訂單建立時門市欄位為 null，運費依平台超商費率計算；買家付款時在 PAYUNi
付款頁選門市，付款成立後才凍結門市快照（§11.5）。取件人姓名與手機在結帳即依 PAYUNi 規則驗證，
避免到付款頁才失敗而留下待付款單。

只有 `local`／`testing` 且明確設定 `LOGISTICS_DEMO_ENABLED=true` 時，可使用自填的合成門市；
`production` 不受此示範開關影響。

### `shipments` 恰一組 —— 它同時是選店的依據

不另設第二個店面欄位：兩個欄位就會有互相矛盾的一天。送兩組（不論是不是同一家店）
一律 **422**，不會取第一組、也不會讓後一組覆蓋前一組。

| 送什麼 | 結果 |
| --- | --- |
| 一組 `shipments` ＋ 該店的品項 | ✅ 一張主訂單 ＋ 一張子訂單 |
| 一組 `shipments` ＋ 混了別家店的品項 | 422，**不會**靜默過濾掉別家店的品項 |
| 一組 `shipments` ＋ 品項全屬另一家店 | 422，**不會**改用品項的店面 |
| 兩組 `shipments`（不同店） | 422 |
| 兩組 `shipments`（同一家店） | 422，**不會**互相覆蓋 |

這道防護在後端的**兩層**各擋一次：欄位數量由 FormRequest 擋，
「這些 SKU 究竟屬於哪一家店」由服務層查資料庫擋。前端藏不藏多店選項與此無關。

### 從購物車結帳（省略 `items`）

只讀**目前買家在該店面**的購物車品項（§F-06.1）；該店面沒有品項時回 422。
不會預設取全部店面。

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
| `payable_amount`（外層） | 主訂單應付。分開結帳後 `shops` 恆只有一家，因此它等於該店應付；**稅額不參與加總**（§11.3、§13.4）。 |
| `tax_amount` | 由商品小計反推，四位小數。**它不是要加上去的**，售價已含稅。 |
| `discount_amount` | 第一階段**恆為 0**。 |

> `shops` 仍是陣列（恆一個元素），回應形狀不因分開結帳而改變 ——
> 讀取端不必為新舊訂單分兩種解析。

> **試算與正式送出跑同一個計算器** —— §13.5.2 要求「結帳頁必須在送出前明確揭示
> 最終成交價與適用依據，不得於付款後才變動」。兩套算法遲早會不一致，所以只有一套。
>
> **成本不會出現在這裡。** 那是賣家的營業資料（§2.3）。

### 配送方式的可用性是**交集**

平台 → 工會 → 店面 → 商品限制，取交集（§13.2.2）。下層開不了上層已停用的方式；
交集為空時該店的商品**不得結帳**（§12.3-62），不是拿平台預設運費硬算一個出來。

**PAYUNi 物流開通只在試算與送出時判定**（Plan32-3）：超商取貨須該店面已開通 7-ELEVEN
店到店常溫、宅配須已開通黑貓宅配，依據是收款同一份新鮮的商店狀態觀察。商品詳情的
`shipping_methods` 不反映這一層；未綁定、未開通、狀態未確認或已過期時，試算即回 422
並說明原因，請改選其他配送方式。

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

1. 確認品項全屬所選店面 —— 不符即 422，且**不留下任何副作用**。
2. **依 SKU 編號排序後**逐一鎖定庫存（排序是為了避免兩張訂單互相等待）。
3. 以**鎖定後的**庫存重新判定可購買性 —— 試算到送出之間可能隔了幾分鐘，
   商品可能已經下架、售完或被停權。**這是唯一算數的一道判定。**
4. 解析成交價與費率並寫入快照。
5. 建立主訂單、該店面的子訂單（編號 `-01`）與逐品項。
6. 建立**待付款階段**的庫存預留。
7. 以購物車結帳時，移除**本次讀進這張訂單的那幾列**購物車品項。
8. 發出訂單通知（不可關閉）。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | `shipments` 不是恰一組 |
| 422 | 品項混了別家店，或全屬 `shipments[].shop_id` 以外的店面 |
| 422 | 以購物車結帳，但所選店面在你的購物車裡沒有品項 |
| 422 | 金額與 `expected_payable_amount` 不符 |
| 422 | 某項此刻買不到（下架、售完、店面停權、會員限定資格不符） |
| 422 | 你本人不可交易（`is_tradable` 為 `false`） |
| 422 | 選了超商取貨但**未取貨次數已達上限**（§14.7.3，見 [09](09-pickup-failures.md)） |
| 422 | 該店的配送方式交集為空 |
| 422 | 店面未開通所選的 PAYUNi 物流通路，或開通狀態尚未確認／已過期（§13.2.2） |

失敗一律是**零副作用**：不建立訂單、不建立付款、不預留庫存、不發成功通知，
也不動購物車（§F-06）。

送出成功後**只有本次結帳讀到的那幾列購物車品項會被移除** —— 其他店面的品項保留，
結帳期間另外加入的品項也不受影響（§F-06.1）。用 `items` 直接購買則完全不動購物車
——**同一個 SKU 同時在購物車裡也一樣**。購物車裡那一筆是另一個還沒結帳的意圖，
數量也可能不同，不會被這張訂單抵掉或移除（§F-06.2）。

要買兩家店的東西，就送兩次 `POST buyer/checkout`、付兩次款。A 店的訂單取消、
付款失敗或退款都不影響 B 店那一張（§12.3-4、§12.3-13）。

會員價與會員限定資格另依有效會籍及商城狀態判斷（spec §13.5.2）。退會或停權後即使仍可登入履約，也不能以會員限定資格加入購物車；商品顯示與結帳皆重新驗證。
