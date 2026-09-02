# 01 總覽與查詢（唯讀）

對應 spec U-01、U-03、P-07.3、§2.3、§11.3。

本頁四個端點**全部唯讀**，且全部限本工會。

---

## 1. 工會商城總覽

```
GET union/overview
```

需要 `order.view` @ 本工會。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `from` | date | 統計區間起始 |
| `to` | date | 統計區間結束 |
| `organization_id` | integer | 指定組織。**省略時為操作者自己的工會**；指定別的工會會得到 403 |

**`from` / `to` 都省略時是「本月至今」，不是全期間。**
賣家報表、工會總覽、平台總覽接受同一組區間參數，同一區間下數字口徑一致（§11.3）。

### 回應 200

```json
{
  "success": true,
  "message": "取得工會商城總覽。",
  "data": {
    "organization": { "id": 3, "code": "TPEUNION", "name": "臺北市某某職業工會" },
    "shops": {
      "total": 17,
      "by_status": { "draft": 1, "pending_review": 2, "approved": 1, "open": 12, "suspended": 1 }
    },
    "products": {
      "total": 430,
      "by_status": { "under_review": 5, "approved": 12, "listed": 400, "delisted": 13 }
    },
    "sales": {
      "basis": "net = Σ payable − Σ processed refunds − Σ negative adjustments",
      "scope": { "type": "organization", "id": 3 },
      "period": { "from": "2026-08-01 00:00:00", "to": "2026-08-18 23:59:59" },
      "gross_payable": 284500,
      "refunded_amount": 6600,
      "negative_adjustment_amount": 0,
      "net_sales": 277900,
      "paid_order_count": 96
    },
    "pending_reviews": { "shop_applications": 2, "product_reviews": 5, "expired_featured_slots": 1 },
    "member_sync": {
      "has_ever_synced": false,
      "is_stale": false,
      "last_synced_at": null,
      "stale_after_hours": 48
    }
  },
  "errors": null,
  "meta": null
}
```

### 欄位說明

**計數類欄位是資料庫分組結果**：筆數為 0 的狀態不會出現在物件裡。
取值請用 `by_status[status] ?? 0`。

`sales` 與平台總覽用的是**同一個服務、同一條算式**：

```
net_sales = gross_payable − refunded_amount − negative_adjustment_amount
```

- `gross_payable`（應付總額）與 `net_sales`（淨銷售額）**必須並列揭露**，
  不得只顯示其一（§11.3）。付款成功後應付金額凍結，全額退款也不會讓 `gross_payable` 變小。
- 金額皆為整數 TWD。

`pending_reviews` 是工會的待辦：

- `shop_applications` —— 待審的店面申請數。
- `product_reviews` —— 待審商品數。**已轉入平台複審的商品不算工會的待辦**
  （球已經在平台那邊），所以這個數字不會包含 `pending_platform_review`。
- `expired_featured_slots` —— **推薦版位上已經顯示不出來的項目數**（§U-04）。

### `expired_featured_slots`：版面上有幾個空格

加入推薦版位時的四道檢查只管**加入的當下**，而商品會在加入**之後**下架或售完
—— 於是版位還在，前台卻顯示不出它。

