# 03 組織與會籍

對應 spec P-01、§2.2、§2.2.1、§2.3、X-05。

組織是資料主線的最上層，**只有平台碰得到**。會籍（`member`）掛在組織下，
也是登入主體；每個組織最多一個 `member`，帳號由該組織逐筆核發。
`user` 是不含認證資訊與 PII 的自然人連結層：加入第二個工會會多一筆會籍，
並由第二個工會核發**另一組登入帳號**。

**這兩類資料都沒有 DELETE 端點**：店面、訂單、撥款、評價、稽核全都引用它們的 ID，
刪除等於讓歷史交易失去歸屬。停用取代刪除；個資刪除權走去識別化（見 [09 個資](09-moderation-and-privacy.md)）。

---

# 組織

## 1. 組織清單

```
GET platform/organizations
```

需要 `organization.manage`。**分頁**。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `type` | string | `union`（工會）\| `organization`（團體）\| `corporation`（法人）\| `platform`（平台預設組織） |
| `status` | string | `active` \| `suspended` \| `closed` |
| `keyword` | string | 最長 50，比對代碼、名稱與**統一編號** |
| `page` / `per_page` | integer | 見[共通約定](../conventions.md#分頁) |

> 這裡的清單**包含平台預設組織**，與前台的工會商城（一律排除它）刻意相反：
> 營運後台必須看得到它存在、狀態是否正常、承接了多少無工會會籍的買家。

### 回應 200

```json
{
  "success": true,
  "message": "取得組織清單。",
  "data": [
    {
      "id": 3,
      "code": "TPEUNION",
      "name": "臺北市某某職業工會",
      "type": "union",
      "tax_id": "12345675",
      "status": "active",
      "status_reason": null,
      "can_host_shops": true,
      "grants_member_benefits": true,
      "allows_new_transactions": true,
      "allows_member_login": true,
      "member_count": 812,
      "shop_count": 17,
      "pending_shop_review_count": 2,
      "created_at": "2026-01-05T09:00:00+08:00"
    }
  ],
  "errors": null,
  "meta": { "pagination": { "current_page": 1, "per_page": 20, "total": 12, "last_page": 1 } }
}
```

### 衍生欄位（由後端算出，前端不要自行以 `type` 判斷）

| 欄位 | 意義 |
| --- | --- |
| `can_host_shops` | 是否可承載店面。**平台預設組織恆為 `false`** —— 它不能開店、沒有工會管理者、沒有商城。 |
| `grants_member_benefits` | 該組織會員是否享有會員價／會員限定商品資格。 |
| `allows_new_transactions` | `suspended` 與 `closed` 皆為 `false`：旗下所有店面不得新交易。既有訂單仍須履行。 |
| `allows_member_login` | **停權與關閉的唯一差別**：只有 `closed` 會讓該組織的會籍登不進來。停權仍可登入，只是不能有新交易。 |

`member_count`／`shop_count`／`pending_shop_review_count` 在清單與詳情都會回傳；
若查詢未帶計數（極少數情境）則為 `0`。

---

## 2. 組織詳情

```
GET platform/organizations/{organization}
```

需要 `organization.manage`。`data` 形狀同清單單筆（計數欄位保證有值），
**外加 `pending_actions`**。

### `pending_actions` —— 這家組織可以開始用了嗎

```json
{
  "pending_actions": [
    "尚無會員。可由「代行建立會籍」新增第一位會員。",
    "尚未指派工會管理者，本工會的店面申請將無人可審核。"
  ]
}
```

一句可直接顯示的中文，**空陣列表示沒事要做**；形狀與開店申請的
`pending_actions` 刻意相同。**只有詳情才有這一欄**，清單沒有 ——
一頁二十列各算一次角色查詢並不值得。

只列**擋住下一步**的事，不列建議（清單一長，看的人就開始略過它）：

| 情形 | 為什麼擋住下一步 |
| --- | --- |
| 組織非 `active` | 旗下店面不得有新交易；此時不再列其餘各項 |
| 旗下沒有任何會員 | 沒有人開得了店 |
| 沒有生效中的工會管理者 | §U-02 的店面審核只有工會管理者做得了，旗下的申請會**永遠停在待審**，而沒有任何地方會亮紅燈 |

`type = platform` 的預設組織恆為空陣列：它不能開店、沒有工會管理者、
沒有商城（§2.2.1）。

---

## 3. 建立組織

```
POST platform/organizations
```

需要 `organization.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `code` | string | ✅ | 最長 20、`^[A-Z][A-Z0-9]{1,19}$`（大寫英數，首字為英文） |
| `name` | string | ✅ | 最長 100 |
| `type` | string | ✅ | `union` \| `organization` \| `corporation`。**不可建立 `platform`** —— 平台預設組織全系統唯一，由部署程序建立 |
| `tax_id` | string | ✅ | 台灣統一編號，含檢核碼驗證 |

```json
{ "code": "NTPCUNION", "name": "新北市某某職業工會", "type": "union", "tax_id": "12345675" }
```

**回應 201**，形狀同清單單筆。

---

## 4. 更新組織資料

```
PATCH platform/organizations/{organization}
```

需要 `organization.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | — | 最長 100 |
| `tax_id` | string | — | 統編格式（含檢核碼） |
| `reason` | string | — | 最長 200，會寫入稽核 |

> `code` 與 `type` **不可變更**。統編「一經建立不得由工會端自行變更，更正須由平台執行並留稽核」（§2.2.1）——
> 所以更正統編時請務必帶 `reason`。

回應 200，形狀同清單單筆。

---

## 5. 變更組織狀態

```
PATCH platform/organizations/{organization}/status
```

需要 `organization.manage`。狀態變更**另立端點**，因為它必須帶原因，
且會讓旗下所有店面不得新交易 —— 不該和改名走同一條路。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `active` \| `suspended` \| `closed` |
| `reason` | string | ✅ | 最長 200 |

```json
{ "status": "suspended", "reason": "統編查核異常，暫停新交易" }
```

### 回應 200

`message`：`組織狀態已更新；既有訂單的出貨、退貨與撥款不受影響。`
`data` 形狀同清單單筆。

**前端請把這句話顯示出來**：停權不是停止履約，既有訂單仍要出貨、退貨與撥款。

---

## 5.1 會務同步來源的綁定

```
GET    platform/organizations/{organization}/sync-binding
POST   platform/organizations/{organization}/sync-binding/preview
POST   platform/organizations/{organization}/sync-binding
DELETE platform/organizations/{organization}/sync-binding
```

需要 **`organization.sync_binding`**（不是 `organization.manage`）。

### 統編不是同步開關

在此之前，「這個組織要不要同步」的判準是「它的 `tax_id` 在會務端找得到」。
那讓一次**資料更正**變成一次來源切換：修正一個打錯的統編，下一輪排程就拿另一家
工會的名冊覆寫這個組織的全部會籍——沒有預覽、沒有確認，也沒有一筆說得出「是誰
決定接這個來源」的紀錄。

而且沒有回頭路：統編一旦對得上，同步就永遠繼續。要讓一個組織改回人工維護，
唯一的作法是把統編清掉或改壞——用一筆錯的主檔資料去關掉一個功能。

因此三件事各自獨立：

| 動作 | 端點 | 影響 |
| --- | --- | --- |
| 改統編 | `PATCH platform/organizations/{organization}` | 只改主檔。**不啟動、不切換同步** |
| 綁定／改綁 | `POST .../sync-binding` | 需預覽、理由與稽核；版本遞增 |
| 解除 | `DELETE .../sync-binding` | 停止後續同步，開放人工維護 |

> **已綁定的組織不得逕行修改統編**：回 422，訊息指引先解除或走改綁。
> 不是權限問題，所以不是 403——正確的動作不是去要權限。

### 執行必須綁在先前看過的那一份預覽上

`POST .../sync-binding/preview` 回傳 `binding_version`（目前的綁定版本，首次綁定為
`null`）、來源工會名稱、來源與商城兩邊的人數，以及 **`preview_token`**。
**執行時兩者都必須帶回來**：

```json
{
  "source_tax_id": "99000001",
  "preview_token": "…",
  "binding_version": null,
  "reason": "封測接入"
}
```

兩道檢查擋的是不同的事：

| 欄位 | 擋什麼 |
| --- | --- |
| `binding_version` | **併發**——期間有人改綁過。那份預覽算的是針對另一個來源的差異，拿它去套用等於用 A 工會的預覽核准對 B 工會的覆寫 |
| `preview_token` | **打錯字**——預覽看的是 A、執行送的是 B。經辦人預覽 99000001（看到三千人，數字對得上），執行時打成 99000002 而那一家剛好存在 |

`preview_token` 綁住組織、來源、綁定版本與時效（30 分鐘），四者任一不符回 422。
少了它，人數規模那道確認就只是畫面上的裝飾——後端無從得知經辦人看的是哪一家。

預覽**不輸出任何身分證**（§2.3）：要判斷的是「接對了嗎」，
而那靠工會名稱與人數規模，不需要看到任何一個人的號碼。

### 預覽給的是變化量，不是兩個總數

「來源 3000 人、商城 3000 人」是一個看起來很安穩的組合，而它完全可能是
「新增 3000、退會 3000」——接錯一家人數相近的工會，兩個總數依然對得上，
而按下執行之後整個工會的會籍被換成另一批人。因此預覽回傳的是差異：

| 欄位 | 意義 |
| --- | --- |
| `provisioning_count` | 名冊上有、商城沒有，**且會籍有效**——會被建檔。建檔仍可能逐筆失敗（身分證重複、會號衝突），因此這是上限 |
| `resigning_count` | 商城有、名冊上沒有——會被轉為退會 |
| `matching_count` | 兩邊都有——狀態會被以會務端為準覆寫 |
| `unmatchable_count` | **來源端**沒有會員編號的筆數。接不上也建不了，而它們讓每一輪同步都不算合格成功（§U-05.3），因此會連帶擋住這個工會的開店審核 |
| `warnings` | 空名冊、過半退會、完全對不上、有無法對接的資料——四種情況各一句 |

這四個數字**以正式同步的判準算出來**：預覽會逐批掃過來源名冊，套用與
`MembershipSyncService` 相同的建檔條件（會籍有效、有會員編號、商城尚無此人），
並以會員編號去重。以兩個總數相減估算會在名冊不乾淨時失真——而名冊不乾淨
正是綁定要防的事。

它們也一併進 `preview_token` 的簽章：操作者按下執行時同意的是他看過的那組數字，
名冊在這段期間變動過，那份預覽談的就是另一件事了。

### 綁定與改綁是同一支 POST

兩者只差在「之前有沒有綁過」，而那件事後端自己知道。分成兩支會讓前端得先查一次
狀態才知道要打哪一支，而那次查詢與執行之間狀態還可能再變。

改綁會解除舊綁定（寫 `released_at`，**不刪除**——那是唯一的操作證據）並遞增版本。
`membership_sync_runs` 記著自己屬於哪一版，因此**改綁前的成功批次不會被當成新來源
的健康證據**（§U-05.3）。

### `9000xxxx` 是測試 fixture 的保留區段，一律接不上

統編落在 **90000000–90009999** 的組織，綁定與預覽一律回 422，
`membership:sync` 也不會把它列為候選（`App\Support\Integration\FixtureTaxIds`、
plans/30 K）。同一個區段的統編也**不得作為來源**：兩個方向都擋。

理由是封測站上真實工會與驗收 fixture 共用同一個資料庫，而統編正是同步對接兩邊
組織的鍵。2026-09-03 那一輪，人工建立的封測工會被真實名冊覆寫，四個角色帳號
整批變成 `resigned`，驗收在中途失去身分。隔離因此不靠「測試不會撞到真實統編」，
而靠「撞到了也接不上」。

驗收與封測資料請一律建在這個區段內；它是**本專案的測試慣例**，
不是外部保證永不配發的號段。

### 解除不撤銷已經發生的事

被同步轉為 `resigned` 的會員**不會**因解除而回到 `active`：那些處置當時依據的是
會務端的事實，解除並不使那個事實變成假的。要改回來是人工維護的事，
而解除正是為了讓人工維護重新可用。

逐筆的 `members.synced_at` 同樣保留——清空等於偽造「從未同步過」，
而那個狀態有它自己的意義（§U-05.3）。

---

# 會籍

> **角色不在這份文件。** 會籍只回答「這個人屬於哪個組織」，不回答「他能做什麼」。
> 把 `union_admin`（工會管理者）、`seller` 等會員側角色授予／撤銷給某個會籍，
> 見 [01 認證與帳號權限 §9 授予角色給會員](01-auth-and-accounts.md#9-授予角色給會員)
> 與 [§10 撤銷會員角色](01-auth-and-accounts.md#10-撤銷會員角色)：
> `POST platform/members/{member}/roles`、`DELETE platform/member-roles/{memberRole}`。
> 工會範圍由 `scope_id` 指向組織 ID（§15.3 的第三道閘門），且**不得指向
> `type = platform` 的預設組織**（§2.2.1）。這兩支需要 `role.manage`，
> 與本文件的 `member.manage` 是兩種權限 —— 建立會籍的人不必然能給角色。

## 6. 會員清單（跨組織）

```
GET platform/members
```

需要 `member.manage`。**分頁**。

這是**跨組織**查詢 —— 與工會端（不得跨工會）恰好相反。

### 查詢參數

| 參數 | 型別 | 說明 |
| --- | --- | --- |
| `organization_id` | integer | 限定單一組織 |
| `membership_status` | string | `active` \| `suspended` \| `resigned`。**篩的是會籍狀態，參數名不是 `status`** —— 送 `status` 不會報錯，會靜默地不篩選 |
| `keyword` | string | 最長 50 |
| `page` / `per_page` | integer | 見共通約定 |

### 回應 200

```json
{
  "success": true,
  "message": "取得會員清單。",
  "data": [
    {
      "id": 501,
      "member_number": "TPE-2026-0031",
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

> **平台人員檢視他人資料一律是遮罩態**（§2.3）：身分證、電話、Email 都是遮罩值。
> 需要完整身分證號請走第 12 條，會留稽核。

`can_open_shop` 與 `eligible_for_member_price` 由所屬組織類型決定，後端算好回傳。

> **`account` 為 `null` 表示商城帳號尚未開通**，該會員登不進商城。這是 §X-01 同步
> 建出來的會籍的常態 —— 同步寫的是會籍事實，開通是所屬組織的另一個動作
> （[第 11 條](#11-核發商城帳號)）。`has_marketplace_account` 是同一件事的布林形式，
> 清單畫面用它就夠了。
>
> `account` 不遮罩：它是登入識別而不是個資，核發之後本來就要告訴會員。

> **會員沒有單一的 `status` 欄位。** 回應帶的是 `membership_status`（會籍事實）、
> `marketplace_status`（商城處置）與後端算出的 `is_tradable`（兩者皆 `active` 才為
> `true`）。要判斷這個人能不能下單、能不能開店，一律讀 `is_tradable`，
> 不要自己 AND 兩個狀態 —— 理由見[第 10 條](#10-變更會員狀態--兩個欄位兩支端點)。

---

## 7. 會員詳情

```
GET platform/members/{member}
```

需要 `member.manage`。比清單多三個欄位（`must_change_password`、`memberships`、`roles`）：

```json
{
  "success": true,
  "message": "取得會員資料。",
  "data": {
    "id": 501,
    "member_number": "TPE-2026-0031",
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
    "must_change_password": false,
    "memberships": [
      {
        "id": 902,
        "member_number": "NTPC-2026-0177",
        "name": "王小明",
        "identity_card": "A12*****89",
        "membership_status": "active",
        "marketplace_status": "active",
        "is_tradable": true,
        "membership_paid": false,
        "organization": { "id": 5, "code": "NTPCUNION", "name": "新北市某某職業工會", "type": "union" },
        "can_open_shop": true,
        "eligible_for_member_price": true,
        "phone": "09**-***-678",
        "email": "w***@example.com"
      }
    ],
    "roles": [
      {
        "id": 88,
        "code": "union_admin",
        "name": "工會管理者",
        "scope_type": "organization",
        "scope_id": 3,
        "granted_at": "2026-08-20 10:12:00"
      }
    ]
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `account` | 登入帳號。**Phase 1 唯一的登入識別碼**，全站唯一，由組織核發，一經核發即不可變更。`null` 表示商城帳號尚未開通（見[第 11 條](#11-核發商城帳號)）。 |
| `must_change_password` | 為 `true` 表示發過初始密碼但尚未變更。 |
| `memberships` | **同一自然人的其他會籍**（跨工會歸戶結果）。空陣列表示只有這一個會籍。每一筆各有自己的登入帳號（§X-05.1）—— 這是全系統唯一看得到「這幾組帳號是同一個人」的地方。 |
| `roles` | 這筆會籍**目前生效**的角色授予。`roles[].id` 是授予紀錄的 ID，撤銷角色（`DELETE platform/member-roles/{memberRole}`）打的正是它；`scope_type` / `scope_id` 一併給出，因為同一個角色代碼在不同 scope 是兩筆不同的授予。 |

---

## 8. 代為建立會籍

```
POST platform/members
```

需要 `member.manage`。這是 §X-01 會務同步的**手動入口**，不是第三種會籍來源 ——
`member_number` 仍由會務端給定，`source` 記為 `manual`。

> **建檔不開通帳號。** 這一支只寫會籍事實，回應的 `account` 必為 `null`，該會員
> 登不進商城。要讓他登入，接著打[第 11 條 核發商城帳號](#11-核發商城帳號)——
> 與 §X-01 同步建出來的會籍走完全同一支。
>
> （2026-09-02 之前這一支同時核發帳號與憑證，`credential_channel` 為必填。
> 拆開的理由見第 11 條。）

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `organization_id` | integer | ✅ | 須存在 |
| `name` | string | ✅ | 最長 50 |
| `member_number` | string | — | 最長 32。省略時由系統產生 |
| `identity_card` | string | ✅ | `^[A-Za-z][0-9]{9}$` |
| `phone` | string\|null | — | 最長 20 |
| `email` | string\|null | — | Email 格式、最長 255 |
| `membership_paid` | boolean | — | 預設 `false` |

```json
{
  "organization_id": 3,
  "name": "王小明",
  "member_number": "TPE-2026-0031",
  "identity_card": "A123456789",
  "phone": "0912345678",
  "email": "wang@example.com",
  "membership_paid": true
}
```

### 回應 201

**新自然人**（本平台第一次見到這個身分證）：

```json
{
  "success": true,
  "message": "會籍已建立；商城帳號尚未開通，請另行核發。",
  "data": {
    "member": { "…": "同會員清單單筆" },
    "account": null,
    "linked_existing_user": false
  },
  "errors": null,
  "meta": null
}
```

**比對到既有自然人**（此人已在別的工會有會籍）：

```json
{
  "success": true,
  "message": "會籍已建立，並已比對到本人在其他組織的既有會籍；商城帳號尚未開通，請另行核發。",
  "data": {
    "member": { "…": "…" },
    "account": null,
    "linked_existing_user": true
  },
  "errors": null,
  "meta": null
}
```

> **`linked_existing_user` 不影響帳密。** 帳號由所屬組織核發（§X-05.1），因此開通後
> 他拿到的是**第二組**帳密 —— 比對的作用只是把新會籍掛到同一列自然人，讓系統知道
> 「這幾筆會籍是同一個人」，供日後合併帳號時使用。
>
> 「這是第二組帳密，不能與其他工會那組互用」的提醒由系統在**核發那一刻**發給會員，
> 不是現在：此刻他還沒有帳號，通知寄出去他也登不進來看。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 組織不存在、該自然人在此組織已有會籍、統編／身分證格式錯誤 |

---

## 9. 更新會員資料

```
PATCH platform/members/{member}
```

需要 `member.manage`。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `phone` | string\|null | — | 最長 20 |
| `email` | string\|null | — | Email、最長 255 |
| `membership_paid` | boolean | — | 會費繳納狀態 |
| `reason` | string | — | 最長 200，寫入稽核 |

`name`、`identity_card`、`member_number`、`organization_id` **不可由此變更** ——
那些是會務端資料。回應 200，形狀同會員清單單筆。

> `membership_paid` 會直接影響開店資格（見[店面](04-shops.md#核准檢核清單)），改動請帶 `reason`。

---

## 10. 變更會員狀態 —— **兩個欄位，兩支端點**

會員有**兩個**互相獨立的狀態，來源與救濟程序完全不同（§2.2、§U-05.2）：

| 欄位 | 值域 | 來源 | 商城可否改寫 |
| --- | --- | --- | --- |
| `membership_status` | `active` \| `suspended` \| `resigned` | 會務職行家同步而來的**會籍事實** | 唯讀（本端點是代行會務同步，見下） |
| `marketplace_status` | `active` \| `suspended` | 工會或平台的**商城營運處置** | ✅ |

**可交易性 = 兩者皆為 `active`。**

> **為什麼是兩支端點而不是一支帶參數的**：共用一支就等於讓一個值決定要改哪一份
> 事實，而填錯的後果是營運處置被寫成會籍處分 —— 兩者的救濟程序完全不同。
> 工會端只拿得到後者（`PATCH union/members/{member}/marketplace-status`）。

### 10.1 變更會籍狀態

```
PATCH platform/members/{member}/membership-status
```

需要 `member.manage`。這支是**代行會務同步**，不是商城的處置手段。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `active` \| `suspended` \| `resigned` |
| `reason` | string | ✅ | 最長 200 |

`message`：`會籍狀態已更新；既有訂單仍會依約履行，且不自動退回平台預設組織 (spec §X-05.3)。`

**兩個常見誤解，前端請照 message 顯示**：

1. 停權／退會**不取消既有訂單**，履約義務不變。
2. 退會**不會**自動把人搬到平台預設組織 —— 那會讓歷史交易的租戶歸屬改變。

#### 已接會務同步的組織會回 422

該組織一旦接上 §X-01 的會務同步，這一欄改由同步全權維護且**一律覆寫**，
本端點即回 422，`errors` 為 `null`、理由在 `message`：

```
「示範已接會務同步工會」的會籍狀態由會務系統同步維護，商城端不得代為變更 (spec §X-05.2)：
此處的變更會在下一輪同步被覆寫。請於會務職行家辦理，商城端於同步後反映。
```

* **為什麼是拒絕而不是接受**：此時接受人工寫入，那次寫入會在下一輪同步被靜默
  還原 —— 在使用者眼中就是「我改了但它自己變回去」，比直接被拒絕難查得多。
* **為什麼是 422 不是 403**：這不是權限問題。403 會讓管理員去要權限，而正確的
  動作是去會務端改。
* **判準**：該組織有沒有任何一筆會籍被同步標記過（`members.synced_at`）。
* **適用範圍縮小，不是廢棄**：尚未接同步的組織（含平台預設組織）仍然只有這一條
  寫入路徑。
* **`marketplace-status` 不受影響**（見 10.2）：那是商城的營運處置，不是會籍事實。

### 10.2 變更商城側狀態

```
PATCH platform/members/{member}/marketplace-status
```

需要 `member.manage`。這是**商城的營運處置**，不動會籍主檔。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `status` | string | ✅ | `active` \| `suspended`（**沒有 `resigned`** —— 退會是會籍事實，不是處置） |
| `reason` | string | ✅ | 最長 200 |

回應的 `data` 同時帶出兩個欄位，`membership_status` 必然維持原值。
工會在商城側停權一位會員**不代表他被除名**，他在其他工會的會籍與店面不受影響
（§12.3-76、§12.3-83）。

兩支端點都會記錄原值、新值、操作者、時間與 IP（§2.3）。

### 10.3 帳號安全停用與解除

```
POST   platform/members/{member}/access-suspension
DELETE platform/members/{member}/access-suspension
```

需要 **`member.access_suspend`**，**平台專屬**——工會端沒有對應路由。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `reason` | string | ✅ | 最長 200 |

#### 這與 10.2 的停權是兩件事

會籍停權、退會與商城側停權限制的是**開始一筆新交易**：不得下單、不得送出開店
申請、不得上架新商品。它們**不擋登入**——被停權的買家可能還在退貨期限內，被停權
的賣家店裡可能還有已付款待出貨的訂單，而「既有訂單一律續行履約」（§P-07.2）。
把人擋在門外，這些義務就沒有人能履行，而買家的錢已經付了。

這一支處理的是另一種事實：**憑證外洩、盜用、詐欺調查**。它立即撤銷該會籍的全部
token，登入與售後入口一併關掉。

| | 10.2 商城側停權 | 10.3 安全停用 |
| --- | --- | --- |
| 誰可執行 | 工會或平台 | **僅平台** |
| 擋什麼 | 新交易 | 憑證本身 |
| 登入 | 允許 | 禁止，既有 token 立即失效 |
| 既有訂單 | 照常履約 | 由既有授權的營運流程接續 |
| 救濟 | 工會章程／§P-07.3 申訴 | 帳號安全流程 |

完整的權限矩陣見 spec §2.2.2。

#### 解除只還原一欄

`DELETE` 只清除 `access_suspended_at`，**不改寫** `membership_status` 或
`marketplace_status`。停用期間發生的會籍異動不因帳號解鎖而回捲——解鎖的人多半
沒有那兩份事實的決定權，順帶改寫等於用安全流程繞過會務與營運的職責分離。

重複停用（已停用者再打 `POST`）與重複解除皆回 422，訊息指出目前狀態。

### 安全停用回應欄位

平台會員清單、詳情、建檔／更新及安全處置回應的會員物件均提供下列欄位。
查詢與維護依原有 `member.manage` 授權，安全處置依 `member.access_suspend` 授權；
工會與會員本人回應不提供這些安全調查資訊。

| 回應欄位 | 型別 | 說明 |
| --- | --- | --- |
| `is_access_suspended` | boolean | 目前是否安全停用 |
| `access_suspended_at` | string 或 null | 停用時間，秒精度 |
| `access_suspended_reason` | string 或 null | 停用原因 |
| `access_suspended_by` | string 或 null | 操作者參照，例如 `admin#3`；完整軌跡仍以稽核為準 |

解除後 `is_access_suspended` 為 false，其餘三欄明確回 null；會籍及商城側狀態不變。


---

## 11. 核發商城帳號

```
POST platform/members/{member}/account
```

需要 **`member.account_issue`**（不是 `member.manage`）。工會端有[同名的一支](../union/04-shop-and-member-management.md#8-核發商城帳號)。

### 為什麼會籍與帳號是兩件事

§X-01 同步寫入的是**會籍事實**：名冊上有這個人，不代表他要用商城。因此同步（與
第 8 條的代建）只寫 `member`，`account` 留空 —— 一個五千人的工會不會因為同步跑過
就多出五千組沒有人要求過的帳密與啟用信。

要用商城的人由**所屬組織**逐筆開通，這正是 §X-05.1「帳號由所屬組織核發」的字面
意思，也讓每一次核發都有操作者、時點與稽核。

### 首次核發與重發是同一支

差別只在 `account` 是不是空的：

| 會員狀態 | 這一支做什麼 | `first_issue` |
| --- | --- | --- |
| `account` 為 `null` | 取號、寫入帳號、交付憑證 | `true` |
| 已有 `account` | **帳號不變**，只重新交付憑證並作廢舊連結 | `false` |

**帳號一經核發即不可變更**（§X-05.1）：換掉它，該會員既有的訂單、評價與稽核就
指向一個他再也登不進去的識別。模型層有 `LogicException` 守著這件事。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `channel` | string | ✅ | `email` \| `sms` \| `offline` |

### 回應 200

```json
{
  "success": true,
  "message": "商城帳號已核發，初始憑證已依指定方式交付。",
  "data": {
    "member": { "…": "同會員清單單筆" },
    "account": "TPEUNION-000031",
    "credential": {
      "channel": "email",
      "destination": "w***@example.com",
      "initial_password": null,
      "expires_at": "2026-08-21T09:00:00+08:00"
    },
    "delivery": {
      "status": "delivered",
      "channel": "email",
      "destination": "w***@example.com",
      "expires_at": "2026-08-21T09:00:00+08:00",
      "failure_reason": null,
      "retryable": false
    },
    "first_issue": true
  },
  "errors": null,
  "meta": null
}
```

重發時 `message` 為 `初始憑證已重新交付；帳號不變，先前未使用的連結已失效。`

### `delivery` 欄位：核發與交付是兩件事

取號成功但憑證**當下送不出去**（郵件供應商、簡訊閘道故障）時，這一支仍回
**200 成功封裝**，由 `delivery.status` 說出交付沒成功。

回 422 的舊作法有三個問題：422 的語意是「什麼都沒發生」，而帳號其實已經核發了；
帳號只出現在訊息字串裡，前端得解析中文句子才拿得到；而「該重送」與「該重新核發」
這兩個完全不同的下一步，422 對它們說同一句話。

| 欄位 | 說明 |
| --- | --- |
| `status` | `delivered`（已送出）或 `failed`（取號完成、憑證沒送出去） |
| `channel` | 本次使用的管道 |
| `destination` | 遮罩後的送達位址；`offline` 為 `null` |
| `expires_at` | 啟用連結到期時間；失敗或 `offline` 為 `null` |
| `failure_reason` | 失敗原因，供操作者判斷是暫時性還是設定問題 |
| `retryable` | 是否可以重打同一支重試。**帳號不會變** |

`status = failed` 時 `credential` 為空物件：**交付失敗不得降級輸出明文密碼**。
Email 送不出去就回一組初始密碼「讓操作者先抄下來」，等於用一次供應商故障繞過
`member.credential.offline` 的權限分離。

> **重送不取新號。** `first_issue` 為 `false` 時帳號一定與先前相同——帳號一經核發
> 即不可變更（§X-05.1）。前端看到 `failed` 該做的是重新交付或換管道，不是重新核發。

### `credential` 欄位

| `channel` | `destination` | `initial_password` | `expires_at` |
| --- | --- | --- | --- |
| `email` | 遮罩後的信箱 | `null` | 啟用連結到期時間（ISO 8601，預設 72 小時） |
| `sms` | 遮罩後的門號 | `null` | 同上 |
| `offline` | `null` | **一次性初始密碼明文**，僅此一次回傳 | `null` |

`offline` 的初始密碼**只會在這個回應裡出現一次**，不會寫入稽核，也沒有第二次查詢的
端點。前端必須讓操作者當場複製或列印；掉了就再打一次這一支。使用該密碼登入後強制
變更密碼。

`offline` 另外需要 **`member.credential.offline`**：`email` 與 `sms` 送的是一次性
連結，密碼由會員自己設定，經辦人從頭到尾看不到；`offline` 則把一組可直接登入的密碼
交到經辦人手上。兩者差的是「能不能以該會員的身分登入」，因此不是同一個權限。

### 失敗

| 狀態 | 情況 | 訊息 |
| --- | --- | --- |
| 403 | 沒有 `member.account_issue` | `您沒有核發商城帳號的授權 (spec §X-05.1)。` |
| 403 | 選 `offline` 但沒有 `member.credential.offline` | `您沒有以線下管道核發初始密碼的授權，請改用 Email 或簡訊寄送啟用連結 (spec §X-05.1)。` |
| 422 | 會籍非有效（退會）而**尚未**有帳號 | 入會與復籍請於會務系統辦理 |
| 422 | 商城側停權而**尚未**有帳號 | 已有帳號者仍可重發：停權不是刪帳號，他還有既有訂單與申訴（§P-07.2） |
| 422 | 選 `email` 卻沒有可用信箱、選 `sms` 卻沒有可用門號 | 訊息會指名改用哪種方式 |

> 422 的情況一律**不佔號**：交付預檢跑在取號之前，因此失敗之後該會員仍是「尚未
> 開通」，補齊資料再打一次即可。
>
> **交付當下送不出去不在這張表裡**：那時取號已經完成，回的是 200 與
> `delivery.status = failed`（見上）。兩者的差別是「還沒開始」與「做了一半」，
> 而它們的下一步不同——前者補資料，後者重新交付。

---

## 12. 取用未遮罩身分證

```
GET platform/members/{member}/identity
```

需要 **`member.identity.reveal`**（預設只有平台管理者與財務人員有，客服沒有）。

### 回應 200

```json
{
  "success": true,
  "message": "已取用未遮罩身分證，本次取用已留稽核。",
  "data": { "identity_card": "A123456789" },
  "errors": null,
  "meta": null
}
```

> **每一次呼叫都會寫一筆稽核**（誰、何時、看了誰的、用途）。這是身分證明文保存下的
> 三道防線之一。前端**不得預先呼叫或快取結果**：不要在進入會員詳情頁時順手取用，
> 必須是操作者明確點擊「顯示完整身分證」才呼叫，且取得的值不進 localStorage、不進日誌。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 403 | 沒有該權限 | `您沒有取用未遮罩身分證的授權 (spec §X-05.4)。` |
