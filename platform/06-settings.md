# 06 系統設定（參數、費率、運費、行事曆）

對應 spec P-04、§13.1、§13.2、§13.3、§13.3.1。

**費率、運費與時間門檻一律活在資料表，不寫在程式或環境變數裡**，
且在下單當下**快照到子訂單上**。因此：

> 改了設定**不會影響既有訂單**。既有訂單用的是下單那一刻的快照值。
> 前端不要提示「此變更將影響所有訂單」——那是錯的。

三者皆以**生效期間**管理：修改 = 新增一個生效時點較晚的版本，**不原地覆寫**。

---

# 系統參數

## 1. 取得當期參數

```
GET platform/system-parameters
```

需要 `system_parameter.manage`。**非分頁**，回傳所有參數的**當期值**。

### 回應 200

```json
{
  "success": true,
  "message": "取得系統參數當期值。",
  "data": [
    {
      "key": "ship_deadline_days",
      "value": 3,
      "value_type": "integer",
      "day_unit": "business_day",
      "effective_from": "2026-01-01 00:00:00",
      "description": "賣家出貨期限；逾時自動取消並全額退款",
      "scheduled_changes": [
        { "value": 5, "effective_from": "2026-09-01 00:00:00" }
      ]
    },
    {
      "key": "vat_rate",
      "value": 0.05,
      "value_type": "decimal",
      "day_unit": null,
      "effective_from": "2026-01-01 00:00:00",
      "description": "營業稅率，供稅額反推 (§13.4)",
      "scheduled_changes": []
    }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `value` | **已依 `value_type` 轉型**：`integer` 回數字、`decimal` 回小數、`boolean` 回 true/false、`string` 回字串。不用自己 parse。 |
| `day_unit` | `business_day`（工作天）\| `calendar_day`（日曆天）\| `null`（非天數參數）。**值帶 `_day` 後綴**，不是 `business` / `calendar`。 |
| `effective_from` | 這個值從何時起生效。 |
| `scheduled_changes` | **已排定但尚未生效**的版本，依生效時點由近而遠。沒有排定變更時是**空陣列，不是 `null`**——畫面要顯示的是一份清單。 |

#### 為什麼要有 `scheduled_changes`

帶未來 `effective_from` 的變更**不會**出現在當期值裡（那是對的），而它原本
也不出現在任何別的地方 —— 於是「我上週排的那個變更還在不在」沒有任何查得到的
地方，而排定變更是這個端點唯一的寫入方式。第二個人來看時會再排一次
（撞到「該生效時點已有版本」的 422），或者以為沒排到。

當期值仍然是當期值，排定中的另外列。兩者放在同一列而不是另開一支端點：
「這個參數現在是多少、什麼時候會變」是同一個問題的兩半。

### `day_unit` 為什麼重要（§13.3.1）

**工作天與日曆天不可互相比較，也不可互相換算顯示。**

- `business_day`（6 項）—— 賣家義務與平台作業：`ship_deadline_days`、
  `seller_return_response_days`、`seller_inspection_days`、`buyer_cancel_response_days`、
  `dispute_resolution_days`、`chargeback_response_days`。
- `calendar_day`（10 項）—— 買家權利：`return_window_days`、`delivery_confirm_days`、
  `buyer_return_ship_days`、`store_pickup_days`、`payout_hold_days`、
  `review_window_days`、`violation_window_days`、`pickup_failure_window_days`、
  `violation_appeal_days`、`settlement_cycle_days`。
- `null`（18 項）—— 其餘非天數參數（分鐘、小時、年、比率、布林、列舉、代碼）。

畫面上顯示「3 天」時請一併標示單位；要顯示實際到期日，**一律用訂單／申請上的
`*_at` 欄位**，不要自行加天數。

### 參數總表

| key | 型別 | 單位 | 預設 | 說明 |
| --- | --- | --- | --- | --- |
| `stock_reservation_minutes` | integer | — | 30 | 待付款階段的庫存預留時效 |
| `payment_deadline_minutes` | integer | — | 30 | 待付款逾期時限 |
| `ship_deadline_days` | integer | 工作天 | 3 | 賣家出貨期限 |
| `delivery_confirm_days` | integer | 日曆天 | 7 | 送達後自動確認收貨，兼鑑賞期 |
| `return_window_days` | integer | 日曆天 | 7 | 退貨申請期。**不得低於法定 7 日** |
| `seller_return_response_days` | integer | 工作天 | 3 | 賣家回應退貨申請；逾時視為同意 |
| `buyer_cancel_response_days` | integer | 工作天 | 2 | 取消申請回應期；逾時視為同意 |
| `buyer_return_ship_days` | integer | 日曆天 | 7 | 買家寄回並登錄單號期限 |
| `seller_inspection_days` | integer | 工作天 | 3 | 退貨驗收期限；逾時視為通過 |
| `dispute_resolution_days` | integer | 工作天 | 5 | 客服裁定退款爭議的期限；逾期由平台主管指派，**不自動裁定** |
| `payout_hold_days` | integer | 日曆天 | 7 | 訂單完成至可撥款的等待天數 |
| `payout_cycle` | string | — | `weekly` | `daily` \| `weekly` \| `biweekly` \| `monthly` |
| `payout_cycle_day` | integer | — | 3 | weekly 為星期幾（1=週一）、monthly 為每月第幾日 |
| `payout_minimum_amount` | integer | — | 0 | 撥款最低金額；未達門檻遞延 |
| `payout_retry_limit` | integer | — | 3 | 撥款暫時性失敗的自動重試次數 |
| `business_day_calendar` | string | — | `TW` | 工作天採用的行事曆代碼 |
| `review_window_days` | integer | 日曆天 | 30 | 完成後可評價的天數 |
| `review_image_limit` | integer | — | 5 | 單則評價圖片上限 |
| `violation_window_days` | integer | 日曆天 | 180 | 違規點數累計期間 |
| `violation_warn_points` | integer | — | 3 | 警告門檻 |
| `violation_restrict_points` | integer | — | 6 | 限制上架門檻 |
| `violation_suspend_points` | integer | — | 10 | 停權門檻 |
| `violation_appeal_days` | integer | 日曆天 | 14 | 違規申訴期限 |
| `store_pickup_days` | integer | 日曆天 | 7 | 超商保管期 |
| `pickup_failure_window_days` | integer | 日曆天 | 180 | 未取貨紀錄累計期間 |
| `pickup_failure_limit` | integer | — | 3 | 未取貨次數上限；達上限暫停超商取貨 |
| `chargeback_response_days` | integer | 工作天 | 10 | 拒付抗辯期限 |
| `personal_data_retention_years` | integer | — | 7 | 去識別化後交易紀錄保留年限 |
| `vat_rate` | decimal | — | 0.05 | 營業稅率（供稅額反推） |
| `favorite_notify_enabled` | boolean | — | true | 收藏商品的降價／補貨通知 |
| `featured_slot_limit` | integer | — | 12 | 單一工會推薦版位上限 |
| `account_activation_hours` | integer | — | 72 | 啟用連結有效時數 |
| `member_sync_stale_hours` | integer | — | 48 | 會務同步逾此時數即視為異常，暫停以會籍為條件的處置（§U-05.3） |
| `settlement_cycle_days` | integer | 日曆天 | 7 | 結算檔預期出檔間隔 |
| `data_export_retention_days` | integer | 日曆天 | 30 | 個資匯出檔的保留天數；逾期自動清理（§X-04.4） |
| `shop_image_max_kb` | integer | — | 5120 | 店面 Logo 與封面的單檔大小上限（KB）（§S-03.1） |
| `shop_document_max_kb` | integer | — | 5120 | 開店申請附件的單檔大小上限（KB）（§S-01） |
| `shop_announcement_limit` | integer | — | 3 | 單一店面同時可發布的公告數上限（§S-13） |
| `product_image_max_kb` | integer | — | 5120 | 商品圖片的單檔大小上限（KB）（§S-04） |
| `attachment_max_count` | integer | — | 5 | 單則訊息可夾帶的附件數上限（§X-03） |
| `attachment_max_kb` | integer | — | 5120 | 評價照片與對話附件的單檔大小上限（KB）（§13.7） |
| `return_photo_limit` | integer | — | 5 | 退貨申請與驗收各可上傳的照片張數上限（§B-03、§S-07） |
| `return_photo_max_kb` | integer | — | 5120 | 退貨佐證與驗收照片的單檔大小上限（KB）（§B-03） |

**共 43 項。** 最後八項是**上傳限制**，全部以 KB 或張數計，`day_unit` 恆為 `null`。
它們散見於各自的端點文件，但調整時看的是這張表 —— 不列在這裡，等於營運知道有這個
限制卻找不到在哪裡改。

`account_activation_hours` 與 `member_sync_stale_hours` **以小時計，
因此 `day_unit` 為 `null`**：前者是連結時效（安全性），後者是資料健康度，
兩者都不屬於 §13.3.1 的期限體系 —— 混進工作天判定會讓連結在連假後才過期。

`data_export_retention_days` 同樣不在 §13.3 的表上，補進來的依據有兩個：
下載端點的合約本來就寫著「檔案已過期或清理」是它的兩種 404 之一
（**設計上檔案有生命週期**），而匯出檔是一份完整的個資副本 ——
永久留存與 §X-04 整節的目的相反。由 `privacy:prune-exports` 每日執行。

---

## 2. 更新參數

```
PATCH platform/system-parameters/{key}
```

需要 `system_parameter.manage`。`{key}` 是參數代碼（字串，例如 `ship_deadline_days`）。

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `value` | any | ✅ | 新值。型別須符合該參數的 `value_type` |
| `effective_from` | date\|null | — | 生效時點。**省略為立即生效** |
| `reason` | string\|null | — | 最長 255，寫入稽核 |

```json
{ "value": 5, "effective_from": "2026-09-01 00:00:00", "reason": "配合物流旺季延長出貨期限" }
```

### 回應 200

`data` 為**新版本**的參數物件（形狀同清單單筆，含該 key 目前的
`scheduled_changes`），`message`：`系統參數已更新。`

> 這是新增版本，不是覆蓋。舊版本仍留在資料表裡，供既有訂單與稽核回溯。
> 若 `effective_from` 是未來時間，`GET` 當期值仍會回舊值，直到生效時點到達。

### 失敗

全部是 **422**（含 key 不存在，因為 key 來自請求而非資源路徑）：

| 情況 | 訊息重點 |
| --- | --- |
| key 不存在 | `未知的系統參數：xxx` |
| 值的型別不符、列舉型參數給了不允許的值 | 指名該參數 |
| `effective_from` 早於今日 | `系統參數的生效時點不得早於今日：變更只影響生效後成立的訂單` |
| 該生效時點已有版本 | `請改用其他生效時點` |
| 參數間的相依限制（例如 `return_window_days` 低於法定 7 日） | 指出違反的限制 |

> **不能回溯生效**：變更只影響生效後成立的訂單。前端的日期選擇器請把今日之前的日子關掉。

---

# 費率規則

`fee_rule` 決定金流手續費、平台服務費與工會分潤怎麼算（§13.1）。

## 3. 費率清單

```
GET platform/fee-rules
```

需要 `fee_rule.manage`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `fee_type` | string | `payment_fee` \| `platform_fee` \| `union_share` |
| `scope_type` | string | `platform` \| `organization` \| `category` \| `shop` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得費率規則清單。",
  "data": [
    {
      "id": 4,
      "scope_type": "platform",
      "scope_id": null,
      "is_current": true,
      "fee_type": "platform_fee",
      "calc_type": "rate",
      "rate": 0.03,
      "fixed_amount": null,
      "basis": "goods_amount",
      "rounding": "round",
      "refund_behavior": "proportional",
      "effective_from": "2026-01-01 00:00:00",
      "effective_to": null,
      "status": "active"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 6, "last_page": 1 } }
}
```

