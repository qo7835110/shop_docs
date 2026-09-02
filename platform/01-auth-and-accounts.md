# 01 認證與帳號權限

對應 spec §12.2、§15.1、§15.2、P-05、P-06。

平台身分與會員身分是**兩個彼此隔離的主體**：平台 token 打買賣家路由會被 401 拒絕，
反之亦然。平台後台**不提供自行註冊**，帳號由部署程序或既有管理者建立。

---

## 1. 登入

```
POST platform/auth/login
```

不需 token。

### 請求

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `account` | string | ✅ | 最長 32 |
| `password` | string | ✅ | 最長 100 |

```json
{ "account": "platform.admin", "password": "correct-horse-battery" }
```

### 回應 200

```json
{
  "success": true,
  "message": "登入成功。",
  "data": {
    "token": "12|xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "account": "platform.admin",
    "name": "平台管理者"
  },
  "errors": null,
  "meta": null
}
```

後續所有請求帶 `Authorization: Bearer {token}`。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 401 | 帳號不存在或密碼錯誤 | `帳號或密碼不正確。` |
| 401 | 帳號已停用 | `此帳號已停用。` |
| 429 | 同帳號連續失敗 5 次，鎖定 900 秒 | `登入嘗試次數過多，請於 N 秒後再試。` |
| 422 | 欄位缺漏 | `送出的資料未通過驗證。` |

> 帳號不存在與密碼錯誤回同一句訊息，前端不要嘗試區分。每次登入（成功與失敗）都會寫入
> `authentication_logs`，含 IP 與 User-Agent。

---

## 2. 登出

```
POST platform/auth/logout
```

無請求 body。註銷當前 token（其他裝置的 token 不受影響）。

### 回應 200

```json
{ "success": true, "message": "已登出。", "data": null, "errors": null, "meta": null }
```

---

## 3. 取得目前登入身分

```
GET platform/me
```

### 回應 200

```json
{
  "success": true,
  "message": "取得管理員資料。",
  "data": {
    "id": 1,
    "account": "platform.admin",
    "name": "平台管理者",
    "organization": null,
    "roles": ["platform_admin"],
    "permissions": ["audit.view", "category.manage", "member.manage", "organization.manage", "…"]
  },
  "errors": null,
  "meta": null
}
```

`organization` **恆為 `null`** —— 平台身分不隸屬任何組織（spec §2.2）。
前端不要用它來判斷租戶範圍。

### `roles` 與 `permissions` —— 畫選單用的，不是授權本身

| 欄位 | 內容 |
| --- | --- |
| `roles` | 目前生效的角色代碼（已排序） |
| `permissions` | 各角色權限的**聯集**（已排序、去重） |

**這兩欄不放寬任何閘門。** §15.3 的三道閘門仍然在每一條路由上判定；
這裡回的是「畫面該顯示什麼」。

沒有這兩欄時前端只有三條路，而三條都不成立：

1. 打 `GET platform/roles` 自己推導 —— 但那需要 `role.manage`，
   **客服與財務都沒有**，於是這條路只有管理者走得通；
2. 把全部選單畫出來，讓使用者一路撞 403；
3. 前端自己維護一份角色 → 權限對照表 —— 它會與後端漂移，
   而漂移的症狀是「按鈕在但按不動」。

授予與撤銷**即時生效**（§P-05），因此這兩欄不做快取；重新呼叫即為最新。

---

## 4. 權限清單

```
GET platform/permissions
```

需要 `role.manage`。回傳**全系統權限主檔**（非分頁，`meta` 為 `null`）。
權限是程式碼定義的固定清單，不可透過 API 新增。

### 回應 200

```json
{
  "success": true,
  "message": "取得權限清單。",
  "data": [
    { "code": "shop.manage", "name": "管理自有店面", "module": "shop" },
    { "code": "refund.process", "name": "執行退款", "module": "refund" }
  ],
  "errors": null,
  "meta": null
}
```

`module` 可直接拿來做畫面分群：`shop`、`product`、`inventory`、`order`、`refund`、
`review`、`message`、`member`、`union`、`payout`、`settlement`、`risk`、`platform`。

完整清單（39 項）：

| module | code |
| --- | --- |
| shop | `shop.manage`、`shop.review`、`shop.dispose` |
| product | `product.manage`、`product.review` |
| inventory | `inventory.manage` |
| order | `order.place`、`order.fulfill`、`order.view` |
| refund | `return.request`、`refund.dispute`、`refund.process` |
| review | `review.write`、`review.reply`、`review.moderate` |
| message | `message.moderate` |
| member | `member.view`、`member.manage`、`member.marketplace_dispose`、`member.identity.reveal` |
| union | `announcement.manage`、`featured_slot.manage` |
| payout | `payout.view`、`payout.execute` |
| settlement | `settlement.import`、`settlement.reconcile`、`chargeback.manage`、`adjustment.create`、`adjustment.approve` |
| risk | `violation.view`、`violation.record`、`violation.manage` |
| platform | `organization.manage`、`category.manage`、`fee_rule.manage`、`system_parameter.manage`、`role.manage`、`admin.manage`、`audit.view` |

