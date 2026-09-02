# 08 風控與違規

對應 spec P-07、P-07.1、P-07.2、P-07.3、§14.7.2、§14.7.3。

## 兩條不可違背的原則

1. **處置一律以記錄為依據**，不得憑印象停權；每一次處置都必須可回溯到具體的違規記錄。
   因此**沒有「直接調整處置等級」的端點** —— 等級只能是記錄變動的結果。
2. **買家未取貨不是賣家的違規**（§14.7.2）。未取貨紀錄記在**買家**身上，
   與店面違規點數是兩套完全獨立的機制，畫面上不要混在一起。

停權與恢復的執行在[店面管理](04-shops.md#7-停權)（`platform/shops/{shop}/suspend|resume`）——
那是處置的**執行**，且重大違規本來就可不經點數累計直接停權。

---

## 違規類型與點數

| `type` | 名稱 | 點數 | 來源 | 重大 |
| --- | --- | --- | --- | --- |
| `overdue_shipment` | 逾期未出貨 | 3 | 自動 | — |
| `cancellation_rate` | 取消率超標 | 2 | 自動 | — |
| `prohibited_product` | 商品違規下架 | 3 | 人工 | — |
| `review_coercion` | 要脅或利誘修改評價 | 5 | 人工 | — |
| `off_platform_solicitation` | 誘導站外交易 | 5 | 人工 | — |
| `return_shipping_shifted` | 要求買家分擔退貨運費 | 3 | 人工 | — |
| `misrepresented_goods` | 商品與描述不符 | 3 | 人工 | — |
| `major_misconduct` | 重大違規 | 10 | 人工 | ✅ 可直接停權 |

點數在 `violation_window_days`（預設 180 日曆天）內累計，逾期自動失效。
門檻（預設）：警告 3 點、限制上架 6 點、停權 10 點 —— 實際值以[系統參數](06-settings.md)為準。

### 兩個累計值，門檻各看一個（§P-07.2、§12.3-84）

工會也可以建立違規記錄（`POST union/violations`，`source = manual`，限本工會店面），
但**工會記的點不計入停權門檻**：

| 欄位 | 涵蓋來源 | 用於 |
| --- | --- | --- |
| `points` | **全部來源**（平台 ＋ 工會） | 警告、限制上架兩級門檻 |
| `platform_points` | **僅平台人員記錄者** | **停權門檻** |

處置回應一律並列兩個數字。只讀 `points` 的畫面會答不出「累計 10 點為什麼不是停權」
—— 而答案是那幾點來自工會，工會無權發動停權，情節重大時應暫停店面並轉平台（§P-07.4）。

工會也不得標記 `is_major`：重大違規不受點數門檻限制、直接停權，開放給工會就等於
給了一條繞過該分工的路。

---

> **舉證是複合的**（§11.1、§P-07.1）：
>
> | 欄位 | 內容 | 為什麼分開 |
> | --- | --- | --- |
> | `evidence` | 站內物件的**參照**（`order:…`、`conversation:…`、`review:…`），以及自動來源寫入的**判定脈絡**（如取消率的 `{cancelled, total}`） | 那些本來就查得到、也不會被竄改。強制上傳等於要人把站內畫面截圖再傳回來 |
> | `attachments[]` | 上傳的**檔案**（截圖、掃描件），`jpg` / `jpeg` / `png`，單檔最大 `attachment_max_kb`（預設 5 MB），最多 `attachment_max_count` 個 | 站外的事實只能靠上傳保全，且必須驗大小與格式（§13.7） |
>
> 兩者**不可互相取代**，回應裡並列。帶檔案時請求為 `multipart/form-data`。

## 1. 違規記錄列表

```
GET platform/violations
```

需要 `violation.view`（客服也有）。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `shop_id` | integer | 限定店面 |
| `organization_id` | integer | 限定組織 |
| `status` | string | `active` \| `revoked` \| `expired` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得違規記錄。",
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
      "appeal_deadline_at": "2026-09-01T23:59:59+08:00",
      "can_appeal": true,
      "created_at": "2026-08-18T09:00:00+08:00",
      "appeal": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 7, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `source` | `auto`（依規則自動記錄）\| `manual`（**人工建立，平台或工會都可能**，§P-07.4） |
| `recorded_by_type` | 記錄者身分：`admin`（平台人員）\| `system`（自動偵測）\| `member`（工會）。**`source` 分不出這件事** —— 平台與工會的人工記錄都是 `manual`。 |
| `counts_towards_points` | **後端算出的衍生值**：是否仍計入累計點數（撤銷、失效者為 `false`）。畫面請用它，不要自行以 `status` + `expires_at` 推導。 |
| `appeal_deadline_at` | **申訴那扇門開到什麼時候**（`violation_appeal_days`，預設 14 日曆天，自建立日起算）。⚠️ 與 `expires_at` 不是同一個日期：後者是**點數失效**（180 日曆天），兩者差了半年。 |
| `can_appeal` | **後端算出的衍生值**：這一刻還送不送得出申訴（期限未過、記錄生效中、且尚未申訴過）。 |
| `appeal` | 申訴物件；沒有申訴時為 `null`。 |

> **期限由後端給，不要拿 `created_at` 加 14 天。** 期限屆至固定為當日 23:59:59，
> 天數是可調的系統參數，而日曆天與工作天不可互相換算（§13.3.1）——
> 前端自己加的那個日期，只要參數被調過就是錯的。

---

## 2. 建立違規記錄

```
POST platform/violations
```

需要 `violation.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `shop_id` | integer | ✅ | 須存在 |
| `type` | string | ✅ | 須為既有違規類型代碼 |
| `reason` | string | ✅ | 5 ～ 500 字 |
| `evidence` | string[] | — | 站內**參照**，每項最長 255 |
| `attachments[]` | file[] | — | 上傳的檔案，見下方說明 |

```json
{
  "shop_id": 21,
  "type": "off_platform_solicitation",
  "reason": "對話紀錄顯示賣家提供私人匯款帳號要求站外交易",
  "evidence": ["conversations/8812/message-33.png"]
}
```

### 回應 201

```json
{
  "success": true,
  "message": "違規記錄已建立，處置等級已重新判定。",
  "data": {
    "record": { "…": "同違規記錄單筆" },
    "disposition": {
      "level": "restricted",
      "points": 8,
      "platform_points": 8,
      "thresholds": { "warn": 3, "restrict": 6, "suspend": 10 },
      "reason": "累計 8 點，達限制上架門檻",
      "decided_at": "2026-08-18T10:05:00+08:00"
    }
  },
  "errors": null,
  "meta": null
}
```

`level`：`none` \| `warned` \| `restricted` \| `suspended`。
`points` 與 `platform_points` **一律並列輸出**，門檻各看一個（見上）。

> 建立記錄會**立即重新判定處置等級**，但**不會自動執行停權** ——
> 停權是另一個明確的動作（見[店面管理](04-shops.md#7-停權)）。

---

## 3. 撤銷違規記錄

```
POST platform/violations/{violation}/revoke
```

需要 `violation.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 5 ～ 500 字 |

回應 200，`data` 同第 2 條（`record` + `disposition`），
`message`：`違規記錄已撤銷，點數已扣回並重新判定處置等級。`

> **撤銷不解除停權。** 平台主動撤銷與申訴成立是兩件事：§P-07.3 的「須即時解除」
> 只綁在申訴成立那條路上。要恢復營業請另外呼叫 `platform/shops/{shop}/resume`。

---

## 4. 違規申訴列表

```
GET platform/violation-appeals
```

需要 `violation.view`。**分頁**。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `status` | string | `pending` \| `upheld` \| `rejected` |

### 回應 200

```json
{
  "success": true,
  "message": "取得違規申訴。",
  "data": [
    {
      "id": 12,
      "violation_record_id": 33,
      "reason": "當日物流罷工，已提供公告佐證",
      "evidence": ["appeals/12/notice.pdf"],
      "status": "pending",
      "decision_reason": null,
      "decided_at": null,
      "created_at": "2026-08-19T14:00:00+08:00",
      "pending_days": 6,
      "record": { "…": "對應的違規記錄" }
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 3, "last_page": 1 } }
}
```

申訴期限為 `violation_appeal_days`（預設 14 **日曆天**），自違規記錄建立日起算
—— 那是**賣家那一側**的期限。

### 佇列的順序：待裁定的最前面，最舊的先做

spec **沒有**給裁定側的期限參數（賣家側有 `violation_appeal_days`，平台側沒有
對應的一項），因此這裡不發明一個 —— 那會讓參數目錄與 §13.3 不一致。

但佇列仍然需要「先做哪一件」的依據：申訴期間**處置照舊生效**（§P-07.3），
等愈久代價愈高。因此：

- **排序**：`pending` 一律排在最前面，其中最舊的先做；已裁定的依 id 由新到舊。
- **`pending_days`**：這一件已經等了幾天。已裁定者為 `null`。

要不要另立一個裁定期限，是營運政策的決定，不是這份文件的。

---

## 5. 申訴裁定

```
POST platform/violation-appeals/{appeal}/decision
```

需要 `violation.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `upheld` | boolean | ✅ | `true` 申訴成立、`false` 不成立 |
| `reason` | string | ✅ | 5 ～ 500 字 |

### 回應 200

```json
{
  "success": true,
  "message": "申訴成立：違規記錄已撤銷，點數已扣回；處置若因此低於門檻已即時解除。",
  "data": {
    "appeal": { "…": "申訴單筆" },
    "disposition": { "level": "warned", "points": 3, "platform_points": 3, "thresholds": { "warn": 3, "restrict": 6, "suspend": 10 }, "reason": "…", "decided_at": "…" }
  },
  "errors": null,
  "meta": null
}
```

| `upheld` | 結果 |
| --- | --- |
| `true` | 違規記錄撤銷、點數扣回；**若處置因此低於門檻，限制或停權即時解除** |
| `false` | `message` 為 `申訴不成立：違規記錄維持生效。`，記錄維持有效 |

---

## 6. 店面處置現況

```
GET platform/shops/{shop}/disposition
```

需要 `violation.view`。單一店面的處置現況與依據。

### 回應 200

```json
{
  "success": true,
  "message": "取得店面處置現況。",
  "data": {
    "disposition": {
      "level": "restricted",
      "points": 8,
      "platform_points": 6,
      "thresholds": { "warn": 3, "restrict": 6, "suspend": 10 },
      "reason": "累計 8 點，達限制上架門檻",
      "decided_at": "2026-08-18T10:05:00+08:00"
    },
    "restricts_listing": true,
    "active_records": [ { "…": "仍計入點數的違規記錄" } ]
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `disposition` | 最新一筆處置判定。**從未有過處置時為 `{ "level": "none", "points": 0, "platform_points": 0, "thresholds": null, "decided_at": null }`**（沒有 `reason` 欄位），前端要處理這個形狀。 |
| `disposition.platform_points` | **僅平台來源**的累計點數，停權門檻只看它（見上）。 |
| `restricts_listing` | 是否處於限制上架狀態。 |
| `active_records` | **仍計入點數**的違規記錄，即 `disposition.points` 的組成依據（含工會來源）。點數加總對不上時就是這張清單有問題。 |

**兩個累計值都能由 `active_records` 重算**，依據是 `recorded_by_type`：

```
points          = Σ active_records[].points
platform_points = Σ active_records[] where recorded_by_type !== "member"
```

分界是**誰的判斷**，不是人還是機器：`system`（平台的自動偵測，例如逾期未出貨）
計入停權門檻，只有 `member`（工會）不計。把 `system` 也排除掉的話，
逾期未出貨將永遠不可能導致停權 —— 而那正是 §P-07 要處置的核心行為之一。

> **門檻不追溯**：`thresholds` 是**做出那次判定當下**的門檻值，不是現在的系統參數值。
> 事後調整門檻不會改寫歷史判定。

---

# 未取貨申訴

## 7. 待裁定的未取貨申訴

```
GET platform/pickup-failure-appeals
```

需要 `violation.manage`。**分頁**。

**只列待裁定的** —— 這裡是待辦清單，不是查詢介面。買家完整的未取貨史屬於買家自己，
平台沒有可以把全站買家紀錄拉出來的端點。

### 回應 200

```json
{
  "success": true,
  "message": "取得待裁定的未取貨申訴。",
  "data": [
    {
      "id": 44,
      "order_sub_no": "SO20260801000045",
      "occurred_at": "2026-08-09T18:00:00+08:00",
      "expires_at": "2027-02-05T18:00:00+08:00",
      "status": "active",
      "counts_towards_limit": true,
      "appeal_status": "pending",
      "appeal_reason": "超商未通知到店，簡訊紀錄可證",
      "decision_reason": null,
      "decided_at": null,
      "pending_days": 3,
      "member_id": 501
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `status` | `active` \| `revoked` \| `expired` |
| `appeal_status` | `none` \| `pending` \| `upheld` \| `rejected` |
| `counts_towards_limit` | 是否計入 `pickup_failure_limit`（預設 3 次，達上限暫停使用超商取貨）。 |
| `pending_days` | 這一件申訴已經等了幾天。**佇列依它排序：最舊的先做** —— 等的期間那一筆仍然計入上限，而達上限的買家選不了超商取貨。與違規申訴佇列同一種排序（spec 兩邊都沒有給裁定側的期限參數）。 |

累計期間為 `pickup_failure_window_days`（預設 180 **日曆天**）。

---

## 8. 未取貨申訴裁定

```
POST platform/pickup-failures/{pickupFailure}/decision
```

需要 `violation.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `upheld` | boolean | ✅ | `true` 成立、`false` 不成立 |
| `reason` | string | ✅ | 5 ～ 500 字 |

回應 200，`data` 為該筆未取貨紀錄。

| `upheld` | `message` |
| --- | --- |
| `true` | `申訴成立：該筆未取貨紀錄已失效，累計次數立即減少。` |
| `false` | `申訴不成立：該筆未取貨紀錄維持生效。` |

> 申訴成立會讓買家的累計次數**立即**下降；若原本已達上限而被暫停超商取貨，該限制隨之解除。
> 這條路徑**完全不碰賣家的違規點數**。
