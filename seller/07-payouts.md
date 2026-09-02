# 07 撥款與拒付

對應 spec S-08、§14.6、§14.8、§14.9、§13.2.1。

## 撥款金額怎麼算

```
撥款金額 = 商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整
```

回應中的 `formula` 欄位就是這一句，可直接顯示在明細頁 —— §S-08 要求賣家看得懂
「為什麼是這個數字」，所以算式跟著數字一起走。

**退貨運費不在這條算式裡**：你與物流業者直接簽約，那筆錢不經過平台，
平台也沒有紀錄（§13.2.1）。

## 沒有「申請撥款」的端點

撥款由**排程產生候選**，不由賣家申請、不由單筆訂單即時觸發（§14.9.1）。
少一條路，就少一個繞過候選條件的入口。

一張子訂單要進入撥款候選，**五項條件全部成立**：

1. 子訂單已完成，且距 `completed_at` 已逾 `payout_hold_days`（**日曆天**）。
2. **該子訂單的退貨窗口已屆滿**（送達日 ＋ `return_window_days`，日曆天）。
3. 該筆付款已完成對帳且比對相符。
4. 無進行中的退貨退款、爭議或拒付。
5. 店面銀行帳戶完整且**通過驗證**（[02 §7](02-shop.md#7-撥款帳戶)）。

> 條件 1 與 2 不能合併：兩者都是「等一段日曆天」，但**起算點不同** ——
> 前者自完成時間、後者自**送達日**，而買家可以在送達當日就點確認收貨。
> 只靠條件 1 會讓款項在買家仍有法定退貨權的當下撥出（§12.3-80）。
>
> 條件 5 是最常見的卡關原因，而它只有平台解得開。撥款遲遲沒動時先看
> `bank-account` 的 `payout_ready`。

---

## 1. 撥款列表

```
GET seller/payouts
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | `pending` \| `paid` \| `failed` \| `deferred` \| `aborted`；**不合法的值回 422** |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

> `status` 用的是**撥款自己的語彙**。要看各狀態的筆數分佈請用
> [撥款報表](09-reports.md#5-撥款報表)的 `status_counts`。

### 回應 200

```json
{
  "success": true,
  "message": "取得撥款列表。",
  "data": [
    {
      "payout_no": "P2026082400001817",
      "shop_id": 1,
      "status": "pending",
      "period_start_at": "2026-08-04 00:00:00",
      "period_end_at": "2026-08-10 23:59:59",
      "formula": "商品成交小計（退款前） + 買家運費 − 退款扣抵 − 金流費 − 平台費 − 工會分潤 − 負向調整",
      "goods_amount": 42800,
      "shipping_amount": 1200,
      "refund_amount": 1340,
      "payment_fee": 1290,
      "platform_fee": 1284,
      "union_share": 856,
      "adjustment_amount": 0,
      "net_amount": 39230,
      "retry_count": 0,
      "failed_reason": null,
      "aborted_reason": null,
      "paid_at": null,
      "sub_orders": [
        {
          "order_sub_id": 3312,
          "goods_amount": 1280,
          "shipping_amount": 60,
          "refund_amount": 0,
          "payment_fee": 40,
          "platform_fee": 38,
          "union_share": 25,
          "net_amount": 1237
        }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 3, "last_page": 1 } }
}
```

### 五種狀態，兩種是「正常但沒出款」

| `status` | 意義 | 你該做什麼 |
| --- | --- | --- |
| `pending` | 候選已產生，等待出款批次 | 等 |
| `paid` | 已出款，`paid_at` 有值 | —— |
| `deferred` | `net_amount` ≤ 0 或未達最低金額，**遞延下期繼續抵扣** | 等，不是錯誤 |
| `aborted` | 出款前再驗證未通過，原因在 `aborted_reason` | 看原因，多半是條件 3～5 |
| `failed` | 出款送出後被銀行或金流端拒絕 | 看 `failed_reason`；系統會自動重試，逾次數才停 |

> **`deferred` 與 `aborted` 都不是錯誤。** 前端不要顯示成紅色警示。
>
> **`net_amount` 可以是負數，而且如實輸出**（§14.6）：那代表你尚欠平台，
> 餘額**結轉下期繼續抵扣**，平台不會向你追收現金。畫面要顯示成 0 是另一回事，
> 但 API 不能先騙一次。

`sub_orders[]` 是逐筆子訂單的拆解 —— 撥款金額對不上時從這裡查。

---

### `expected_paid_at`：什麼時候會撥

`pending` 的撥款單帶這一格 —— 候選已經建好，剩下的是下一次出款排程
（每日固定時刻執行）。已出款、已中止、已失敗的為 `null`，
那時看 `paid_at` 或 `aborted_reason` / `failed_reason`。

**這是預期不是保證。** 出款前會在同一個交易內再驗證五項條件（§14.9.2）：
期間新增的拒付、買家申請的退貨、對帳發現的金額不符，都會讓那一次出款被攔下。

---

## 2. 撥款單詳情

```
GET seller/payouts/{payoutNo}
```

`{payoutNo}` 是**撥款單號字串**，不是數字 ID。`data` 形狀與列表單筆相同。

---

## 3. 我的拒付

```
GET seller/chargebacks
```

需要 `order.fulfill` @ 本店。**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | `pending` \| `responded` \| `won` \| `lost` |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

> **狀態值域與撥款不同**：拒付是 `pending`／`responded`／`won`／`lost`，
> 撥款是 `pending`／`paid`／`failed`／`deferred`／`aborted`，只有 `pending` 重疊。
> 帶 `status=paid` 會得到 422。

### 這個清單為什麼重要

抗辯有期限（`response_deadline_at`，**工作天**），而下一條的
`POST chargebacks/{id}/response` 吃的是**整數 id** ——
在這個清單出現之前，你沒有任何方式知道自己被拒付了，
就算知道也拿不到那個 id。

配套的是[店面總覽](02-shop.md#todo--十二個數字每一個都對應一個點得下去的待辦)的
`pending_chargeback_response`：那個數字讓你知道要來看這個清單。

### 回應 200

`data` 為拒付陣列，單筆形狀與下一條的回應相同。

---

## 4. 拒付抗辯

```
POST seller/chargebacks/{chargeback}/response
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `evidence` | string[] | — | 站內**參照**（出貨單號、對話編號），每項最長 500 |
| `attachments[]` | file[] | — | 上傳的檔案（物流商簽收單掃描），`jpg` / `jpeg` / `png`，單檔最大 `attachment_max_kb` |

### 舉證是複合的，但**至少要有一項**

兩欄各自選填，**兩者皆空回 422**（§14.8）—— 一份沒有任何依據的抗辯送到金流商
只會被駁回，而你會以為自己已經抗辯過了。

寫成「至少一項」而不是把某一欄設為必填，是因為兩種舉證**可互相替代**：
站內物件（出貨單號、對話編號）本來就查得到也不會被竄改，用參照即可；
站外的事實（簽收單）只能靠上傳保全。要求特定一種等於逼你補一份你沒有的東西。

帶檔案時請求為 `multipart/form-data`。回應的 `evidence` 與 `attachments` 並列。

回應 200，`data` 為拒付單，
`message`：`抗辯已送出，將由平台彙整送交金流商。`

### 拒付是平台無法拒絕的外部強制

發卡機構發動，不走退貨退款流程（§14.8）。你能做的只有提出佐證，
由平台彙整後送交金流商 —— **平台不裁定輸贏，發卡機構才裁定**。

```json
{
  "id": 9,
  "order_sub_no": "M2026082400011738-01",
  "amount": 1340,
  "status": "pending",
  "notified_at": "2026-08-18 09:00:00",
  "response_deadline_at": "2026-09-01 09:00:00",
  "responded_at": null,
  "resolution": null
}
```

| `status` | 意義 |
| --- | --- |
| `pending` | 待抗辯 —— 這時才送得出去 |
| `responded` | 已送出抗辯，等待結果 |
| `won` | 抗辯成立；先前建立的追扣調整單**會被撤銷** |
| `lost` | 抗辯失敗；追扣調整單維持有效，於下期撥款扣抵 |

`response_deadline_at` 依 `chargeback_response_days`（**工作天**，預設 10）推算。
**逾期就送不出去了**，前端請顯示倒數。

### 拒付對撥款的影響

- **尚未撥款**：那筆子訂單被候選條件 4 擋下，不會出款。
- **已經撥款**：登錄拒付的同時建立一張**負向調整單**追扣，
  於下期撥款的 `adjustment_amount` 出現。

> **拒付不自動記你違規**（§14.8）：盜刷與持卡人誤認都有可能。
> 查證屬實才另循 §P-07.1 建立記錄。
