# 09 報表

對應 spec S-09、§11.3、§13.2.1、§13.4。

## 全站只有一條營收算式

```
淨銷售額 = Σ 應付金額 − Σ 已處理退款 − Σ 負向調整
```

賣家報表、工會統計、平台總覽**共用同一個服務、同一條算式**（§11.3）——
同一區間下三側數字必然對得起來。回應中的 `basis` 就是這句話的文字版，
可直接顯示在報表註腳。

> **不要自行從訂單列表加總。** 應付金額在付款成功後凍結，全額退款也不會讓它變小，
> 自己加出來的一定是高估的（§11.3）。

## 區間參數（前五支查詢共用）

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `from` | date | — | 統計區間起始 |
| `to` | date | — | 統計區間結束 |

**兩者都省略時是「本月至今」，不是全期間。** 區間倒置（`from` 晚於 `to`）回 422。

庫存報表與撥款報表**不吃區間** —— 它們是現況快照，不是期間統計。

---

## 1. 銷售報表

```
GET seller/shops/{shop}/reports/sales
```

```json
{
  "success": true,
  "message": "取得銷售報表。",
  "data": {
    "basis": "淨銷售額 = Σ 應付金額 − Σ 已處理退款 − Σ 負向調整 (spec §11.3)",
    "scope": { "type": "shop", "id": 1 },
    "period": { "from": "2026-08-01 00:00:00", "to": "2026-08-24 23:59:59" },
    "gross_payable": 4800,
    "refunded_amount": 0,
    "negative_adjustment_amount": 0,
    "net_sales": 4800,
    "paid_order_count": 19,
    "gross_profit": {
      "goods_amount": 3040,
      "cost_amount": 2052,
      "fee_amount": 205,
      "gross_profit": 783,
      "items_without_cost": 0,
      "excluded_costs": "賣家實付的退貨運費由賣家與物流業者直接結算，平台未經手亦無紀錄，故未計入本表 (spec §13.2.1、§14.5.1)。"
    },
    "product_status_counts": { "draft": 2, "under_review": 1, "listed": 6 }
  },
  "errors": null,
  "meta": null
}
```

### `paid_out_amount`：實際入帳的金額

以 `paid_at` 歸屬期間 —— 與退款以結案日歸屬同一個道理：
已對外揭露的月報不因後續事件而變動。

**它與 `net_sales` 永遠不會相等**，而那正是它要存在的理由：
淨銷售額還要再扣金流手續費、平台費與工會分潤才是入帳金額（§S-08）。

少了它，你會拿看得到的 `net_sales` 去對銀行帳，然後每個月都差一截 ——
而那是正常的。

---

### `gross_payable` 與 `net_sales` 必須並列顯示

§11.3 的要求：只顯示其中一個都會誤導。付款成功後應付金額凍結，
只給 `gross_payable` 會嚴重高估；只給 `net_sales` 則看不出退了多少。

算式**逐項不化簡**也是刻意的 —— 化簡之後就看不出少扣了哪一項。

### `gross_profit` —— 毛利

```
gross_profit = goods_amount − cost_amount − fee_amount
```

| 欄位 | 說明 |
| --- | --- |
| `cost_amount` | 以**移動平均成本**計（§S-05.7）。 |
| `fee_amount` | 金流費 + 平台費 + 工會分潤。 |
| `items_without_cost` | **沒有成本資料的品項數**。大於 0 時毛利是低估的（成本算 0），畫面請一併顯示這個數字，不要只給毛利。 |
| `excluded_costs` | 一句說明，照實顯示即可。 |

> **退貨運費不在成本裡**，因為平台沒有那筆資料 —— 你與物流業者直接結算（§13.2.1）。
> 這表示實際毛利會比表上低，而少掉的部分只有你自己知道。

---

## 2. 庫存報表

```
GET seller/shops/{shop}/reports/inventory
```

**不吃區間**，回的是現況。