### 三組看起來相似、但**刻意分開**的權限（§P-07.4、§U-05.2）

工會管理者拿得到左欄，拿不到右欄。合成一個權限的後果寫在括號裡：

| 工會可持有 | 平台專屬 | 合併的後果 |
| --- | --- | --- |
| `shop.dispose`（暫停旗下店面與解除暫停） | `violation.manage`（記點、限制上架、裁定申訴） | 工會就能裁定自己所記的違規申訴，而申訴一律由平台裁定 |
| `violation.record`（**建立**違規記錄） | `violation.manage`（**撤銷**與裁定） | 同上：記錄者兼裁定者，申訴程序就不存在了 |
| `member.marketplace_dispose`（改 `marketplace_status`） | `member.manage`（改會籍主檔） | 工會就能改寫姓名、會員編號與會籍，而那是會務端的事實 |

> **持有 `shop.dispose` 不等於解得掉任何一筆暫停**：能不能解除由 `suspended_by_type`
> 的層級決定（§11.2），權限與層級是兩道各自獨立的閘門。

---

## 5. 角色清單

```
GET platform/roles
```

需要 `role.manage`。非分頁。

### 回應 200

```json
{
  "success": true,
  "message": "取得角色清單。",
  "data": [
    {
      "id": 2,
      "code": "platform_finance",
      "name": "財務人員",
      "subject_type": "admin",
      "scope_type": "platform",
      "is_system": true,
      "permissions": ["adjustment.create", "chargeback.manage", "member.identity.reveal", "order.view", "payout.execute", "payout.view", "refund.process", "settlement.import", "settlement.reconcile"]
    }
  ],
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `id` | 角色的數字識別碼。**修改角色的兩支端點都以它為路徑參數**（`PATCH platform/roles/{role}`、`PUT platform/roles/{role}/permissions`）—— `code` 不能代用，路由已將 `role` 限定為數字。 |
| `subject_type` | `member`（會員側角色）或 `admin`（平台側角色）。決定這個角色可以授予誰。 |
| `scope_type` | `platform`、`organization`、`shop`。決定授予時需不需要 `scope_id`。 |
| `is_system` | 系統預設角色。可改權限，但這是動到職責分離的動作，須謹慎。 |
| `permissions` | 權限代碼陣列，已排序。 |

系統預設六個角色：`platform_admin`、`platform_finance`、`platform_cs`、
`union_admin`、`seller`、`buyer`。各自的預設權限見 [平台 API 總覽的職責分離表](README.md#職責分離三個預設角色能做什麼)。

---

## 6. 建立角色

```
POST platform/roles
```

需要 `role.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `code` | string | ✅ | 最長 40、`^[a-z][a-z0-9_]*$`、不可重複 |
| `name` | string | ✅ | 最長 50 |
| `subject_type` | string | ✅ | `member` \| `admin` |
| `scope_type` | string | ✅ | `platform` \| `organization` \| `shop` |
| `description` | string\|null | — | 最長 255 |

```json
{
  "code": "platform_auditor",
  "name": "稽核人員",
  "subject_type": "admin",
  "scope_type": "platform",
  "description": "唯讀查核"
}
```

**回應 201**，`data` 形狀同角色清單的單筆（新建角色的 `permissions` 為空陣列，
需另外呼叫第 8 條同步權限）。

---

## 7. 更新角色名稱／描述

```
PATCH platform/roles/{role}
```

需要 `role.manage`。`{role}` 為角色數字 ID。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | — | 最長 50 |
| `description` | string\|null | — | 最長 255 |

**只能改這兩個欄位**：`code`、`subject_type`、`scope_type` 一經建立不得變更 ——
既有授予紀錄都綁在上面。回應 200，形狀同單筆角色。

---

## 8. 同步角色權限

```
PUT platform/roles/{role}/permissions
```

需要 `role.manage`。**這是全量覆蓋，不是新增**：沒列在陣列裡的權限會被移除。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `permissions` | string[] | ✅（可為空陣列） | 每項最長 60；重複值會自動去除 |

```json
{ "permissions": ["order.view", "violation.view", "audit.view"] }
```

回應 200，形狀同單筆角色（`permissions` 為同步後的結果）。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 422 | 把 `refund.process` 或 `payout.execute` 給**客服類角色** | `客服類角色不得取得「refund.process」權限：客服只裁定爭議，退款與撥款由財務執行 (spec §15.3、§P-02)。` |
| 422 | 權限代碼不存在 | `權限項目不存在：…（權限由系統定義，不由使用者新增）。` |

第一項是 spec §15.3 的硬規則，**不可由設定放寬**。

---

## 9. 授予角色給會員

```
POST platform/members/{member}/roles
```

需要 `role.manage`。`{member}` 為會員數字 ID。

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `role_code` | string | ✅ | 角色代碼，須為 `subject_type = member` 的角色 |
| `scope_id` | integer\|null | 視 `scope_type` | `organization` 範圍填組織 ID、`shop` 範圍填店面 ID、`platform` 範圍填 `null` |