| 欄位 | 值 | 說明 |
| --- | --- | --- |
| `fee_type` | `payment_fee` / `platform_fee` / `union_share` | 金流手續費／平台服務費／工會分潤 |
| `calc_type` | `rate` / `fixed` / `rate_plus_fixed` | 費率、定額、費率加定額 |
| `basis` | `goods_amount` / `goods_plus_shipping` / `total_amount` | 計算基準 |
| `rounding` | `round` / `floor` / `ceil` | 進位方式 |
| `refund_behavior` | `proportional` / `non_refundable` | 退款時費用是否按比例退還 |
| `scope_id` | integer\|null | `platform` 範圍為 `null`；其餘為對應的組織／分類／店面 ID。⚠️ 資料表存的是 `0`（那一欄進了唯一鍵，而唯一鍵不約束 NULL），轉換發生在**輸出層** |
| `is_current` | **後端算出的衍生值**：此刻適用的是不是這一條。見下 |

#### `is_current` —— 清單同時列出三種版本

`status` 只回答「有沒有被停用」，而清單同時列出已過期、生效中與尚未生效的版本。
前端拿 `effective_from` / `effective_to` 自己比對，就是**第二份實作** ——
撥款計算已經有一份，而兩份會漂移。

判準與撥款計算**完全相同**（同層重疊時取 `effective_from` 最新者）：
清單標「生效中」而撥款算的是另一條，是最難查的一種帳目不符 ——
兩邊都沒錯，只是不同意。

