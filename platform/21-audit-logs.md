# 21 稽核日誌

查閱全平台的稽核紀錄 —— **唯讀**（§P-03、§11.1）。稽核紀錄「只增，不改，不刪」，
因此只有這一支 GET，沒有任何寫入或刪除的路由。

需要 `audit.view`（預設只有 `platform_admin`）。**不隨 `order.view` 附帶**：
查得到訂單的客服與財務，不必看得到誰改了誰的權限或銀行帳戶。

## 1. 稽核紀錄列表

```
GET platform/audit-logs
```

**分頁**，新到舊（依寫入順序）。

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `action` | string | 最長 100，完全比對，如 `member.updated`、`shop.suspend`、`role.grant` |
| `actor_type` | string | `member`／`admin`／`system` |
| `actor_id` | integer | 操作者識別碼；system 操作沒有 |
| `auditable_type` | string | 最長 100，稽核對象種類，如 `member`、`shop`、`refund`、`payout` |
| `auditable_id` | integer | 稽核對象識別碼 |
| `created_from` | string | `Y-m-d`，寫入日（含）起 |
| `created_to` | string | `Y-m-d`，寫入日（含）止；須不早於 `created_from` |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

### 回應 200

```json
{
  "success": true,
  "message": "取得稽核紀錄。",
  "data": [
    {
      "id": 5012,
      "action": "member.updated",
      "actor": { "type": "admin", "id": 3 },
      "auditable": { "type": "member", "id": 31 },
      "old_value": { "phone": "0912***678", "email": "wa***@example.com", "membership_paid": true },
      "new_value": { "phone": "0987***321", "email": "ne***@example.com", "membership_paid": true },
      "reason": "會員來電更正聯絡方式",
      "ip": "203.0.113.10",
      "user_agent": "Mozilla/5.0 …",
      "created_at": "2026-10-18 10:21:07"
    },
    {
      "id": 5011,
      "action": "organization.updated",
      "actor": { "type": "admin", "id": 3 },
      "auditable": { "type": "organization", "id": 2 },
      "old_value": { "name": "臺北市某某職業工會", "tax_id": "12345678" },
      "new_value": { "name": "臺北市某某產業工會", "tax_id": "12345678" },
      "reason": "更名",
      "ip": "203.0.113.10",
      "user_agent": "Mozilla/5.0 …",
      "created_at": "2026-10-18 10:15:42"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 2, "last_page": 1 } }
}
```

### `old_value`／`new_value` 的遮罩

稽核**保存**完整的原值與新值，但「遮罩是輸出層的規則」（§2.3）：這裡輸出的一律是遮罩後的值，
巢狀物件同樣處理。要遮的鍵與去識別化共用同一份清單（`AnonymizationFieldMap::sensitiveKeys()`）：

| 鍵 | 輸出 |
| --- | --- |
| `phone`、`recipient_phone` | 前 4 後 3，如 `0912***678` |
| `email` | 帳號前 2 碼＋網域，如 `ne***@example.com` |
| `identity_card` | 前 3 後 3，如 `A12****789` |
| `shipping_address` | 保留到行政區 |
| `account_number`、`bank_account_number` | 只留末 4 碼 |
| `recipient_name`、`account_name` | 只留姓，如 `王○○` |
| `name` | **只在稽核對象是 `member` 或 `admin` 時**遮罩；組織、商品、分類改名照原值輸出 —— 那正是稽核要看出「改了什麼」的地方 |
| `postal_code`、`pickup_store_code`、`pickup_store_name`、`buyer_tax_id`、`invoice_title` | `***` |

未遮罩的敏感欄位須另經授權並留稽核（§P-03、§2.3），不從這支端點取得；
會員身分證號的受控讀取見 [03 組織與會籍](03-organizations-and-members.md)。

查閱本身**不寫稽核**：輸出已遮罩，不屬於「讀取未遮罩值」。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 401 | 未帶 token，或帶的是會員 token（認證層即拒絕） |
| 403 | 平台人員沒有 `audit.view`（客服、財務預設皆無） |
| 422 | `actor_type` 不在列舉內、日期格式錯誤、`created_to` 早於 `created_from` |
