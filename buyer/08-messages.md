# 08 訊息

對應 spec X-03、X-03.3、X-03.4、X-03.5、X-03.6。

## 對話綁在店面上，通常也綁在子訂單上

一條對話屬於「你與某一家店」，多數情況下還指向一張子訂單 ——
爭議發生時客服要看得出在談哪一筆交易。

## 訊息不能刪

只有平台能隱藏違規訊息，且採**狀態變更、原文保留**（§X-03.4）。
你看到被隱藏的訊息會變成 `此訊息已由平台隱藏。`，附件一併隱藏。

---

> **附件是真實檔案上傳**（§13.7）。它存在 append-only 的附件表，
> 因此「訊息不得由買賣家刪除，僅平台可隱藏」（§X-03.4）在資料層就成立 ——
> 平台的隱藏處置作用在訊息上，附件本身不動。

## 1. 對話列表

```
GET buyer/conversations
```

**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `shop_id` | integer | 限定店面 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

```json
{
  "id": 1,
  "status": "open",
  "order_sub_id": 18,
  "order_sub_no": "M2026082400018-01",
  "last_message_at": "2026-08-24 10:34:47",
  "last_message_sender_role": "seller",
  "cs_present": false,
  "cs_joined_at": null,
  "cs_left_at": null,
  "shop": { "id": 1, "name": "示範開業店面", "slug": "open-seller" }
}
```

| 欄位 | 說明 |
| --- | --- |
| `status` | `open` \| `closed` |
| `order_sub_id` | 對應的子訂單；沒有指定訂單的一般詢問為 `null` |
| `order_sub_no` | 同一張單的編號。訂單端點吃的是編號，只給整數主鍵的話從對話跳不到訂單 |
| `last_message_sender_role` | 最後一則是誰說的：`buyer` \| `seller` \| `platform_cs`。要回答「賣家回我了嗎」用它，不是靠 `last_message_at` |
| `cs_present` | **平台客服現在在不在這條對話裡** —— 後端算出（已加入且尚未退出） |
| `cs_joined_at` / `cs_left_at` | 客服加入與退出的時間 |

### 客服在場是你有權知道的事

客服**只在退款進入爭議時**才加得進來（§X-03.5），加入與退出皆留痕，
而 `leave` 之後他就再也讀不到內容了。

前端請在 `cs_present` 為 `true` 時明確顯示「平台客服已加入本對話」——
那不只是給稽核看的。

---

## 2. 發起對話

```
POST buyer/conversations
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | —— |
| `order_sub_no` | string\|null | — | 最長 32，子訂單編號 |

回應 `data` 為對話摘要（形狀同第 1 條）。

**同一店面（同一子訂單）已有進行中的對話時，回的是既有那一條**，
不會產生第二條 —— 否則同一筆交易的對話會散在好幾串裡，而客服只加得進其中一條。

---

## 3. 讀取訊息

```
GET buyer/conversations/{conversation}/messages
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

| 欄位 | 說明 |
| --- | --- |
| `sender_type` | `member`（你或賣家）\| `admin`（平台客服）。**它分不出你和賣家** —— 兩邊都是 `member` |
| `sender_role` | `buyer` \| `seller` \| `platform_cs`。訊息氣泡要靠它決定左右；沒有它就只能拿 `sender_id` 去比對會員 id，而那個 id 你在畫面上拿不到 |
| `status` | `visible` \| `hidden` |
| `content` | **被隱藏時是 `此訊息已由平台隱藏。`**，不是原文消失 |
| `attachments` | 可直接顯示的網址；**被隱藏時為空陣列** —— 隱藏的是內容，附件同屬內容（§X-03.4） |

第三者讀取別人的對話回 **403**。

---

## 4. 發送訊息

```
POST buyer/conversations/{conversation}/messages
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `content` | string | ✅ | 最長 2000 |
| `attachments[]` | file[] | — | `jpg` / `jpeg` / `png`，單檔最大 `attachment_max_kb`（預設 5 MB），最多 `attachment_max_count` 個（預設 5）。**帶附件時本端點為 `multipart/form-data`** |

**回應 201**，`data` 為訊息單筆，`message`：`訊息已送出。`

### 附件是瑕疵舉證的主要形式

**原始檔會被保留**（§X-03.3）—— 退貨爭議走到客服裁定時，那些照片就是證據。
前端請不要在上傳前壓縮到看不出瑕疵。

### 敏感內容會被標記

訊息會經過偵測（身分證號、銀行帳號、外部連結、LINE 帳號、通訊軟體帳號、
電子郵件、電話號碼、站外交易用語）。命中不會擋下發送，但那則訊息會進入平台的
**待查清單**（§X-03.6）。

**這對你也適用**，不只賣家。不過真正的風險在另一邊：

> 賣家提供私人匯款帳號、要你「直接匯款比較便宜」是**違規**（5 點）。
> 更重要的是實質後果：交易脫離平台的金流、退貨與稽核保護之後，
> **你會失去退款依據** —— 爭議最後仍會回到平台，只是那時已經沒有紀錄可查。
>
> 賣家要求你分擔**退貨運費**同樣是違規（3 點）：那筆錢本來就是他要出的（§14.5.1）。
