# 08 評價與對話

對應 spec S-11、S-11.1、B-04、B-04.1、B-04.4、X-03。

## 評價你移不掉

賣家對評價只有兩個動作：**公開回覆**與**檢舉**。沒有修改、沒有刪除、沒有隱藏 ——
隱藏只有平台做得到，而且平台的預設角色裡連客服都沒有這個權限（§B-04.1）。

理由很直接：可由被評價者移除的評價沒有任何參考價值。

## 對話你退不出

買賣雙方的對話綁在子訂單上。你看得到自己店面的全部對話，
但**訊息只能隱藏不能刪除**，而隱藏同樣只有平台做得到（§X-03.4）。

---

## 1. 評價列表

```
GET seller/reviews
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `status` | string | — | `visible` \| `hidden` |
| `without_reply` | boolean | — | **只列尚未回覆的** —— 對應總覽的 `pending_review_reply` |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

> 回覆**至多一次且不可修改**（§S-11.1），因此「哪幾則還沒回」是這個畫面唯一的
> 問題：已經回過的那些，你在這裡什麼都做不了。每一列都有 `can_reply`，
> 但客戶端只篩得到當前這一頁。

### 回應 200

```json
{
  "success": true,
  "message": "取得店面評價列表。",
  "data": [
    {
      "id": 771,
      "rating": 2,
      "content": "包裝破損",
      "images": ["reviews/771/1.jpg"],
      "reviewer_name": "王＊明",
      "created_at": "2026-08-14 20:11:00",
      "fully_refunded": true,
      "fully_refunded_notice": "此筆交易已全額退款。",
      "append": null,
      "reply": { "content": "已與買家聯繫並致歉", "created_at": "2026-08-15 09:00:00" },
      "status": "visible",
      "hidden_reason": null,
      "product_id": 3301,
      "reported": true,
      "can_reply": false
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 88, "last_page": 5 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `reviewer_name` | **遮罩後**的姓名。你看不到買家全名，也沒有端點取得。 |
| `fully_refunded` | 「先評價、後全額退款」的標註。**評價保留、仍計入評分統計**，只是標註 —— 否則賣家就有動機以退款換掉負評（§B-04.4）。買賣雙方都關不掉它。 |
| `append` | 買家追評（`content` / `images` / `created_at`），沒有則 `null`。 |
| `reply` | 你的回覆，沒有則 `null`。 |
| `reported` | 是否已被檢舉（你自己送出的）。 |
| `can_reply` | **後端算好的「還能不能回覆」** —— 一則評價只能回一次。用它決定按鈕，不要自己判斷 `reply === null`。 |

> **這裡沒有任何移除欄位**，也不會有。

---

## 2. 回覆評價

```
POST seller/reviews/{review}/reply
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `content` | string | ✅ | 最長 1000 |

**回應 201**，`data` 為評價（`reply` 已填、`can_reply` 變 `false`），
`message`：`回覆已提交。`

**一則評價只能回一次**（§S-11.1）。回覆與原評價一併公開顯示，
買家看得到、其他買家也看得到 —— 它是說明，不是私訊。

要私下溝通請走[對話](#4-對話列表)。

### 不要在回覆裡做的事

- 要脅或利誘買家修改／撤下評價 → 5 點違規（`review_coercion`，§B-04.1）
- 留下站外聯絡方式 → 5 點違規（`off_platform_solicitation`，§X-03.6）

---

## 3. 檢舉評價

```
POST seller/reviews/{review}/report
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 500 |

### 回應 201

```json
{
  "success": true,
  "message": "檢舉已送出。",
  "data": {
    "id": 12,
    "review_id": 771,
    "status": "pending",
    "notice": "檢舉已受理，由平台判定；評價在判定前維持顯示 (spec §B-04.1)。"
  },
  "errors": null,
  "meta": null
}
```

**`notice` 請照實顯示。** 檢舉不會讓評價消失，也不會暫時隱藏它 ——
在平台判定之前它一直都在。這句話寫在回應裡就是為了讓前端不必自己解釋，
也不會寫成「已下架該評價」。

檢舉由**平台管理者**判定（客服沒有 `review.moderate`）—— 隱藏評價會直接改變
店面評分與商品頁，那是營運判斷，不是第一線爭議處理。

---

## 4. 對話列表

```
GET seller/conversations
```

**分頁**。

| 參數 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 你的店面 ID |
| `page` / `per_page` | integer | — | 見[共通約定](../conventions.md#分頁) |

> 沒帶 `shop_id` 回 **422**，`errors.shop_id` 說明缺什麼；帶了但那間店面不存在才回 404。
> 兩者各守一件事：前者是「沒給」，後者是「給了但沒有」。

```json
{
  "success": true,
  "message": "取得對話列表。",
  "data": [
    {
      "id": 1,
      "status": "open",
      "order_sub_id": 20,
      "order_sub_no": "M2026082400020680-01",
      "last_message_at": "2026-08-24 05:01:15",
      "last_message_sender_role": "buyer",
      "cs_present": false,
      "cs_joined_at": null,
      "cs_left_at": null,
      "buyer_name": "示範＊家"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `buyer_name` | **遮罩後**的姓名。要寄件請看[訂單的 `recipient`](05-orders-and-shipping.md#recipient-不遮罩)，那裡才是完整資料。 |
| `last_message_sender_role` | 最後一則是誰說的：`buyer` \| `seller` \| `platform_cs`。**收件匣唯一要回答的問題是「有沒有人在等我回話」** —— 只有 `last_message_at` 的話，那件事得逐則點開才知道。沒有任何訊息時為 `null`。 |
| `order_sub_no` | 這條對話掛在哪張子訂單上。訂單端點吃的是 `order_no`，只給 `order_sub_id`（整數主鍵）的話，從對話跳到訂單就沒有路。 |
| `cs_present` | **平台客服現在在不在這條對話裡** —— 由後端算出（已加入且尚未退出）。 |
| `cs_joined_at` / `cs_left_at` | 客服加入與退出的時間。 |

> **客服在場是你有權知道的事**（§X-03.5）。客服只在退款進入爭議時才加得進來，
> 加入與退出皆留痕，而 `leave` 之後他就再也讀不到了。

---

## 5. 讀取訊息

```
GET seller/conversations/{conversation}/messages
```

**分頁**。

```json
{
  "id": 33,
  "sender_type": "member",
  "sender_role": "buyer",
  "sender_id": 501,
  "content": "您好，商品已寄出",
  "attachments": [],
  "status": "visible",
  "hidden_reason": null,
  "created_at": "2026-08-17 21:40:00"
}
```

`sender_type` 為 `member`（買家或你）或 `admin`（平台客服）—— **它分不出買賣雙方**，
因為兩邊都是 `member`。

因此每一則另外帶 `sender_role`：`buyer` \| `seller` \| `platform_cs`。
沒有它的話，誰說了哪一句只能拿 `sender_id` 去比對買家會員 id 與店主會員 id，
而那兩個 id 你在自己的畫面上拿不到 —— 連左右對齊都畫不出來。

§X-03 的對話存在的理由是爭議裁定要看「當時說了什麼」。分不出說話的人，
那份紀錄的證據力就少一半。

**被平台隱藏的訊息，你看到的是 `此訊息已由平台隱藏。`**，附件一併隱藏 ——
原文只有平台視角看得到（§X-03.4）。`status` 為 `hidden` 時 `content` 就是那句話，
不要以為是內容真的變了。

---

## 6. 發送訊息

```
POST seller/conversations/{conversation}/messages
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `content` | string | ✅ | 最長 2000 |
| `attachments` | string[] | — | 每項最長 500 |

**回應 201**，`data` 為訊息單筆，`message`：`訊息已送出。`

### 敏感內容會被標記，而且標記不會消失

訊息會經過偵測（身分證號、銀行帳號、外部連結、LINE 帳號、通訊軟體帳號、
電子郵件、電話號碼、站外交易用語）。命中不會擋下發送，但那則訊息會進入平台的
**待查清單**（§X-03.6），並可能成為違規記錄的佐證。

兩件事在對話裡做特別容易被抓到，而且點數都不低：

| 做了什麼 | 點數 |
| --- | --- |
| 提供私人匯款帳號、誘導站外交易 | 5（`off_platform_solicitation`） |
| 要求買家分擔退貨運費 | 3（`return_shipping_shifted`，§14.5.1） |

> 誘導站外交易的實質後果不只是點數：交易脫離平台的金流、退貨與稽核保護後，
> **買家失去退款依據**，而爭議最後仍會回到平台，只是那時已經沒有任何紀錄可查。
