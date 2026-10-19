# 02 營運總覽

對應 spec P-01、§11.3、§S-09、§14.7.2。

---

## 平台營運總覽

```
GET platform/overview
```

需要 `organization.manage`。

### 查詢參數

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `from` | date | — | 統計區間起始 |
| `to` | date | — | 統計區間結束 |

**兩者都省略時是「本月至今」，不是全期間。** 區間倒置（`from` 晚於 `to`）會回 422。

賣家、工會、平台三側報表接受**同一組區間參數**，同一區間下三側數字口徑一致（§11.3）。

### 回應 200

```json
{
  "success": true,
  "message": "取得平台營運總覽。",
  "data": {
    "organizations": { "union": 9, "organization": 1, "corporation": 1, "platform": 1 },
    "shops": {
      "total": 48,
      "by_status": { "draft": 3, "pending_review": 5, "needs_info": 1, "approved": 4, "open": 33, "suspended": 1, "closed": 1 }
    },
    "products": {
      "total": 1250,
      "by_status": { "draft": 40, "under_review": 12, "pending_platform_review": 2, "approved": 60, "listed": 1100, "delisted": 20, "sold_out": 5 }
    },
    "sales": {
      "basis": "net = Σ payable − Σ processed refunds − Σ negative adjustments",
      "scope": { "type": "platform", "id": null },
      "period": { "from": "2026-07-31 16:00:00", "to": "2026-08-18 15:59:59", "from_date": "2026-08-01", "to_date": "2026-08-18", "timezone": "Asia/Taipei" },
      "gross_payable": 1284500,
      "refunded_amount": 32600,
      "negative_adjustment_amount": 1200,
      "net_sales": 1250700,
      "paid_order_count": 431
    },
    "pending_reviews": { "shop_applications": 5, "product_reviews": 12 },
    "risk": {
      "shops_at_violation_threshold": [
        {
          "shop_id": 21,
          "shop_name": "職人手作坊",
          "level": "warn",
          "points": 3,
          "warn_threshold": 3,
          "restrict_threshold": 6,
          "suspend_threshold": 10
        }
      ],
      "pickup_failure": {
        "convenience_store_orders": 120,
        "returned": 27,
        "rate": 0.225,
        "alerting": true,
        "note": "未取貨比率僅為人工查核的觸發指標，不得據以自動記點 (spec §14.7.2)。"
      },
      "open_reconciliation_discrepancies": 4,
      "failed_payouts": 1,
      "open_chargebacks": 2,
      "seller_cancellation_window_days": 30,
      "links": {
        "shops_at_violation_threshold": "platform/shops/{shop}/disposition",
        "open_reconciliation_discrepancies": "platform/reconciliation-discrepancies?status=open,watching",
        "failed_payouts": "platform/payouts?status=failed,aborted",
        "open_chargebacks": "platform/chargebacks?status=pending,responded",
        "pickup_failure": null
      }
    }
  },
  "errors": null,
  "meta": null
}
```

### 欄位說明

#### 計數類欄位

`organizations`、`shops.by_status`、`products.by_status` 都是**資料庫分組結果**：
筆數為 0 的類別／狀態**不會出現在物件中**，不是回傳 0。前端取值請用
`by_status[status] ?? 0`，不要假設鍵一定存在。

`pending_reviews` 固定有兩個鍵：`shop_applications`（待審店面申請數）、
`product_reviews`（待審商品數）。**高風險分類轉入平台複審的商品不計入 `product_reviews`**，
它們要看 `products.by_status.pending_platform_review`。

#### `sales` —— 營收口徑

**這是全站唯一的營收定義**，賣家、工會、平台三側共用：

```
net_sales = gross_payable − refunded_amount − negative_adjustment_amount
```

- `basis` 是算式的文字說明，可直接顯示在報表註腳。
- `gross_payable`（應付總額）與 `net_sales`（淨銷售額）**必須並列揭露**（§11.3）——
  前端不可只顯示其中一個。付款成功後應付金額凍結，全額退款也不會讓 `gross_payable` 變小，
  只顯示它會嚴重高估營收。
- 算式**逐項不化簡**是刻意的：化簡之後就看不出少扣了哪一項。
- 金額皆為整數 TWD。

#### `risk` —— 風險摘要是「指標」，不是「判定」

五項各自指向一份待人工查核的清單，**沒有一項會由這個端點觸發處置**：

| 欄位 | 意義 | 前端該做什麼 |
| --- | --- | --- |
| `shops_at_violation_threshold` | 違規點數接近或達門檻的店面 | 每一列帶 `shop_id`，連到 `platform/shops/{shop}/disposition` |
| `pickup_failure` | 超商取貨的未取回率 | 僅顯示。`alerting` 為 `true` 表示樣本 ≥ 10 筆且比率 ≥ 20% |
| `open_reconciliation_discrepancies` | 未結案的對帳差異筆數 | 見 `links` |
| `failed_payouts` | 失敗的撥款筆數 | 見 `links` |
| `open_chargebacks` | 進行中的拒付案件 | 見 `links` |

#### `links` —— 每一個數字自己的去處

**連結與數字由同一段程式碼給**，因此兩者不會各自漂移。這一點在這裡特別要緊：
三個數字各是**兩個狀態的聯集**——

    未結案的對帳差異 = open + watching
    失敗的撥款       = failed + aborted
    進行中的拒付     = pending + responded

而三支清單端點的 `status` 因此接受**逗號分隔的多個值**（見
[07 金流作業](07-finance.md)）。少了這一點，總覽上那個數字會
「數得出來卻列不出來」——點下去看到的筆數與畫面上的數字對不起來。

`pickup_failure` 的連結**刻意為 `null`**：它只是人工查核的觸發指標，
不得提供一鍵處置的入口（§14.7.2）。

> **未取貨比率再高也不得由此自動記點**：「買家未取貨不是賣家的違規」（§14.7.2）。
> 畫面上不要把這個數字放在賣家的違規欄位旁邊，也不要提供「一鍵記點」按鈕。

`level` 的四個值：`none`、`warn`、`restrict`、`suspend`。
`seller_cancellation_window_days` 是賣家取消率的統計視窗天數，供畫面標註用。
