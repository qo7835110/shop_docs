# 營運告警

spec §12.1、§15.2。付款、退款、出貨、撥款、對帳等作業失敗時，系統會主動寫入告警；本頁的兩支端點讓平台人員列出告警，並標記已處理。告警依類別分流（業主 2026-10-15 決策）：

| 權限 | 可檢視與處理的類別 | 預設角色 |
| --- | --- | --- |
| `operation_alert.all` | 全部（`payment`、`refund`、`shipment`、`payout`、`settlement`、`scheduling`、`risk`、`catalog`、`membership_sync`） | `platform_admin` |
| `operation_alert.finance` | `payment`、`refund`、`payout`、`settlement` | `platform_finance` |
| `operation_alert.fulfillment` | `shipment` | `platform_cs` |

會員 token 回 401；沒有上述任何權限，或篩選、標記的類別超出自己的範圍，回 403；告警不存在回 404。

## 列出告警

```http
GET /api/platform/operation-alerts?operation=shipment&unacknowledged=1&per_page=20
```

| 欄位 | 必填 | 說明 |
| --- | --- | --- |
| `operation` | — | 只列此類別；須是上表類別之一（否則 422），且在自己的範圍內（否則 403） |
| `unacknowledged` | — | `1` 只列尚未處理的 |
| `per_page` / `page` | — | 分頁，`per_page` 上限 100 |

不帶 `operation` 時，只列出自己範圍內的類別。依建立時間新到舊排序。`data` 每筆：

| 欄位 | 說明 |
| --- | --- |
| `id` | 告警編號 |
| `operation` | 類別 |
| `severity` | `warning`／`critical` |
| `source` | 來源排程或服務，例如 `payuni.logistics.adoption` |
| `summary` | 摘要 |
| `context` | 查核用的識別碼（如 `order_sub_id`、`payment_no`）。**不含 `error` 原文**：例外訊息可能夾帶 SQL 參數，原文只留在主機日誌 |
| `reference_type` / `reference_id` | 關聯物件，可為 null |
| `created_at` | 建立時間 |
| `acknowledged_at` / `acknowledged_by` | 處理時間與處理者（平台人員編號）；未處理為 null |

## 標記已處理

```http
PATCH /api/platform/operation-alerts/{operationAlert}/acknowledge
```

沒有 body。成功回 200，`data` 形狀同列表單筆，`message` 為「營運告警已標記為已處理。」並寫入稽核（`operation_alert.acknowledged`）。

**冪等**：已處理的告警再送一次也回 200，但原處理者與時間不變，也不再新增稽核。標記只代表「已有人接手」，不會改變付款、訂單、出貨或撥款的任何狀態；實際處理照各自流程進行（例如 PAYUNi 物流例外見 [營運手冊](../../ops/payuni-logistics-operations.md)）。
