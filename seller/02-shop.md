# 02 店面

對應 spec S-02、S-03、S-10、§11.2、§P-07.2。

## 店面狀態只有四種

審核期的狀態全在**申請單**上（[01](01-shop-application.md)）；店面只承載營業狀態：

```
approved ──(你自己點開始營業)──> open ──pause──> suspended ──resume──> open
                                      open / suspended ──close──> closed   ← 沒有回頭路
```

**`resume` 不在賣家端**：解除暫停由處置者或更高層級執行。你自己下的休假由平台或工會
解除，也可以由你在 `open` 端點解除 —— 見第 3 條。

---

## 1. 賣家中心總覽

```
GET seller/shops/{shop}/overview
```

一次拿齊「店面現況 + 該做什麼」。

### 回應 200

```json
{
  "success": true,
  "message": "取得賣家中心總覽。",
  "data": {
    "shop": {
      "id": 1,
      "slug": "open-seller",
      "name": "示範開業店面",
      "description": "開發用示範資料。",
      "organization_id": 2,
      "status": "open",
      "is_buyable": true,
      "invoice_capability": "none",
      "customer_service_contact": "0912345678",
      "pickup_note": null,
      "shipping_note": null,
      "return_note": null,
      "favorite_count": 0,
      "rating_avg": 0,
      "rating_count": 0,
      "opened_at": "2026-08-24 05:01:12",
      "owner_member_id": 2,
      "approved_at": "2026-08-24 05:01:12",
      "suspended_reason": null,
      "suspended_by_type": null,
      "closed_reason": null,
      "disposition_level": "none"
    },
    "pending_actions": [],
    "todo": {
      "awaiting_shipment": 5,
      "overdue_shipment": 0,
      "pending_return_decision": 1,
      "awaiting_inspection": 1,
      "undisposed_defective_skus": 1,
      "pending_review_reply": 0,
      "low_stock_skus": 0,
      "low_stock_threshold": 5,
      "pending_cancel_response": 1,
      "pending_chargeback_response": 0,
      "pending_violation_appeal": 0,
      "payout_status_counts": []
    }
  },
  "errors": null,
  "meta": null
}
```

### `todo` —— 十二個數字，每一個都對應一個**點得下去**的待辦