---

## 4. 建立費率規則

```
POST platform/fee-rules
```

需要 `fee_rule.manage`。**修改費率 = 建立一個生效時點較晚的新版本**，沒有 PATCH 內容的端點。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `scope_type` | string | ✅ | `platform` \| `organization` \| `category` \| `shop` |
| `scope_id` | integer\|null | 視範圍 | `platform` 填 `null` |
| `fee_type` | string | ✅ | `payment_fee` \| `platform_fee` \| `union_share` |
| `calc_type` | string | ✅ | `rate` \| `fixed` \| `rate_plus_fixed` |
| `rate` | number\|null | `calc_type` 含 rate 時 | 0 ～ 1 |
| `fixed_amount` | integer\|null | `calc_type` 含 fixed 時 | ≥ 0，整數 TWD |
| `basis` | string | ✅ | `goods_amount` \| `goods_plus_shipping` \| `total_amount` |
| `rounding` | string | ✅ | `round` \| `floor` \| `ceil` |
| `refund_behavior` | string | ✅ | `proportional` \| `non_refundable` |
| `effective_from` | date | ✅ | 生效起始 |
| `effective_to` | date\|null | — | 生效結束；`null` 為無限期 |

```json
{
  "scope_type": "organization",
  "scope_id": 3,
  "fee_type": "union_share",
  "calc_type": "rate",
  "rate": 0.02,
  "basis": "goods_amount",
  "rounding": "floor",
  "refund_behavior": "proportional",
  "effective_from": "2026-09-01 00:00:00"
}
```

