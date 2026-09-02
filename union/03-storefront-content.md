# 03 工會商城內容（公告、推薦版位）

對應 spec U-04、F-02、P-03、§X-03、§12.3-18。

這兩組端點決定工會商城前台長什麼樣子。**後台與前台的可見性規則刻意不同** ——
管理者要看得到「這一則還沒開始」「這一格已經不能推薦了」，前台則只看得到現在能看的。

---

# 公告

## 1. 公告列表

```
GET union/announcements
```

需要 `announcement.manage` @ 本工會。**分頁**。

範圍固定為操作者自己的工會，不接受組織參數。

**後台看得到全部**：包含尚未生效、已逾期的公告。排序為置頂優先、再依發布時間新到舊。

### 回應 200

```json
{
  "success": true,
  "message": "取得公告列表。",
  "data": [
    {
      "id": 18,
      "title": "中秋節出貨時間調整",
      "content": "9/28 至 10/1 暫停出貨，訂單將於 10/2 起依序處理。",
      "audience": "public",
      "is_pinned": true,
      "published_from": "2026-09-20 00:00:00",
      "published_until": "2026-10-05 23:59:59"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 12, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `audience` | `public`（所有人可見）\| `members_only`（僅本工會會員可見） |
| `is_pinned` | 置頂。前台與後台都優先排序。 |
| `published_from` | 生效起始。**未來時間表示尚未在前台顯示**。 |
| `published_until` | 生效結束；`null` 為無限期。 |

> **沒有 `status` 欄位**：公告的可見性是由 `published_from` / `published_until` 與
> `audience` **推導**出來的，不是一個儲存的狀態。前端要顯示「進行中／未開始／已結束」的話，
> 自行以現在時間與這兩個欄位比對即可。

---

> 每一則都帶 `is_currently_visible`：`published_from ≤ now ≤ published_until`
> 的答案，由後端算好。讓每個前端各比一次日期，時區與邊界就會各錯一次 ——
> 而公告是**前台會顯示**的東西：算錯的那一次，過期的公告會留在首頁上。
>
> 「還沒開始」與「已過期」都是 `false`，但在畫面上是兩種不同的話 ——
> 要分辨請看 `published_from` 與 `published_until`。

---

## 2. 建立公告

```
POST union/announcements
```

需要 `announcement.manage` @ 本工會。公告自動歸屬於操作者自己的工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `title` | string | ✅ | 最長 120 |
| `content` | string | ✅ | 最長 5000 |
| `audience` | string | — | `public`（預設）\| `members_only` |
| `is_pinned` | boolean | — | 預設 `false` |
| `published_from` | date | ✅ | 生效起始 |
| `published_until` | date\|null | — | 生效結束，預設 `null`（無限期） |

```json
{
  "title": "中秋節出貨時間調整",
  "content": "9/28 至 10/1 暫停出貨，訂單將於 10/2 起依序處理。",
  "audience": "public",
  "is_pinned": true,
  "published_from": "2026-09-20 00:00:00",
  "published_until": "2026-10-05 23:59:59"
}
```

**回應 201**，`data` 為公告單筆，`message`：`公告已建立。`

### 內容檢查：不得導向站外交易

標題與內容都會被掃描，命中就是 **422**，訊息會指出偵測到什麼：

```
公告內容不得包含導向站外交易的聯絡方式或連結（偵測到：LINE 帳號、站外交易用語）(spec §U-04)。
```

被擋下的六類：

| 類別 | 例子 |
| --- | --- |
| 外部連結 | `https://…`、`www.…` |
| LINE 帳號 | 「加 LINE」、「LINE ID：」、「賴 ID」 |
| 通訊軟體帳號 | wechat／微信／whatsapp／telegram／skype 後接帳號 |
| 電子郵件 | 任何 `xxx@yyy.zz` |
| 電話號碼 | 市話與手機格式 |
| 站外交易用語 | 「私訊」「私下」「匯款」「轉帳」「面交」「直接聯絡」 |

