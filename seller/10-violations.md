# 10 違規與申訴

對應 spec P-07、P-07.1、P-07.2、P-07.3、P-07.4。

## 你看得到，但改不了

賣家端只有三支，全部是**查詢加一次申訴**：記錄由平台或工會建立，撤銷與裁定
一律由平台執行（§P-07.3）。這一章沒有任何寫入自己違規記錄的路。

## 兩個累計值，門檻各看一個

| 欄位 | 涵蓋來源 | 用於 |
| --- | --- | --- |
| `points` | **全部來源**（平台 ＋ 自動偵測 ＋ 工會） | 警告、限制上架兩級門檻 |
| `platform_points` | **`recorded_by_type` 不是 `member` 的那些** | **停權門檻** |

分界是**誰的判斷**，不是人還是機器：平台的自動偵測（`system`，例如逾期未出貨）
計入停權門檻；只有工會建立的（`member`）不計 —— 工會無權發動停權，
它的記點不該間接推動那個處置（§P-07.2）。

因此你可能看到「累計 9 點卻只是限制上架」——差額來自工會的記錄。

---

> **舉證是複合的**（§11.1、§P-07.1）：
>
> | 欄位 | 內容 | 為什麼分開 |
> | --- | --- | --- |
> | `evidence` | 站內物件的**參照**（`order:…`、`conversation:…`、`review:…`），以及自動來源寫入的**判定脈絡**（如取消率的 `{cancelled, total}`） | 那些本來就查得到、也不會被竄改。強制上傳等於要人把站內畫面截圖再傳回來 |
> | `attachments[]` | 上傳的**檔案**（截圖、掃描件），`jpg` / `jpeg` / `png`，單檔最大 `attachment_max_kb`（預設 5 MB），最多 `attachment_max_count` 個 | 站外的事實只能靠上傳保全，且必須驗大小與格式（§13.7） |
>
> 兩者**不可互相取代**，回應裡並列。帶檔案時請求為 `multipart/form-data`。

## 1. 本店面的違規記錄

```
GET seller/violations
```

**分頁**。不需參數：範圍就是你自己的店面。

> **沒有店面時回的是空陣列加一句說明**（`您目前沒有店面，因此沒有違規記錄。`），
> 不是 403 也不是 404。

### 回應 200