```json
{ "role_code": "union_admin", "scope_id": 3 }
```

### 回應 201

```json
{
  "success": true,
  "message": "角色已授予。",
  "data": { "id": 41, "role": "union_admin", "scope_type": "organization", "scope_id": 3 },
  "errors": null,
  "meta": null
}
```

回傳的 `id` 是**授予紀錄 ID**（`member_role`），撤銷時要用它，不是會員 ID。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 角色代碼不存在：`角色「xxx」不存在。` |
| 422 | 角色的 `subject_type` 是 `admin`：`此角色屬於平台身分，不得授予會員 (spec §15.1)。` |
| 422 | `scope_id` 與角色的 `scope_type` 不相符（組織／店面不存在或未給） |
| 422 | 對**平台預設組織**的會員授予 `union_admin` 或 `seller` —— 該組織不承載店面，也沒有工會管理職能 |

---

## 10. 撤銷會員角色

```
DELETE platform/member-roles/{memberRole}
```

需要 `role.manage`。`{memberRole}` 是授予紀錄的 ID：授予時（第 9 條）回一次，
之後可從 `GET platform/members/{member}` 的 `roles[].id` 讀回來。

回應 200，`data` 為 `null`，`message` 為 `角色已撤銷。`。權限變更會寫入稽核
（舊值、新值、操作者、時間、IP）。

---

## 11. 授予角色給平台管理員

```
POST platform/admins/{admin}/roles
```

需要 `role.manage`。

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `role_code` | string | ✅ | 須為 `subject_type = admin` 的角色 |

平台角色一律是 `platform` 範圍，**不接受 `scope_id`**（帶了也會被忽略）。
授予會員身分的角色（`subject_type = member`）會回 422：
`此角色屬於會員身分，不得授予平台人員 (spec §15.1)。`

### 回應 201

```json
{
  "success": true,
  "message": "角色已授予。",
  "data": { "id": 7, "role": "platform_finance" },
  "errors": null,
  "meta": null
}
```

---

## 12. 撤銷平台管理員角色

```
DELETE platform/admin-roles/{adminRole}
```

需要 `role.manage`。`{adminRole}` 是第 11 條回傳的授予紀錄 ID。回應同第 10 條。

---

## 13. 管理員清單

```
GET platform/admins
```

需要 `admin.manage`。**分頁**（支援 `page`、`per_page`）。

### 回應 200

```json
{
  "success": true,
  "message": "取得管理員清單。",
  "data": [
    {
      "id": 1,
      "account": "platform.admin",
      "name": "平台管理者",
      "status": "active",
      "last_login_at": "2026-08-18 09:12:44",
      "roles": [
        { "id": 12, "code": "platform_admin", "name": "平台管理者", "granted_at": "2026-08-18 09:00:00" }
      ]
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 4, "last_page": 1 } }
}
```

`status`：`active` \| `disabled`。

`roles` 是**目前生效**的角色授予（撤銷過的不列 —— 那是歷史，屬於稽核）。

`roles[].id` 是**授予紀錄的 ID，不是角色的 ID** —— 撤銷角色（第 12 條）打的正是它。
這一欄原本沒有，於是那個識別碼只在授予當下的回應裡出現過一次：權限管理頁重新整理
之後，畫面上的撤銷鈕就沒有東西可帶。
帳號盤點要回答的是「誰拿得到 `refund.process`」，而這裡**沒有管理員詳情端點**
可以點進去補問，因此角色必須出現在清單上。回代碼不回權限：權限屬於角色，
由第 5 條給一次，在每一列重複展開會讓一頁回應變成幾百個字串。

---

## 14. 建立管理員帳號

```
POST platform/admins
```

需要 `admin.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `account` | string | ✅ | 最長 32、**全系統唯一** |
| `name` | string | ✅ | 最長 50 |
| `password` | string | ✅ | **最少 12 碼**、最長 100 |
| `email` | string\|null | — | Email 格式、最長 255 |

```json
{
  "account": "finance.wang",
  "name": "王財務",
  "password": "S3cure-Passphrase!",
  "email": "finance@example.org"
}
```

**回應 201**，形狀同管理員清單的單筆。新帳號**沒有任何角色**，
需另外呼叫第 11 條授予。

---

## 15. 停用管理員帳號

```
PATCH platform/admins/{admin}/disable
```

需要 `admin.manage`。**沒有刪除端點**：稽核紀錄以管理員 ID 歸屬，刪除會讓歷史操作失去操作者。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 255 |

```json
{ "reason": "離職，2026-08-31 生效" }
```

回應 200，形狀同管理員清單的單筆（`status` 為 `disabled`）。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 停用自己 | `不得停用自己的帳號，以免系統失去最後一位管理者 (spec §P-06)。` |

停用後該帳號的既有 token 立即失效，登入時回 401 `此帳號已停用。`。
**沒有恢復端點** —— 需要復職請建立新帳號。