> 這是**全文比對，不看語意**：「請勿私下匯款」這種善意提醒也會被擋。
> 前端最好在送出前就提示，或在 422 時把訊息原樣顯示 —— 訊息已經指名是哪一類。
>
> 客服聯絡方式應該寫在店面資料（`customer_service_contact`）而不是公告裡。

### 其他失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 跨工會 | `您只能維護本工會的公告，不得跨工會存取 (spec §2.3)。` |
| 403 | 沒有權限 | `您沒有管理公告的授權 (spec §U-04)。` |
| 403 | 用平台身分呼叫 | `工會公告由工會管理者維護，平台人員不得代發 (spec §U-04)。` |
| 422 | `published_until` 早於或等於 `published_from` | `公告的結束時間必須晚於開始時間 (spec §U-04)。` |

---

## 3. 更新公告

```
PATCH union/announcements/{announcement}
```

需要 `announcement.manage` @ 該公告所屬工會。全部欄位皆為選填，只送要改的。

| 欄位 | 型別 | 規則 |
| --- | --- | --- |
| `title` | string | 最長 120 |
| `content` | string | 最長 5000 |
| `audience` | string | `public` \| `members_only` |
| `is_pinned` | boolean | — |
| `published_from` | date | — |
| `published_until` | date\|null | — |

回應 200，`data` 為公告單筆，`message`：`公告已更新。`

內容檢查與期間檢查同建立；期間檢查會**把未送出的欄位與現有值合併後**再驗，
所以只改 `published_until` 也可能因為早於現有的 `published_from` 而被擋。

修改會記錄新舊值稽核 —— 公告曾經說過什麼，是爭議時的事實依據。

---

## 4. 下架公告

```
DELETE union/announcements/{announcement}
```

需要 `announcement.manage` @ 該公告所屬工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

> `DELETE` 帶 body 是刻意的：**下架必須有理由**。前端的 HTTP 客戶端要確認能在
> DELETE 帶 JSON body（fetch/axios 都可以，部分封裝預設會丟掉）。

回應 200，`data` 為 `null`，`message`：`公告已下架。`

### 這是軟刪除

公告不會真的消失 —— 它從前台與後台列表移除，但紀錄保留，理由與操作者寫入稽核。
**沒有復原端點**：要重新公告請建一則新的。

> **逾期的公告不需要走這裡。** 可見性是推導值，`published_until` 到了自然就不顯示。
> 下架是用在「說錯了、要收回」，那才需要理由。

---

# 推薦版位

工會可以把本工會的**商品**或**店面**放上商城首頁版位。

## 5. 推薦版位列表

```
GET union/featured-slots
```

需要 `featured_slot.manage` @ 本工會。**非分頁**，回傳全部版位。

### 回應 200