**回應 201**，形狀同清單單筆（`status` 為 `active`）。

---

## 5. 啟用／停用費率規則

```
PATCH platform/fee-rules/{feeRule}/status
```

需要 `fee_rule.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `active` \| `inactive` |
| `reason` | string | ✅ | 最長 255 |

回應 200，形狀同清單單筆。

---

# 運費規則

本組端點維護的是**平台預設值**（§13.2）。店面覆寫值由賣家中心維護，
且只能在平台已啟用的配送方式範圍內設定。

## 6. 平台運費設定

```
GET platform/shipping-rates
```

需要 `fee_rule.manage`。**非分頁**，只回傳 `scope_type = platform` 的設定
—— 工會層與店面層的覆寫不在這份清單裡。

### 金額往內覆寫，可用性往外交集 —— 兩者方向相反（§13.2、§13.2.2）

同一張 `shipping_rates` 上最容易寫錯的地方，因此兩件事由兩個服務分別回答：

| | 方向 | 規則 |
| --- | --- | --- |
| **金額** | 往內覆寫 | 店面 > 工會 > 平台，取**最內層有效**的那一份 |
| **可用性** | 往外交集 | 平台 → 工會 → 店面 → 商品限制的**交集** |

把「店面有一份 `enabled` 的費率」當成「這個方式可用」，就等於讓店面重新開啟了
工會或平台已停用的方式。**下層無法開啟上層已停用的方式**；交集為空時該店面商品
**不得結帳**（§12.3-62），不是用平台預設運費硬算一個出來。

### 回應 200

```json
{
  "success": true,
  "message": "取得平台運費設定。",
  "data": [
    {
      "id": 2,
      "scope_type": "platform",
      "scope_id": null,
      "shipping_method": "convenience_store",
      "calc_type": "flat",
      "base_fee": 60,
      "tiers": null,
      "free_threshold": 1000,
      "remote_surcharge": 0,
      "enabled": true,
      "effective_from": "2026-01-01 00:00:00",
      "effective_to": null
    }
  ],
  "errors": null,
  "meta": null
}
```

| `shipping_method` | 說明 |
| --- | --- |
| `home` | 宅配 |
| `convenience_store` | 超商取貨 |
| `mail` | 郵寄 |
| `union_pickup` | 工會自取（**零運費**） |
| `self_pickup` | 店面自取（**零運費**） |

---

## 7. 建立運費規則

```
POST platform/shipping-rates
```

需要 `fee_rule.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `scope_type` | string | ✅ | `platform` \| `organization` \| `shop` —— **三層** |
| `scope_id` | integer\|null | 視範圍 | `platform` 填 `null`；`organization` 填組織 ID、`shop` 填店面 ID |
| `shipping_method` | string | ✅ | 見上表 |
| `calc_type` | string | ✅ | `flat` \| `by_weight` \| `by_quantity` |
| `base_fee` | integer\|null | — | ≥ 0，預設 0 |
| `tiers` | array\|null | `calc_type` 非 flat 時 | 級距陣列 |
| `tiers[].up_to` | integer\|null | ✅（可為 null） | 級距上限（重量克數／件數）。**`null` 表示最後一級**（無上限） |
| `tiers[].fee` | integer | ✅ | ≥ 0 |
| `free_threshold` | integer\|null | — | 滿額免運門檻（≥ 1） |
| `remote_surcharge` | integer\|null | — | 偏遠加價，預設 0 |
| `enabled` | boolean | — | 預設 `true` |
| `effective_from` | date | ✅ | — |
| `effective_to` | date\|null | — | — |

