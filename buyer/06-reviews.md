# 06 評價

對應 spec B-04、B-04.1、B-04.4、S-11.1、§2.3。

## 三格狀態：待評價 → 已評價 →（已追評）

一個訂單品項只走這條路，而且**每一格只進一次**：

- 評價**不可修改、不可刪除**。
- 追評**只能一次**。
- 賣家可以公開回覆一次，但**移不掉你的評價**（§B-04.1）。

平台可以隱藏違規評價，但那是狀態變更、原文保留，且只有平台管理者做得到 ——
客服都沒有這個權限。

---

> **評價與追評的照片是真實檔案上傳**（§13.7）：兩支端點都接受
> `multipart/form-data`，不帶照片時仍可用 JSON。回應裡的 `images` 是**可直接顯示
> 的網址**，不是儲存路徑 —— 換磁碟或加 CDN 都不該要求前端改程式。
>
> 照片存在 append-only 的附件表，因此「評價不可竄改」（§B-04.2）在資料層就成立。

## 1. 我的評價

```
GET buyer/reviews
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `shop_id` | integer | 限定店面 |
| `status` | string | `visible` \| `hidden` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

```json
{
  "id": 771,
  "rating": 4,
  "content": "包裝紮實，出貨很快",
  "images": ["http://…/storage/attachments/review/771/aaa.jpg"],
  "reviewer_name": "王＊明",
  "created_at": "2026-08-14 20:11:00",
  "fully_refunded": false,
  "fully_refunded_notice": null,
  "append": null,
  "reply": { "content": "感謝支持！", "created_at": "2026-08-15 09:00:00" },
  "status": "visible",
  "hidden_reason": null,
  "product_id": 3301,
  "order_sub_no": "M2026082400012328-01"
}
```

| 欄位 | 說明 |
| --- | --- |
| `reviewer_name` | **你自己的名字也是遮罩的** —— 評價是公開內容，全名不曾被同意公開（§2.3）。 |
| `status` / `hidden_reason` | 被平台隱藏時 `status` 為 `hidden`，`hidden_reason` 是理由。**你自己看得到自己被隱藏了**。 |
| `fully_refunded` | 「先評價、後全額退款」的標註。**評價保留、仍計入店家評分**；買賣雙方都關不掉它（§B-04.4）。 |
| `append` | 你的追評。 |
| `reply` | 賣家的公開回覆。 |

> **這裡沒有修改或刪除端點**，也不會有。

---

## 2. 待評價清單

```
GET buyer/reviews/pending
```

**非分頁**，`data` 為純陣列（`meta` 為 `null`），元素形狀如下：

```json
{
  "order_item_id": 12,
  "order_sub_no": "M2026082400012328-01",
  "product_id": 1,
  "product_name": "示範包裝食品（多規格）",
  "spec_name": "小包裝 250 公克",
  "quantity": 1,
  "is_reviewable": true,
  "unavailable_reasons": [],
  "review_window_ends_at": "2026-09-23 23:59:59"
}
```

| 欄位 | 說明 |
| --- | --- |
| `is_reviewable` | 此刻能不能評價。 |
| `unavailable_reasons` | **不能評價的具體原因**（陣列，可直接顯示）。回布林就答不出「為什麼不能」。 |
| `review_window_ends_at` | 評價期限：子訂單完成後 `review_window_days`（**日曆天**，預設 30）。 |

> 清單裡**包含已經不能評價的品項**（例如已過期），並用上面兩個欄位說明原因 ——
> 悄悄消失會讓人以為系統忘了。

---

## 3. 撰寫評價

```
POST buyer/order-items/{orderItem}/review
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `rating` | integer | ✅ | 1 ～ 5 |
| `content` | string\|null | — | 最長 2000 —— **只給星等不寫字是可以的** |
| `images[]` | file[] | — | `jpg` / `jpeg` / `png`，單檔最大 `attachment_max_kb`（預設 5 MB）；張數上限 `review_image_limit`（預設 5） |

**回應 201**，`data` 為評價，`message`：`評價已提交。`

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 403 | 不是你買的品項 |
| 422 | 子訂單尚未完成 |
| 422 | **已超過評價期限** |
| 422 | 該品項已經評價過（一個品項一則） |

> **評價一經提交不可修改。** 前端請做一次確認，並把「送出後只能追評一次」寫在確認框裡。

---

## 4. 追評

```
POST buyer/reviews/{review}/append
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `content` | string | ✅ | 最長 2000 |
| `images[]` | file[] | — | 同上 |

**回應 201**，`data` 為評價（`append` 已填），`message`：`追評已提交。`

**每則評價只能追評一次**（§B-04）。追評不會改變原評價的內容與星等 ——
它是另外一段，公開顯示在原評價之後。

### 如果賣家用好處換你改評價

那是**違規**（`review_coercion`，5 點，§B-04.1）：以退款、補償、贈品或威脅
換取修改或撤下評價。對話紀錄就是佐證 —— 你在對話裡收到那種訊息時，
它同時也會進入平台的待查清單（§X-03.6）。