```json
{
  "success": true,
  "message": "取得庫存報表。",
  "data": {
    "low_stock_threshold": 5,
    "summary": { "sku_count": 13, "low_stock_skus": 0, "undisposed_defective_skus": 1 },
    "items": [
      {
        "product_sku_id": 6,
        "product_name": "示範草稿商品（供修改）",
        "spec_name": "標準包裝",
        "actual_quantity": 30,
        "reserved_quantity": 0,
        "awaiting_inspection_quantity": 0,
        "defective_quantity": 0,
        "sellable_quantity": 30
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

`items[]` 是**全部 SKU**（非分頁），四項數量與可售一併列出，
方便直接核對 §11.3 的恆等式。要逐筆的異動明細請用
[04 §2 庫存異動明細](04-inventory.md#2-庫存異動明細)。

`summary.undisposed_defective_skus` 大於 0 代表**可售庫存正被永久扣減**
（[04 §6](04-inventory.md#6-瑕疵品處置)）。

---

## 3. 訂單與出貨報表

```
GET seller/shops/{shop}/reports/orders
```

```json
{
  "basis": "淨銷售額 = Σ 應付金額 − Σ 已處理退款 − Σ 負向調整 (spec §11.3)",
  "period": { "from": "2026-08-01 00:00:00", "to": "2026-08-24 23:59:59" },
  "status_counts": { "awaiting_shipment": 5, "shipped": 2, "delivered": 1, "completed": 5, "refund_pending": 5, "pickup_returned": 1 },
  "net_sales": 4800,
  "shipment": { "awaiting_shipment": 5, "overdue_shipment": 0 }
}
```

`status_counts` 是**資料庫分組結果**：筆數為 0 的狀態不會出現在物件裡。
取值請用 `status_counts[status] ?? 0`。

`shipment.overdue_shipment` 是**已逾出貨期限**的筆數 —— 那是會被自動取消並記違規的
（§14.3）。這個數字不該長時間非 0。

---

## 4. 退款報表

```
GET seller/shops/{shop}/reports/refunds
```

```json
{
  "attribution": "退款以結案日 (processed_at) 歸屬期間，不追溯原訂單期間 (spec §S-09)。",
  "period": { "from": "2026-08-01 00:00:00", "to": "2026-08-24 23:59:59" },
  "total_amount": 0,
  "by_reason": []
}
```

### `attribution` 決定你怎麼讀這張表

退款以**結案日**歸屬期間，**不追溯原訂單的期間**。因此七月賣出、八月退款的那一筆，
會出現在八月的退款表，而七月的銷售表不會被追溯修改。

這也是為什麼銷售報表的 `net_sales` 與「同期間退款表的 `total_amount`」不一定能直接相減 ——
兩者的歸屬規則不同。要淨額請直接讀 `net_sales`。

`by_reason` 依退款事由分組，沒有退款時是**空陣列**。

---

## 5. 撥款報表

```
GET seller/shops/{shop}/reports/payouts
```

**不吃區間。**

```json
{
  "formula": "商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整",
  "status_counts": []
}
```

`status_counts` 依撥款狀態分組（`pending` / `paid` / `failed` / `deferred` / `aborted`），
沒有撥款單時是空陣列。逐筆明細在 [07 撥款列表](07-payouts.md#1-撥款列表)。

---

## 6. 匯出

```
POST seller/shops/{shop}/reports/export
```

欄位為區間（`from` / `to`，皆可省略）。

**回應 201**：

```json
{
  "success": true,
  "message": "報表已匯出。",
  "data": {
    "id": 1,
    "row_count": 19,
    "period": { "from": "2026-08-01 00:00:00", "to": "2026-08-24 23:59:59" },
    "generated_at": "2026-08-24T05:20:00+08:00",
    "download_count": 0,
    "last_downloaded_at": null,
    "masking_note": "匯出內容的收件人姓名、電話與地址皆為遮罩態 (spec §2.3)。"
  },
  "errors": null,
  "meta": null
}
```

**檔案不在這個回應裡** —— 下載是另一支端點（第 8 條）。
「下載須先有一筆已授權的匯出紀錄」就是這條規則的實作方式（§2.3）。

> **`masking_note` 值得注意**：訂單畫面上你看得到完整的收件資訊（要寄件），
> 但**匯出檔裡是遮罩的**。一份帶著幾百筆完整地址的檔案一旦離開系統就收不回來，
> 而報表的用途是對帳，不需要那些欄位。
>
> `download_count` 從 **0** 開始，不是 `null`。

沒有交易的期間也匯得出一份**空結果報表**（`row_count: 0`），不是錯誤檔（§S-09）。

---

## 7. 匯出紀錄

```
GET seller/shops/{shop}/reports/exports
```

**分頁**（固定每頁 20）。`data` 為本店面的匯出紀錄陣列，形狀同第 6 條。

**回應不含檔案路徑。** 路徑是伺服器內部的儲存位置，交出去等於交出一條繞過授權的
下載路徑（§2.3）。

---

## 8. 下載匯出檔

```
GET seller/shops/{shop}/reports/exports/{export}/download
```

回應是**檔案串流**，不是 JSON 信封。

**別的店面的匯出紀錄 ID 回 404，不是 403** —— 租戶條件寫在查詢裡，
那筆紀錄對你而言根本不存在。回 403 等於告訴對方「這個編號有效，只是不給你」，
而那本身就是一則跨租戶資訊（§2.3）。

找不到實體檔案時**同樣回 404**（檔案過期或已清理）。兩種情形回應相同是刻意的。

**每一次下載都留稽核**，並累加該筆紀錄的 `download_count` 與 `last_downloaded_at` ——
下載是**資料真正離開系統的那一刻**，產製當下反而不是（§S-09、§P-03）。