```json
{
  "success": true,
  "message": "取得推薦版位。",
  "data": [
    { "id": 5, "type": "product", "target_id": 3301, "position": 0, "is_currently_visible": true },
    { "id": 6, "type": "shop", "target_id": 21, "position": 1, "is_currently_visible": false }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `type` | `product` \| `shop` |
| `target_id` | 商品或店面的 ID |
| `position` | 排序，0 ～ 999，數字小的在前 |
| `is_currently_visible` | **這一格現在前台看得到嗎** |

### `is_currently_visible` 是這支端點的重點

**後台列表不過濾可購買性，前台才過濾。** 管理者必須看得到「這一格已經不能推薦了」，
才知道要換掉它；前台則永遠只顯示還買得到的（§12.3-18）。

`is_currently_visible` 為 `false` 的常見原因：

- 商品已下架、售完（可售庫存為 0）、或被違規下架。
- 商品所屬店面已停權、關閉或尚未開業。
- 店面本身已不在 `open` 狀態。

> **`false` 是暫時狀態，系統會自己收掉。** 平台有一個每小時執行的清理排程，
> 會把已失效的版位**自動移出**（`spec §12.3-18`），並以系統身分記錄移出理由 ——
> 「推薦對象已下架、售完或店面停業，自動移出版位」。
>
> 也就是說：`is_currently_visible: false` 的版位最多再存在一小時，之後就會從列表消失。
> 前端請標示成「已失效，將自動移除」，並提示管理者補上替代項目 ——
> 不要顯示成「暫時隱藏，補貨後會回來」，因為它不會回來，補貨後要重新加入。

---

## 6. 加入推薦版位

```
POST union/featured-slots
```

需要 `featured_slot.manage` @ 本工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `type` | string | ✅ | `product` \| `shop` |
| `id` | integer | ✅ | 商品或店面的 ID，≥ 1 |
| `position` | integer | — | 0 ～ 999，預設 0 |

```json
{ "type": "product", "id": 3301, "position": 2 }
```

> ⚠️ **請求欄位叫 `id`，回應欄位叫 `target_id`**，兩者是同一個東西。這不是筆誤，
> 送 `target_id` 會得到 422（`id` 為必填）。

### 回應 201

```json
{
  "success": true,
  "message": "已加入推薦版位。",
  "data": { "id": 7, "type": "product", "target_id": 3301, "position": 2 },
  "errors": null,
  "meta": null
}
```

回應**不含** `is_currently_visible`（剛加入的必然可見 —— 不可購買的根本加不進來）。

### 四道檢查，全部是 422

| 情況 | message |
| --- | --- |
| 推薦別的工會的商品／店面 | `只能推薦本工會的商品，不得推薦其他工會的內容 (spec §2.3)。` |
| 商品不可購買 | `只能推薦已上架且尚有可售庫存的商品，且其店面須營業中 (spec §U-04)。` |
| 店面不可購買 | `只能推薦已核准且營業中的店面 (spec §U-04)。` |
| 重複加入同一項目 | `此項目已在推薦版位中 (spec §U-04)。` |
| 超過版位上限 | `推薦版位已達上限 N 個，請先移除其他項目 (spec §U-04)。` |

**版位上限由平台的系統參數 `featured_slot_limit` 決定（預設 12），工會不得自行擴充。**
上限值沒有工會端的查詢端點 —— 前端請以 422 訊息中的數字為準，或由平台端提供。

> 「可購買性」與前台商品頁用的是**同一個判定**：不可能出現「推薦版位點進去卻買不到」。

---

## 7. 調整排序

```
PATCH union/featured-slots/{featuredSlot}
```

需要 `featured_slot.manage` @ 該版位所屬工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `position` | integer | ✅ | 0 ～ 999 |

### 回應 200

```json
{
  "success": true,
  "message": "排序已更新。",
  "data": { "id": 7, "position": 0 },
  "errors": null,
  "meta": null
}
```

> 回應**只有 `id` 與 `position`**，不是完整的版位物件。前端要更新畫面時請自行合併，
> 或重新呼叫列表。
>
> `position` **不保證唯一**：兩格給同一個數字不會報錯，此時它們之間的先後順序未定義。
> 要嚴格控制順序，請前端在拖曳後把受影響的每一格都送一次。

---

## 8. 移出推薦版位

```
DELETE union/featured-slots/{featuredSlot}
```

需要 `featured_slot.manage` @ 該版位所屬工會。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

回應 200，`data` 為 `null`，`message`：`已移出推薦版位。`

同樣是**帶 body 的 DELETE**，理由會寫入稽核。

> 商品售完或店面停權時**不必**手動移出：每小時的清理排程會自動處理，
> 且系統移出時會自己填理由。手動移出是用在「還買得到，但不想再推薦它了」。
>
> 被自動移出後想恢復推薦，要等該商品／店面重新符合可購買條件，再走第 6 條重新加入 ——
> 版位紀錄不會復活。