| 欄位 | 意義 | 連到 |
| --- | --- | --- |
| `awaiting_shipment` | 待出貨的子訂單數 | [05 訂單](05-orders-and-shipping.md#1-訂單列表) |
| `overdue_shipment` | **已逾出貨期限**的子訂單數。逾期會自動取消並全額退款，且記 3 點違規（§14.3） | 同上 |
| `pending_return_decision` | 等你回應的退貨申請。**逾期視為同意**（§14.4） | [06 退貨](06-returns-and-refunds.md#2-回應退貨申請) |
| `awaiting_inspection` | 已收到、等你驗收的退貨。**逾期視為驗收通過** | [06 驗收](06-returns-and-refunds.md#3-退貨驗收) |
| `undisposed_defective_skus` | 有未處置瑕疵品的 SKU 數 —— **不處置會永久扣減可售庫存**（§S-05.4） | [04 瑕疵品處置](04-inventory.md#6-瑕疵品處置) |
| `pending_review_reply` | 尚未回覆的評價數 | [08 評價](08-reviews-and-messages.md#2-回覆評價) |
| `low_stock_skus` | 低於安全庫存的 SKU 數 | [04 庫存](04-inventory.md#1-庫存列表) |
| `low_stock_threshold` | 未設安全庫存時的預設警示門檻 | —— |
| `pending_cancel_response` | **等你回應的取消申請**。逾 `buyer_cancel_response_days`（工作天）有預設結果（§14.3） | [05 訂單](05-orders-and-shipping.md#1-訂單列表)（`?pending_cancel_response=1`） |
| `pending_chargeback_response` | **等你抗辯的拒付**。`response_deadline_at` 是工作天計的期限（§14.8） | [07 我的拒付](07-payouts.md#3-我的拒付) |
| `pending_violation_appeal` | **還申訴得了、但你還沒申訴**的違規記錄數。申訴期是 `violation_appeal_days`（**日曆天**），過了就沒有第二次，且申訴期間處置照舊生效（§P-07.3） | [10 違規](10-violations.md#1-本店面的違規記錄) |
| `payout_status_counts` | 撥款單依狀態分組的筆數（**沒有撥款時是空陣列**） | [07 撥款](07-payouts.md#1-撥款列表) |

> 前四項、`pending_cancel_response`、`pending_chargeback_response`
> 與 `pending_violation_appeal` 是**有期限的**，其餘沒有。畫面上把它們放在
> 一起是可以的，但排序請讓有期限的在前 —— 逾期的代價不一樣。

**每一個數字都要點得下去，而且點到的是同一批資料。** 數得出來卻列不出來的待辦
比沒有待辦更糟：它讓人以為自己已經看過了。五個對應的清單篩選：

    overdue_shipment          → GET seller/orders?status=awaiting_shipment&overdue=1
    pending_cancel_response   → GET seller/orders?pending_cancel_response=1
    undisposed_defective_skus → GET seller/inventories?only_defective=1
    low_stock_skus            → GET seller/inventories?only_low_stock=1
    pending_review_reply      → GET seller/reviews?without_reply=1
    pending_chargeback_response → GET seller/chargebacks?status=pending
    pending_violation_appeal    → GET seller/violations（看 can_appeal）

`pending_actions` 是一句可直接顯示的中文，只在兩種狀態下有內容：

| 狀態 | 內容 |
| --- | --- |
| `approved` | `審核已通過，可點選「開始營業」正式營運。` |
| `suspended` | `店面暫停中，恢復須由處置者或更高層級執行；既有訂單仍須依約履行。` |

### 店面欄位

| 欄位 | 說明 |
| --- | --- |
| `is_buyable` | **後端算出的可購買性**，僅 `open` 為 `true`。前端不要自行以 `status` 判斷。 |
| `disposition_level` | 目前的處置等級：`none` \| `warned` \| `restricted` \| `suspended`（§P-07.2）。**`restricted` 的意思是「不得新增或上架商品」**，而此時 `status` 仍是 `open`、`is_buyable` 仍是 `true` —— 兩者分不出這件事，少了它賣家會在按下上架、拿到一句 422 的那一刻才發現。完整的點數與門檻在 [10 處置現況](10-violations.md#2-處置現況)，總覽只回答「有沒有事」。 |
| `suspended_by_type` | 這筆暫停是誰下的：`seller` \| `union` \| `platform`（**不是 `admin`**）。決定誰解得掉。 |
| `approved_at` | 申請核准（＝店面被建立）的時間。**審核歷程不在這裡**，在申請單上。 |
| `rating_avg` / `rating_count` | 評分快照。全額退款的評價**仍計入**（§B-04.4）。 |
| `logo_url` / `cover_url` | 識別圖像的網址；未設定為 `null`，前台以預設樣式呈現（§S-03.1）。**回應不含儲存路徑。** |
| `service_hours` | 客服可回應的時間。**純顯示** —— 見第 2 條。 |
| `open_violation_count` / `open_violation_points` | 生效中的違規記錄筆數與點數合計。只是指標，處置等級看 `disposition_level`。 |

---

## 2. 維護店面資料

```
PATCH seller/shops/{shop}
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | — | 最長 50 |
| `description` | string\|null | — | 最長 2000 |
| `customer_service_contact` | string\|null | — | 最長 100 |
| `service_hours` | string\|null | — | 最長 100，單行文字 |
| `pickup_note` / `shipping_note` / `return_note` | string\|null | — | 各最長 1000 |
| `invoice_capability` | string | — | `triplicate` \| `duplicate_only` \| `none` |

回應 200，`data` 為店面詳情，`message`：`店面資料已更新。`

**`slug`、`organization_id` 與銀行資料不在這裡**：`slug` 是既有連結與收藏的指向，
`organization_id` 決定審核者與費率解析，銀行帳戶走第 7 條。

> `invoice_capability` 是**事前揭露**：買家在商品頁就看得到這家店能不能開三聯式，
> 才知道能不能報帳（§13.6.2）。改動會立即反映在前台。

### `service_hours` 只是顯示，不拘束任何流程

它**不**影響可購買性、**不**影響 `ship_deadline_days` 等任何期限判定、
也**不**影響客服訊息的回應期限。虛擬店面全時可下單，把下單擋在客服時間外
沒有任何理由（§S-03.2）。

> 欄位名稱刻意不叫 `business_hours`：叫「營業時間」的欄位遲早有人拿去擋結帳。
> 內容是自由文字（「平日 10:00-18:00，例假日不定時」），不是結構化時段 ——
> 真實的回應時間本來就表達不進格子裡。

**本頁所有欄位都即時生效，不經審核**（§S-03）。店面在核准之後不再回到任何審核狀態。

---

## 3. 開始營業

```
POST seller/shops/{shop}/open
```

無請求 body。`approved → open`。回應 200，`message`：`店面已開始營業。`

**核准不自動開業**是刻意的（§11.2）：備貨、上架、設運費都需要時間，
而 `open` 的那一刻商品就對外可買了。

這支也是**解除自己所下休假**的路徑（`suspended` 且 `suspended_by_type` 為 `seller`）。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 暫停是**工會或平台**下的 | 訊息會指出越級解除，可直接顯示 —— 下一步是聯絡處置者，不是再按一次 |
| 403 | 不是你的店 | `您只能維護自己的店面 (spec §2.3)。` |

---

## 4. 暫停營業

```
POST seller/shops/{shop}/pause
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data.suspended_by_type` 為 **`seller`**，
`message`：`店面已暫停營業；既有訂單仍須完成出貨、退貨與撥款 (spec §S-03、§P-07.2)。`

### 暫停 ≠ 停止履約

擋的是**新交易**：商品不再可購買、不能上架新品。**已成立的訂單照走完**出貨、退貨與撥款。
前端請把回應的 `message` 顯示出來 —— 賣家最常誤解的就是這一點，
以為按了暫停就不用出貨了，然後逾期被記違規。

> 休假前請先看總覽的 `awaiting_shipment`。

---

## 5. 關閉店面

```
POST seller/shops/{shop}/close
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`message`：`店面已關閉。`

**§11.2 沒有定義 `closed` 的回頭路。** 前端請做二次確認，並把這句話寫在確認框裡。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | **尚有進行中的訂單**。必須先讓它們走完（出貨、退貨、撥款）才關得掉 |

---

## 6. 被收藏數

```
GET seller/shops/{shop}/favorite-count
```

```json
{ "success": true, "message": "取得被收藏數。", "data": { "shop_id": 1, "favorite_count": 128 }, "errors": null, "meta": null }
```

**只有數字，沒有名單** —— 誰收藏了你的店是買家的資料（§2.3）。
店面詳情裡的 `favorite_count` 是同一個數字。

---

## 7. 撥款帳戶

```
GET seller/shops/{shop}/bank-account
PUT seller/shops/{shop}/bank-account
```

請求欄位與回應形狀**與申請階段完全相同**，見
[01 §6 撥款帳戶](01-shop-application.md#6-撥款帳戶)。差別只有兩點：

- 對象是**店面**而不是申請單（核准後帳戶跟著店面走）。
- `PUT` 的成功訊息是 `撥款帳戶已更新，並已通知店主。` —— **變更會發通知**（§X-02），
  因為帳號被改是一件店主必須立刻知道的事。

> **只有店主本人維護得了它。** 工會管理者一律不得查看賣家的銀行資料，連遮罩態都不給；
> 平台則只能**驗證**，不能代改（§S-10）。


---

## 8. 店面識別圖像

```
POST   seller/shops/{shop}/images/{slot}
DELETE seller/shops/{shop}/images/{slot}
```

`{slot}` 只有 **`logo`** 與 **`cover`** 兩個值 —— 其餘一律 **404**，
因為那是「沒有這條路」而不是「資料不合格」。

### 上傳

`POST` 為 **`multipart/form-data`**：

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `file` | file | ✅ | `jpg` / `jpeg` / `png`，最大 `shop_image_max_kb`（預設 **5120 KB＝5 MB**） |

回應 200，`data` 為店面詳情，`message`：`店面圖像已更新。`

* **不經審核，上傳即生效**（§S-03.1）—— 店面營業狀態完全不受影響。
* 兩個位置**各自獨立**：換封面不會動到 Logo。
* 同一位置重新上傳即取代前一張。

### 刪除

`DELETE` 回應 200，`message`：`店面圖像已移除。`
該位置回到 `null`，前台以預設樣式呈現。

> 刪除的是**指向**，不是實體檔案：稽核紀錄留著舊的檔案識別，
> 把實體刪掉會讓那筆紀錄指向空。實體檔案的清除屬保存期限議題（§X-04.2）。

### 超過大小或格式不符是 422

§13.7 明訂「拒絕並提示重傳，**不自動壓縮、不靜默截斷**」。
被拒絕的檔案不會寫進店面，因此重傳不需要先做任何清理。

**已關閉的店面不得變更圖像**（422）—— `closed` 是終態處置，
與其餘店面資料一致（§S-03）。


---

## 9. 店家公告

```
GET   seller/shops/{shop}/announcements
POST  seller/shops/{shop}/announcements
PATCH seller/shops/{shop}/announcements/{announcement}
POST  seller/shops/{shop}/announcements/{announcement}/unpublish
```

**不經人工審核，送出即發布**（§S-13）。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `title` | string | ✅ | 最長 100 |
| `content` | string | ✅ | 最長 2000，**純文字** |
| `status` | string | — | `draft` \| `published`（預設 `draft`） |
| `is_pinned` | boolean | — | 置頂 |
| `published_from` / `published_until` | datetime\|null | — | `published_until` 必須晚於 `published_from`（台北時間，見 [conventions](../conventions.md#時間)） |

**回應 201**，`data` 為公告，`message`：`公告已建立。`

### 沒有審核，但有一道擋在發布當下的偵測

公告**不得包含導向站外交易的聯絡方式或連結**（§X-03），命中即 **422** 並回傳
命中的樣態（連結／通訊軟體 ID／電話／站外付款用語），讓你知道要改哪裡。
判定與工會公告完全相同。

**修改同樣要過偵測** —— 少了這一道，先發一則乾淨的再改成帶 LINE ID 的就繞過去了。

> 擋下而不是標記：公告是可以重寫的正式揭示，改一句就能送出。
> 訊息之所以只標記不擋，是 §X-03.6 要求留存原文供查證 —— 那是已經發生的事實。

### 「逾生效期間自動下架」是推導，不是狀態轉換

過了 `published_until` 之後，`status` **仍然是 `published`**，只是前台看不到。
沒有排程去翻它 —— 若改以排程翻狀態，同一件事就有兩個答案（狀態欄位與生效期間），
而排程還沒跑到的那段時間裡兩者不一致。

因此賣家後台的清單多一個 **`is_currently_visible`**：

| 欄位 | 說明 |
| --- | --- |
| `status` | `draft` \| `published` \| `unpublished`（**`unpublished` 是主動下架**） |
| `is_currently_visible` | 此刻前台是否真的看得到 —— **不等於 `status === "published"`**：還沒到生效日、或已過期的公告，狀態仍是 `published` 而前台看不到 |

少了 `is_currently_visible`，你會盯著「已發布」卻找不到它出現在哪裡。

### 下架

`POST …/unpublish` 須填 `reason`（最長 255）—— 每一個下架動作都要有原因（§3.4）。
因此下架**不走** `PATCH` 的欄位更新：允許 `status` 從那裡改成 `unpublished`，
下架就變成一個沒有理由的欄位更新。

### 上限與可用狀態

* 同時處於 `published` 的公告最多 `shop_announcement_limit` 則（預設 **3**）；
  達上限再發布回 422。**草稿不佔名額。**
* **店面 `suspended` 或 `closed` 時不得新增或修改公告**（422）——
  處置期間還能對買家發話，等於繞過處置的效果。
* 沒有「限本店會員」的揭示對象：店面的買家可能來自任何組織，
  這裡沒有對應的資格判定（§S-13）。