```json
{
  "scope_type": "platform",
  "shipping_method": "home",
  "calc_type": "by_weight",
  "base_fee": 0,
  "tiers": [
    { "up_to": 1000, "fee": 80 },
    { "up_to": 5000, "fee": 120 },
    { "up_to": null, "fee": 180 }
  ],
  "free_threshold": 1500,
  "remote_surcharge": 40,
  "effective_from": "2026-09-01 00:00:00"
}
```

**回應 201**，形狀同清單單筆。

---

## 8. 啟用／停用運費規則

```
PATCH platform/shipping-rates/{shippingRate}/status
```

需要 `fee_rule.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `enabled` | boolean | ✅ | — |
| `reason` | string | ✅ | 最長 255 |

回應 200，形狀同清單單筆。

> 停用某配送方式後，**賣家的店面覆寫值也會失去依據** —— 該方式在結帳時不再可選。

---

# 年度行事曆

工作天判定需要假日資料（§13.3.1）。**必須在每年度開始前匯入次年度假日**，
否則工作天判定會暫停 —— 出貨期限、驗收期限這些賣家義務都算不出來。

## 9. 已匯入的年度

```
GET platform/business-days
```

需要 `system_parameter.manage`。**非分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `calendar_code` | string | 行事曆代碼，`^[A-Z]{2,10}$`。省略為預設值 `TW` |

### 回應 200

```json
{
  "success": true,
  "message": "取得已匯入的行事曆年度。",
  "data": {
    "calendar_code": "TW",
    "imported_years": [
      { "calendar_code": "TW", "year": 2026, "day_count": 18, "imported_at": "2025-11-20 09:30:00" }
    ],
    "next_year": 2027,
    "next_year_imported": false
  },
  "errors": null,
  "meta": null
}
```

> **這個端點的用途就是讓營運看得出「次年度還沒匯入」**，因此結論由後端算好：
> `next_year_imported` 為 `false` 時直接顯示警示。
>
> 讓畫面自己拿今年是哪一年去陣列裡找，等於把「哪一年算次年度」這條規則
> 複製一份到前端 —— 而少了次年度的資料，**六項賣家義務**的期限全部算不出來
> （出貨、退貨回應、驗收、取消回應、爭議裁定、拒付抗辯）。
> 等到排程暫停判定才發現就晚了。

---

## 10. 匯入年度行事曆

```
POST platform/business-days/import
```

需要 `system_parameter.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `calendar_code` | string | ✅ | `^[A-Z]{2,10}$` |
| `year` | integer | ✅ | 2000 ～ 2100 |
| `days` | array | ✅（可為空陣列） | 日期清單 |
| `days[].date` | string | ✅ | `YYYY-MM-DD` |
| `days[].is_business_day` | boolean | ✅ | 是否為工作天 |
| `days[].note` | string\|null | — | 最長 100，例如「國慶日」 |
| `reason` | string\|null | — | 最長 255 |

