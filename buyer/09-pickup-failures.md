# 09 未取貨紀錄

對應 spec §14.7、§14.7.2、§14.7.3。

## 這是記在**你**身上的紀錄，不是賣家的違規

超商取貨逾期未取會退回賣家，並在你名下留一筆未取貨紀錄。累計達上限時
**暫停使用超商取貨**，其他配送方式不受影響。

**「買家未取貨不是賣家的違規」**（§14.7.2）—— 這條紀錄不會讓賣家被記點，
反過來也一樣：賣家的違規不會影響你的取貨資格。

累計期間是 `pickup_failure_window_days`（**日曆天**，預設 180），
上限是 `pickup_failure_limit`（預設 3 次）。

---

## 1. 我的未取貨紀錄

```
GET buyer/pickup-failures
```

**分頁**。

### 回應 200 —— `message` 就是狀態摘要

```json
{
  "success": true,
  "message": "目前累計 1 次未失效的未取貨紀錄；超商取貨可正常選用。",
  "data": [
    {
      "id": 1,
      "order_sub_no": "M2026082400016760-01",
      "occurred_at": "2026-08-24T10:34:47+00:00",
      "expires_at": "2027-02-20T10:34:47+00:00",
      "status": "active",
      "counts_towards_limit": true,
      "appeal_status": "none",
      "appeal_reason": null,
      "appeal_deadline_at": "2027-09-07T23:59:59+08:00",
      "can_appeal": true,
      "pending_days": null,
      "decision_reason": null,
      "decided_at": null
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 1, "last_page": 1 } }
}
```

**`message` 是動態算出來的**，兩種結尾：

- `……；超商取貨可正常選用。`
- `……；超商取貨暫時無法選用。`

前端請直接顯示這一句 —— 它同時回答了「我現在幾次」與「還能不能選超商取貨」，
不必自己數陣列長度（`status` 為 `revoked` / `expired` 的那些不算）。

| 欄位 | 說明 |
| --- | --- |
| `status` | `active` \| `revoked`（申訴成立而失效）\| `expired`（逾累計期間） |
| `counts_towards_limit` | **後端算出的衍生值**：這一筆現在算不算數。畫面請用它，不要自己以 `status` + `expires_at` 推導。 |
| `appeal_status` | `none` \| `pending` \| `upheld` \| `rejected` |
| `appeal_deadline_at` | **申訴那扇門開到什麼時候**。§14.7.3「處理方式比照 §P-07.3」，因此期限是 `violation_appeal_days`（預設 14 **日曆天**），自 `occurred_at` 起算 |
| `can_appeal` | **後端算出的衍生值**：這一刻還送不送得出申訴（生效中、尚未申訴過、期限未過）。畫面的按鈕請用它 |
| `pending_days` | 申訴已經等了幾天；非待裁定時為 `null` |
| `expires_at` | 這一筆失效的時間。到期後自動不再計入。 |

> ⚠️ **`appeal_deadline_at` 與 `expires_at` 不是同一個日期**：後者是這筆紀錄
> **失效**（`pickup_failure_window_days`，180 日曆天），前者是申訴截止（14 天）。
> 兩者差了半年，畫面上很容易把前者當成後者。

### 達上限時會在哪裡撞到

**結帳時**：選超商取貨會被擋成 422（§12.3 的未取貨上限）。
其他配送方式照常，所以那不是「不能買東西」，是「不能選那一種取貨方式」。

---

## 2. 提出申訴

```
POST buyer/pickup-failures/{pickupFailure}/appeal
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | **10 ～ 1000 字** |

回應 200，`data` 為該筆紀錄（`appeal_status` 為 `pending`），
`message`：`申訴已送出，由平台裁定；成立則該筆失效，超商取貨限制隨之解除 (spec §14.7.3)。`

### 什麼情況值得申訴

超商未通知到店、門市提前退回、包裹在店內遺失 —— 這些不是你的責任。
`reason` 至少 10 字是因為「我沒收到通知」這種一句話裁定不了，
請寫上簡訊紀錄、到店日期之類可查證的事實。

### 裁定結果

| `appeal_status` | 結果 |
| --- | --- |
| `pending` | 待平台裁定 |
| `upheld` | **成立**：該筆 `status` 轉 `revoked`、`counts_towards_limit` 變 `false`，累計**立即**下降；若原本已達上限，超商取貨限制隨之解除 |
| `rejected` | 不成立，該筆維持生效；`decision_reason` 是理由 |

**一筆紀錄只能申訴一次**，重複送出回 422。

**逾期不受理**：§14.7.3 的「處理方式比照 §P-07.3」包含那一節的三條規則 ——
一筆一次、**期限內提出**、申訴期間不中止已生效的效果。
逾期送出回 422 並指出屆滿日。截止時間請直接讀 `appeal_deadline_at`，
不要自己加天數（期限屆至固定為當日 23:59:59，而天數是可調的系統參數）。

> 這條路徑**完全不碰賣家的違規點數**（§14.7.2）—— 申訴成立不代表賣家有錯，
> 申訴不成立也不代表你有錯，它只決定這一筆算不算進你的取貨上限。