沒有這個數字，管理者只有打開
[推薦版位頁面](03-storefront-content.md#5-推薦版位列表)才會發現版面破了，
而那是前台看得到的事。逐格的 `is_currently_visible` 在那一頁上。

> 這一格**只有工會總覽有**：推薦版位由工會管理，平台總覽沒有對應的概念。

`member_sync` 是**會務同步健康度，一律回傳**（不是只在異常時才出現 ——
一個只在出事時才存在的欄位，出事時沒有人認得它）。四個欄位的意義與
`is_stale = true` 時哪些動作會被暫停，見
[04 §13 會務同步異常時的行為](04-shop-and-member-management.md#13-會務同步異常時的行為u-053)。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 指定了別的工會 | `您只能檢視本工會的統計，不得跨工會存取 (spec §2.3)。` |
| 403 | 沒有 `order.view` | `您沒有檢視工會統計的授權 (spec §U-01)。` |
| 403 | 指定的是**平台預設組織** | `平台預設組織不具工會商城，無總覽可檢視 (spec §2.2.1)。` |
| 422 | 組織不存在 | `指定的組織不存在。` |

---

## 2. 本工會店面的訂單

```
GET union/orders
```

需要 `order.view` @ 本工會。**分頁**。**唯讀** —— 工會不能出貨、不能取消、不能退款。

範圍固定是**操作者自己的工會**，這支不接受 `organization_id`。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | 子訂單狀態，見下表 |
| `shop_id` | integer | 限定本工會的某一間店面 |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

子訂單狀態（§11.2 的完整列舉）：
`awaiting_payment`、`awaiting_shipment`、`shipped`、`in_transit`、`delivered`、
`completed`、`pending_pickup`、`pickup_returned`、`delivery_exception`、
`cancelled`、`refund_pending`、`refunded`。

> 「爭議中」不是子訂單狀態，那是**退款單**的子狀態（`disputed`）。

> `status` 在這支端點**只驗證為字串、不驗證是否為合法狀態**：打錯字不會得到 422，
> 而是得到一個空清單。前端請用下拉選單而不是自由輸入。

### 回應 200

```json
{
  "success": true,
  "message": "取得本工會店面的訂單。",
  "data": [
    {
      "order_no": "SO20260810000127",
      "status": "delivered",
      "goods_amount": 1280,
      "shipping_fee": 60,
      "discount_amount": 0,
      "payable_amount": 1340,
      "tax_amount": 63.8095,
      "invoice_tax_amount": 64,
      "shipping_method": "convenience_store",
      "reservation_expires_at": null,
      "ship_deadline_at": "2026-08-13 23:59:59",
      "pickup_deadline_at": "2026-08-19 23:59:59",
      "delivered_at": "2026-08-12 15:20:00",
      "auto_complete_at": "2026-08-19 15:20:00",
      "return_window_ends_at": "2026-08-19 15:20:00",
      "is_return_window_open": true,
      "completed_at": null,
      "first_completed_at": null,
      "shipment": {
        "carrier": "hct",
        "tracking_number": "1234567890",
        "status": "delivered",
        "label_created_at": "2026-08-11 09:30:00",
        "shipped_at": "2026-08-11 10:00:00",
        "delivered_at": "2026-08-12 15:20:00",
        "last_event_at": "2026-08-12 15:20:00",
        "correction_reason": null
      },
      "items": [
        {
          "product_id": 3301,
          "product_sku_id": 9001,
          "product_name": "手工皮革刀具組",
          "spec_name": "中型",
          "condition": "new",
          "unit_price": 1280,
          "price_type": "member",
          "price_basis_organization_id": 3,
          "quantity": 1,
          "subtotal": 1280
        }
      ],
      "shop": { "id": 21, "name": "職人手作坊" },
      "recipient": {
        "name": "王＊明",
        "phone": "09**-***-678",
        "address": "臺北市中正區＊＊＊",
        "pickup_store_name": "7-ELEVEN 城中門市"
      }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 96, "last_page": 5 } }
}
```

### 三個容易踩到的欄位

| 欄位 | 說明 |
| --- | --- |
| `recipient` | **收件資訊一律遮罩**。工會有查詢履約的職責，沒有查看買家個資明細的必要（§U-01、§2.3）。工會端**沒有**取得完整收件資訊的端點。 |
| `tax_amount` | 由商品小計反推的浮點數，**四位小數是權威值**（§13.4.1），**不參與應付加總**。`payable_amount` 已是含稅價，不要再加上去。 |
| `invoice_tax_amount` | **發票上要寫的整數元稅額**。捨入只由後端做一次 —— 開立人是賣家（§13.6），讓每個賣家自己捨入，同一張子訂單會出現不只一個「正確」的稅額。 |
| `first_completed_at` | **首次**完成時間。退貨結案會改寫 `completed_at`，這一欄不動（§11.1）—— 要算「這筆交易多久前完成的」用它。 |
| `discount_amount` | 第一階段**恆為 0**：優惠券、點數與購物金只保留欄位不啟用。畫面可以直接不顯示。 |

`auto_complete_at` 與 `return_window_ends_at` **並列且互不影響**：訂單自動完成之後，
買家依然可以在退貨期限前申請退貨。`is_return_window_open` 是後端算好的布林值，
只看時間、不看訂單是否已完成 —— 請直接用它，不要自己比對日期。

`items[].price_type` 說明這筆成交價是原價還是會員價，
`price_basis_organization_id` 則是「會員價憑哪一個組織的會籍成立」——
跨工會的會員價爭議要看這個欄位。

`shipment` 在尚未出貨時為 `null`；買家、賣家、工會、平台看到的是**同一張出貨單**。
出貨單裡 `label_created_at` 與 `shipped_at` **並列**：前者有值而後者為 `null`，
代表單號印出來了但商品還沒交運 —— 庫存也還沒扣（§11.4）。合成一個欄位就再也
表達不出那段期間。

---

## 3. 本工會會員清單

```
GET union/members
```

需要 `member.view` @ 本工會。**分頁**。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `organization_id` | integer | 指定組織。省略為自己的工會；指定別的工會得到 403 |
| `account_status` | string | `pending`（商城帳號尚未開通）\| `issued`（已開通）。省略則不篩選 |
| `page` / `per_page` | integer | 見共通約定 |

### 回應 200

```json
{
  "success": true,
  "message": "取得本工會會員清單。",
  "data": [
    {
      "id": 501,
      "member_number": "000031",
      "name": "王小明",
      "identity_card": "A12*****89",
      "membership_status": "active",
      "marketplace_status": "active",
      "is_tradable": true,
      "membership_paid": true,
      "organization": { "id": 3, "code": "TPEUNION", "name": "臺北市某某職業工會", "type": "union" },
      "can_open_shop": true,
      "eligible_for_member_price": true,
      "phone": "09**-***-678",
      "email": "w***@example.com",
      "account": "TPEUNION-000031",
      "has_marketplace_account": true
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 812, "last_page": 41 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `identity_card` / `phone` / `email` | **一律遮罩**。工會端**沒有**取用完整身分證號的端點 —— 那需要 `member.identity.reveal`，而 `union_admin` 預設不含此權限。 |
| `membership_status` | 會籍事實：`active` \| `suspended` \| `resigned`。**會務端同步而來，工會唯讀。** |
| `marketplace_status` | 商城處置：`active` \| `suspended`。這一個工會改得動（見 [04 §7](04-shop-and-member-management.md#7-會員商城側停權與恢復)）。 |
| `is_tradable` | 後端算出：兩個狀態**皆為 `active`** 才是 `true`。判斷這個人能不能下單一律用它，不要自己 AND。 |
| `membership_paid` | 會費是否已繳。**這會直接影響該會員能否開店**。 |
| `can_open_shop` / `eligible_for_member_price` | 由組織類型決定，後端算好回傳，前端不要自行推導。 |
| `account` | 登入帳號。**`null` 表示商城帳號尚未開通**，該會員登不進商城。不遮罩：這是登入識別，核發之後本來就要告訴會員。 |
| `has_marketplace_account` | 同一件事的布林形式。清單畫面用它就夠了。 |

> **沒有單一的 `status` 欄位。** 會員有兩個來源不同的狀態，合成一欄就答不出
> 「我要去哪裡申訴」——會籍處分循工會章程，商城處置循 §P-07.3。

> **同步跑完之後，這份清單裡多半是 `has_marketplace_account: false` 的人。**
> §X-01 同步寫的是會籍事實 —— 名冊上有這個人，不代表他要用商城。要開通就用
> `?account_status=pending` 找出來，逐筆走
> [04 §8 核發商城帳號](04-shop-and-member-management.md#8-核發商城帳號)。

> 這支是**唯讀**。要改會員資料、會費狀態或會籍狀態，都要由平台執行
> （`PATCH platform/members/{member}`）。工會發現資料有誤時的流程是通報平台，不是自行修改。
> 唯一的例外是**核發商城帳號**——那不是改資料，是開通，工會自己做得到（04 §8）。

---

## 4. 本工會店面的違規記錄（唯讀）

```
GET union/violations
```

需要 `violation.view` @ 本工會。**分頁**。

「工會管理者可檢視本工會店面的違規記錄與申訴進度，**不得裁定，也不得跨工會檢視**」（§P-07.3）。
因此這裡**只有 GET** —— 建立、撤銷與申訴裁定都在平台端，工會 token 打過去會被 401 拒絕。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `organization_id` | integer | 省略為自己的工會；指定別的工會得到 403 |
| `status` | string | `active` \| `revoked` \| `expired` |
| `shop_id` | integer | 限定本工會的某一間店面 |
| `page` / `per_page` | integer | 見共通約定 |

> **`shop_id` 不是跨越租戶邊界的側門**：組織條件永遠一起套用，
> 帶別的工會的店面編號會得到**空清單**，不是那間店的記錄，也不是 403 ——
> 租戶條件寫在查詢裡，而 403 等於告訴對方那個編號存在（§2.3）。

### 回應 200

```json
{
  "success": true,
  "message": "取得本工會店面的違規記錄。",
  "data": [
    {
      "id": 33,
      "shop": { "id": 21, "name": "職人手作坊", "organization_id": 3 },
      "type": "overdue_shipment",
      "type_name": "逾期未出貨",
      "source": "auto",
      "recorded_by_type": "system",
      "points": 3,
      "is_major": false,
      "reason": "訂單 SO20260801000045 逾 3 個工作天未出貨，已自動取消並全額退款",
      "evidence": [],
      "status": "active",
      "status_reason": null,
      "counts_towards_points": true,
      "expires_at": "2027-02-14T00:00:00+08:00",
      "created_at": "2026-08-18T09:00:00+08:00",
      "appeal": {
        "id": 12,
        "violation_record_id": 33,
        "reason": "當日物流罷工，已提供公告佐證",
        "evidence": ["appeals/12/notice.pdf"],
        "status": "pending",
        "decision_reason": null,
        "decided_at": null,
        "created_at": "2026-08-19T14:00:00+08:00"
      }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 7, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `source` | `auto`（系統依規則自動記錄）\| `manual`（**人工建立，平台或工會都可能**，§P-07.4） |
| `recorded_by_type` | 記錄者身分：`admin`（平台人員）\| `system`（平台的自動偵測）\| `member`（工會）。**`source` 分不出這件事** —— 平台與工會人工建立的都是 `manual`。 |
| `counts_towards_points` | 後端算出的衍生值：是否仍計入累計點數。畫面請用它，不要自行以 `status` + `expires_at` 推導。 |
| `appeal` | 申訴進度；沒有申訴時為 `null`。`status`：`pending` \| `upheld` \| `rejected`。 |
| `is_major` | 重大違規可不經點數累計直接停權。 |

### 工會端看不到的

- **處置等級與門檻**（`platform/shops/{shop}/disposition`）是平台端端點。
  工會端只看得到個別記錄與點數，看不到「這間店目前是警告還是限制上架」的彙總判定。
- **未取貨紀錄**完全不在這裡：那記在買家身上，且「買家未取貨不是賣家的違規」（§14.7.2）。
