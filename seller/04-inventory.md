# 04 庫存

對應 spec S-05、S-05.1～S-05.7、§11.3、§11.4、§11.4.2。

## 可售庫存是**算出來的**

```
sellable = actual − reserved − awaiting_return_inspection − defective
```

這條恆等式是全站的硬約束（§11.3）。四項數量各自由**庫存異動**加總而成，
可售庫存從來不是被「設定」的 —— 因此：

- 畫面上顯示「還能賣幾件」一律讀 `sellable_quantity`，不要自己減。
- 每一次數量變化都會產生一筆異動紀錄，**沒有例外**。查不到異動的數量變化就是缺陷。

## 賣家能直接操作的只有四種異動

| 類型 | 端點 | 對四項數量的作用 |
| --- | --- | --- |
| `stock_in` | [入庫](#3-入庫) | 實際 **+** |
| `stock_out` | [出庫](#4-出庫) | 實際 **−** |
| `stocktake_adjustment` | [盤點](#5-盤點) | 實際 **±**（唯一可為負的類型） |
| `defective_disposal` | [瑕疵品處置](#6-瑕疵品處置) | 實際 **−**、瑕疵 **−** |

其餘六種由交易流程產生，**沒有、也不會有讓賣家直接寫入的端點**：

| 類型 | 何時發生 | 作用 |
| --- | --- | --- |
| `reservation` | 買家下單 | 預留 + |
| `release_reservation` | 出貨前的任何取消／逾時 | 預留 − |
| `sale` | **確認出貨** | 實際 −、預留 − |
| `return_to_inspection` | 退回商品入庫 | 實際 +、待檢 + |
| `return_to_sellable` | 驗收**通過** | 待檢 − |
| `return_to_defective` | 驗收**不通過** | 待檢 −、瑕疵 + |

### 這張表裡沒有「付款成功」

不是遺漏 —— **付款成功不產生任何數量異動**（§11.4）。它只是把預留從「待付款階段」
換成「待出貨階段」，一件都沒有移動。全系統唯一的扣庫時點是 `sale`，也就是確認出貨。

因此**出貨前的取消不會有「回補庫存」的異動**：`actual` 根本沒有離開過，
釋放的是預留。前端如果在取消後找 `stock_in` 之類的補回紀錄，會永遠找不到。

---

## 1. 庫存列表

```
GET seller/inventories
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `product_id` | integer | — | 限定單一商品 |
| `only_defective` | boolean | — | **只列有未處置瑕疵品的 SKU** —— 那是待辦清單。查詢字串接受 `1`／`true`／`0`／`false`，見[慣例 §2.1](../conventions.md#21-布林查詢參數) |
| `only_low_stock` | boolean | — | **只列低於安全庫存的 SKU**。同上 |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

### 低庫存的規則只有一條

你設了 `safety_stock` 就用你設的；沒設（0）才用平台預設門檻 ——
那正是總覽上 `low_stock_threshold` 的語意：「**未設安全庫存時**的預設警示門檻」。

判準是**可售**庫存而不是實際庫存（§S-05.6）：實際庫存高但大量被預留或待檢
占用時仍屬低庫存，以實際庫存判定會讓你在無貨可賣的當下看不到警示。

每一列都有 `is_low_stock`，因此前端「可以」自己篩 —— 但**只篩得到當前這一頁**。
商品一多，客戶端篩選就安靜地給出錯的答案：畫面上有東西，只是少了幾條。

### 回應 200

```json
{
  "success": true,
  "message": "取得庫存列表。",
  "data": [
    {
      "product_sku_id": 1,
      "product_id": 1,
      "product_name": "示範包裝食品（多規格）",
      "sku_code": "DEMO-FOOD-S",
      "spec_name": "小包裝 250 公克",
      "actual_quantity": 30,
      "reserved_quantity": 1,
      "awaiting_inspection_quantity": 0,
      "defective_quantity": 0,
      "sellable_quantity": 29,
      "safety_stock": 0,
      "average_cost": 108,
      "is_low_stock": false,
      "is_out_of_stock": false,
      "requires_defective_disposal": false,
      "updated_at": "2026-08-24 05:01:15"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 13, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `sellable_quantity` | **唯一該顯示給「還能賣幾件」的數字。** |
| `average_cost` | 移動平均成本（§S-05.7），由入庫單價重算。 |
| `is_low_stock` | 可售 ≤ `safety_stock`；`safety_stock` 為 0 時用預設門檻（見總覽的 `low_stock_threshold`）。 |
| `requires_defective_disposal` | **有未處置的瑕疵品**。為 `true` 時可售庫存正被永久扣減（§S-05.4）。 |

---

## 2. 庫存異動明細

```
GET seller/inventories/{sku}/movements
```

**分頁**。倒序（最新在前）。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `type` | string | — | 上面十種異動類型之一。不在列舉內回 **422**，不是空清單 —— 靜默忽略會讓打錯字的人以為沒有資料 |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

> **`type` 不是可有可無的。** 瑕疵品處置要填 `source_order_sub_id`，
> 而賣家側**只有這份帳本輸出得了它**（訂單輸出的是 `order_no`）。
> 沒有篩選就得把整條 SKU 的進出翻一遍 —— 而瑕疵品正是賣得好的 SKU 才會累積，
> 越需要用的時候越難用。要找那一件是哪張單退回來的，用
> `?type=return_to_defective`。

```json
{
  "success": true,
  "message": "取得庫存異動明細。",
  "data": [
    {
      "id": 51,
      "type": "reservation",
      "quantity": 1,
      "reference_type": "order_sub",
      "reference_id": "20",
      "source_order_sub_id": null,
      "disposal_method": null,
      "reason": "建立訂單預留：M2026082400020680-01",
      "operator_type": "member",
      "created_at": "2026-08-24 05:01:15"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 51, "last_page": 3 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `type` | 上面十種之一。 |
| `quantity` | **恆為正**，除了 `stocktake_adjustment`（盤虧為負）。方向由 `type` 決定，不看正負號。 |
| `reference_type` / `reference_id` | 這筆異動的來源單據。交易流程產生的一定有；人工異動可能為 `null`。 |
| `source_order_sub_id` / `disposal_method` | 只有瑕疵品處置會用到（見第 6 條）。 |
| `operator_type` | `member`（你）\| `admin`（平台）\| `system`（排程）。 |

> **這張明細就是庫存的帳本。** 四項數量都能由它重算出來 —— 對不上就是缺陷，
> 不是顯示問題。

---

## 3. 入庫

```
POST seller/inventories/{sku}/stock-in
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `quantity` | integer | ✅ | ≥ 1 |
| `unit_cost` | integer\|null | — | ≥ 0，**供移動平均成本重算**（§S-05.7） |
| `reason` | string | ✅ | 最長 255 |

**回應 201**，`data` 為該 SKU 的最新庫存（形狀同列表單筆），`message`：`已完成入庫。`

`unit_cost` 選填：未帶時以 SKU 上你自填的 `cost` 為初值，兩者皆無則不動平均成本。
**出庫不看這一欄** —— 賣出不改變成本。

---

## 4. 出庫

```
POST seller/inventories/{sku}/stock-out
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `quantity` | integer | ✅ | ≥ 1 |
| `reason` | string | ✅ | 最長 255 |

**回應 201**，`data` 為最新庫存，`message`：`已完成出庫。`

> **這不是出貨。** 出貨走 [05 §4 確認出貨](05-orders-and-shipping.md#4-確認出貨)，
> 它會自己產生 `sale`。用出庫來「處理訂單」會讓同一批貨被扣兩次。
>
> 出庫是給破損、樣品、自用這類**與訂單無關**的減項用的。

出庫數量超過可售庫存時回 422 —— 不得把已被預留的貨拿走。

---

## 5. 盤點

```
POST seller/inventories/{sku}/stocktake
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `counted_quantity` | integer | ✅ | ≥ 0，**實際數到的數量** |
| `reason` | string | ✅ | 最長 255 |

送的是**盤點結果**，不是差額 —— 差額由後端算（`counted − actual`）。

### 回應 200 —— 兩種 message

| 情況 | `message` |
| --- | --- |
| 有差異 | `盤點調整已記錄。` |
| 帳實相符 | `盤點結果與帳面相符，未產生調整。` |

**相符時不產生異動紀錄**：一筆數量為 0 的調整沒有資訊，只會讓帳本變難讀。
前端請照實顯示這句話，不要說成「盤點失敗」。

---

## 6. 瑕疵品處置

```
POST seller/inventories/{sku}/defective-disposal
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `quantity` | integer | ✅ | ≥ 1，不得超過瑕疵數量 |
| `method` | string | ✅ | `return_to_buyer`（退回買家）\| `scrap`（報廢） |
| `reason` | string | ✅ | 最長 255 |
| `source_order_sub_id` | integer | ✅ | 這批瑕疵品來自哪一張子訂單 |

**回應 201**，`data` 為最新庫存，`message`：`瑕疵品處置已記錄。`

### 為什麼一定要處置

`return_to_defective` **只會增加**瑕疵數量。沒有這一支，那些數量會**永久扣減可售庫存** ——
帳面上有 30 件，可售只有 27，而且永遠不會回來（§S-05.4）。

### 為什麼不能改回可售

**瑕疵品不得在同一 SKU 上復原。** 沒有 `defective → sellable` 這條路，
而那不是疏漏：驗收判定為瑕疵的貨若能一鍵變回好貨，驗收這道程序就不存在了。
整理後仍可販售的商品請建立**新的 SKU**（例如「福利品」），並揭露狀況（§11.6）。

`source_order_sub_id` 必填，因為瑕疵品一定來自某一次退貨驗收 —— 缺了它，
這筆處置回溯不到任何事實。

---

## 7. 安全庫存

```
PATCH seller/inventories/{sku}/safety-stock
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `safety_stock` | integer | ✅ | ≥ 0 |

回應 200，`data` 為最新庫存，`message`：`安全庫存已更新。`

**`0` 是合法值，代表「不設自訂警示」**，不是「未設定」—— 此時 `is_low_stock`
改用系統預設門檻判斷。要完全關掉低庫存提示，目前沒有這個選項。
