# 02 地址簿與發票設定

對應 spec B-02、§11.5、§13.6、§13.6.2、§X-04。

## 這兩本都是**範本**，不是訂單資料

地址簿與發票設定只是讓你結帳時少打字。**下單當下這些值會被複製成快照**寫進子訂單
（§11.5），此後兩者各走各的：

- 改地址簿**不會**改變已成立訂單的收件資訊。
- 刪地址簿**不會**讓已成立的訂單失去收件人。

因此這兩本可以放心整理，不會動到歷史交易。

---

## 1. 地址簿

```
GET buyer/addresses
```

**非分頁**。

```json
{
  "success": true,
  "message": "取得地址簿。",
  "data": [
    {
      "id": 1,
      "recipient_name": "王小明",
      "recipient_phone": "0912345678",
      "postal_code": "10041",
      "address": "臺北市中正區重慶南路一段 122 號",
      "is_default": true
    }
  ],
  "errors": null,
  "meta": null
}
```

**本人看自己的資料不遮罩** —— 你要拿它來寄件。

---

## 2. 新增地址

```
POST buyer/addresses
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `recipient_name` | string | ✅ | 最長 60 |
| `recipient_phone` | string | ✅ | 最長 30 |
| `postal_code` | string | ✅ | 最長 10 |
| `address` | string | ✅ | 最長 255 |
| `is_default` | boolean | — | 預設 `false` |

**回應 201**，`data` 為該筆地址，`message`：`地址已新增。`

> **`is_default` 是互斥的**：設為 `true` 時後端會在同一交易內把其他地址的預設取消。
> 前端不必先送一次「取消舊的預設」——那反而會在兩次請求之間出現沒有預設地址的狀態。

---

## 3. 修改地址

```
PATCH buyer/addresses/{address}
```

欄位與第 2 條相同，**全部必填**（這是整筆覆寫，不是部分更新）。
回應 200，`message`：`地址已更新。`

別人的地址編號回 **404**，不是 403 —— 後者會洩漏那個編號存在（§2.3）。

---

## 4. 刪除地址

```
DELETE buyer/addresses/{address}
```

回應 200，`data` 為 `null`，`message`：`地址已刪除。`

**軟刪除**：已成立訂單的收件快照不受影響，而來源紀錄不斷鏈（§X-04）——
去識別化時才會真的清掉內容。

---

## 5. 發票設定

```
GET buyer/invoice-profiles
```

**非分頁**。

```json
{
  "id": 1,
  "invoice_type": "triplicate",
  "buyer_tax_id": "12345675",
  "invoice_title": "某某有限公司",
  "carrier": "/ABC1234",
  "donation_code": null,
  "is_default": true
}
```

---

## 6. 新增發票設定

```
POST buyer/invoice-profiles
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `invoice_type` | string | ✅ | `duplicate`（二聯式）\| `triplicate`（三聯式） |
| `buyer_tax_id` | string\|null | **`triplicate` 時必填** | 恰 8 碼 |
| `invoice_title` | string\|null | **`triplicate` 時必填** | 最長 120 |
| `carrier` | string\|null | — | 最長 64，載具號碼 |
| `donation_code` | string\|null | — | 最長 16，捐贈碼 |
| `is_default` | boolean | — | 預設 `false` |

**回應 201**，`message`：`發票設定已新增。`

### 三聯式要看店家開不開得了

**發票由賣家開立，不是平台**（§13.6）。每家店有自己的 `invoice_capability`：

| 值 | 意義 |
| --- | --- |
| `triplicate` | 三聯式與二聯式都開得了 |
| `duplicate_only` | 只開得了二聯式 |
| `none` | 開不了發票 |

這個值在**商品頁與結帳試算就會揭露**（§13.6.2）——
不會讓你付完款才發現拿不到三聯式。結帳時若某家店是 `none`，
你仍然買得到，只是那一段不帶發票資訊。

---

## 7. 修改發票設定

```
PATCH buyer/invoice-profiles/{invoiceProfile}
```

欄位同第 6 條，**全部依原規則必填**。回應 200，`message`：`發票設定已更新。`

`is_default` 同樣互斥。

---

## 8. 刪除發票設定

```
DELETE buyer/invoice-profiles/{invoiceProfile}
```

回應 200，`data` 為 `null`，`message`：`發票設定已刪除。`

已開立的發票不受影響 —— 那些資訊在下單當下就快照進子訂單了。