```json
{
  "success": true,
  "message": "取得本店面的違規記錄。",
  "data": [
    {
      "id": 1,
      "shop": { "id": 1, "name": "示範開業店面", "organization_id": 2 },
      "type": "overdue_shipment",
      "type_name": "逾期未出貨",
      "source": "auto",
      "recorded_by_type": "system",
      "points": 3,
      "is_major": false,
      "reason": "訂單 M2026082400001388-01 逾 3 個工作天未出貨，已自動取消並全額退款",
      "evidence": [],
      "status": "active",
      "status_reason": null,
      "counts_towards_points": true,
      "expires_at": "2027-02-20T00:00:00+08:00",
      "appeal_deadline_at": "2026-09-07T23:59:59+08:00",
      "can_appeal": true,
      "created_at": "2026-08-24T05:01:15+08:00",
      "appeal": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

| 欄位 | 說明 |
| --- | --- |
| `source` | `auto`（依規則自動記錄）\| `manual`（人工建立，平台或工會都可能） |
| `recorded_by_type` | `admin`（平台人員）\| `system`（自動偵測）\| `member`（工會）。**`source` 分不出平台與工會** —— 兩者的人工記錄都是 `manual`。 |
| `counts_towards_points` | **後端算出的衍生值**：是否仍計入累計（撤銷、失效者為 `false`）。畫面請用它，不要自己以 `status` + `expires_at` 推導。 |
| `expires_at` | 點數失效時間。累計期間為 `violation_window_days`（預設 180 **日曆天**）。 |
| `appeal_deadline_at` | **申訴那扇門開到什麼時候**（`violation_appeal_days`，預設 14 日曆天，自建立日起算）。⚠️ 與 `expires_at` 差了半年，畫面上很容易把後者當成前者。 |
| `can_appeal` | **後端算出的衍生值**：這一刻還送不送得出申訴（期限未過、記錄生效中、且尚未申訴過）。總覽的 `pending_violation_appeal` 數的就是它為 `true` 的那些。 |
| `is_major` | 重大違規**不受點數門檻限制、直接停權**（§P-07.2）。 |
| `appeal` | 申訴進度；沒有申訴時為 `null`。 |

### 違規類型與點數

| `type` | 名稱 | 點數 | 來源 |
| --- | --- | --- | --- |
| `overdue_shipment` | 逾期未出貨 | 3 | 自動 |
| `cancellation_rate` | 取消率超標 | 2 | 自動 |
| `prohibited_product` | 商品違規下架 | 3 | 人工 |
| `review_coercion` | 要脅或利誘修改評價 | 5 | 人工 |
| `off_platform_solicitation` | 誘導站外交易 | 5 | 人工 |
| `return_shipping_shifted` | 要求買家分擔退貨運費 | 3 | 人工 |
| `misrepresented_goods` | 商品與描述不符 | 3 | 人工 |
| `major_misconduct` | 重大違規 | 10 | 人工（可直接停權） |

點數是**建立當下的快照**，事後平台調整類型的點數不會改寫既有記錄。

---

## 2. 處置現況

```
GET seller/violations/disposition
```

不需參數。

```json
{
  "success": true,
  "message": "取得本店面的處置現況。",
  "data": {
    "level": "none",
    "points": 0,
    "platform_points": 0,
    "thresholds": null,
    "decided_at": null
  },
  "errors": null,
  "meta": null
}
```

**從未有過處置時就是上面這個形狀**：`thresholds` 為 `null`、`decided_at` 為 `null`、
**沒有 `reason` 欄位**。前端要處理這個形狀，不要假設鍵一定存在。

有處置時：

```json
{
  "level": "restricted",
  "points": 8,
  "platform_points": 6,
  "thresholds": { "warn": 3, "restrict": 6, "suspend": 10 },
  "reason": "累計 8 點，達限制上架門檻",
  "decided_at": "2026-08-18T10:05:00+08:00"
}
```

### 四級處置（§P-07.2）

| `level` | 意義 |
| --- | --- |
| `none` | 無處置 |
| `warned` | 僅通知 |
| `restricted` | **不得新增或上架商品**；既有已上架商品維持販售 |
| `suspended` | 店面不可販售；**既有訂單照常履約** |

> **門檻不追溯**：`thresholds` 是**做出那次判定當下**的門檻值，不是現在的系統參數值。
> 平台事後調整門檻不會改寫歷史判定 —— 所以你的門檻可能與別家不同。

**沒有店面時這一支回 422**（`您目前沒有店面。`），與第 1 條的空陣列不同 ——
處置現況是一個必須有對象的問題。

---

## 3. 提出申訴

```
POST seller/violations/{violation}/appeal
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | **10～1000 字** |
| `evidence` | string[] | — | 站內**參照**，每項最長 255 |
| `attachments[]` | file[] | — | 上傳的檔案，見下方說明 |

**回應 201**，`data` 為申訴單，
`message`：`申訴已送出，由平台裁定；申訴期間已生效的處置不中止 (spec §P-07.3)。`

```json
{
  "id": 12,
  "violation_record_id": 1,
  "reason": "當日物流罷工，已提供公告佐證",
  "evidence": ["appeals/12/notice.pdf"],
  "status": "pending",
  "decision_reason": null,
  "decided_at": null,
  "created_at": "2026-08-24T14:00:00+08:00"
}
```

### 三件事請照實顯示

1. **申訴期間處置不中止**（§P-07.3）。被限制上架的店面在裁定前仍然上不了架 ——
   訊息裡那句話就是為了讓前端不必自己解釋，也不會寫成「申訴中，暫停處置」。
2. **一筆記錄只能申訴一次**。重複送出回 422。
3. **申訴期限是 `violation_appeal_days`（預設 14 **日曆天**），自違規記錄建立日起算**。
   逾期就送不出去了 —— **截止時間直接讀 `appeal_deadline_at`，不要自己加 14 天**：
   期限屆至固定為當日 23:59:59，天數是可調的系統參數，而日曆天與工作天不可互相
   換算（§13.3.1）。

### 裁定結果

| `status` | 意義 |
| --- | --- |
| `pending` | 待裁定 |
| `upheld` | **申訴成立**：違規記錄撤銷、點數扣回；若處置因此低於門檻，限制或停權**即時解除** |
| `rejected` | 不成立，記錄維持有效；`decision_reason` 是理由 |

裁定一律由**平台**執行，即使記錄是工會建立的 —— 記錄者兼裁定者的話，
申訴程序就不存在了（§P-07.4）。

> **平台主動撤銷不等於申訴成立。** 「須即時解除處置」只綁在申訴成立那條路上；
> 平台自行撤銷記錄時，停權要另外解除。