```json
{
  "calendar_code": "TW",
  "year": 2027,
  "days": [
    { "date": "2027-01-01", "is_business_day": false, "note": "開國紀念日" },
    { "date": "2027-02-19", "is_business_day": false, "note": "農曆除夕" },
    { "date": "2027-02-27", "is_business_day": true, "note": "補班" }
  ],
  "reason": "匯入 2027 年度政府行事曆"
}
```

上例只有三筆：`2027-01-01` 是假日、`2027-02-19` 是農曆假日、`2027-02-27` 是
**週六補班**。其餘日子不必送。

### 回應 201

```json
{
  "success": true,
  "message": "年度行事曆已匯入。",
  "data": {
    "calendar_code": "TW",
    "year": 2027,
    "day_count": 3,
    "imported_at": "2026-11-18 10:00:00"
  },
  "errors": null,
  "meta": null
}
```

`day_count` 是該年度**已登錄的例外日筆數**，不是 365。它的用途是確認這次匯入的
筆數與送出的一致；要檢查「次年度是否已匯入」看的是第 9 條有沒有那一年。

> **只需列出「與週末預設不同的日子」。** 判定邏輯是：該日期有登錄就用登錄值，
> 沒登錄就以「週一至週五為工作天」推導。因此要送的是國定假日、調整放假日與
> **補班日**（補班日 `is_business_day` 為 `true`），不是 365 天全列。
>
> **重複匯入同一年度是「整年替換」，不是合併。** 送出的 `days` 會取代該年度原有的
> 全部例外日 —— 只補送一天假日，其餘例外日會被洗掉，那些日子就退回週末預設。
> 要改一天，請把該年度的例外日整組重送。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | `days` 中有日期不屬於 `year`（混入其他年度會讓那一年被誤判為已匯入） |
| 422 | 日期格式非 `YYYY-MM-DD`、`year` 超出 2000～2100、`calendar_code` 格式錯誤 |
