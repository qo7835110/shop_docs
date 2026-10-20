# 01 認證

對應 spec X-05、X-05.1、X-05.2、B-01、§2.2、§2.2.1、§12.2。

## 三條進入系統的路，結果不同

| 路 | 誰用 | 進到哪個組織 | 能不能開店 |
| --- | --- | --- | :---: |
| `auth/register` | 一般消費者自行註冊 | **平台預設組織**（§2.2.1） | ❌ |
| `auth/activate` | 工會核發帳號的會員 | 該工會 | ✅ |
| `auth/login` | 兩者都用 | —— | —— |

忘記密碼走[第 6 條](#6-忘記密碼申請重設連結)：連結寄到會籍上的 Email 或手機，兩種帳號都適用（只要已啟用過）。

**自行註冊的人開不了店**：平台預設組織不承載店面、沒有工會管理者、沒有商城頁
（§2.2.1）。要開店必須先加入某個工會，而會籍由**工會核發**，不是自己申請的 ——
那條路走的是[啟用帳號](#2-啟用帳號)。

## `account` 是唯一的登入識別碼

全站唯一、不可變更、由組織核發（§X-05.1）。**電話與 Email 只是聯絡方式，
不能拿來登入。** 前端的登入表單請標成「帳號」而不是「Email」。

## 平台身分是另一個世界

平台管理員走 `platform/auth/login`，用的是**完全隔離的憑證與 guard**。
會員 token 打 `platform/*` 會在**認證層**被拒（401），反之亦然 ——
那不是權限不足，是主體錯了（§2.2、§12.2）。

---

## 0. 健康檢查

```
GET health
```

不需 token。給負載平衡器與監控用。

```json
{ "success": true, "message": "服務正常", "data": { "status": "ok" }, "errors": null, "meta": null }
```

> 路徑是 `/api/health`，**不在 `storefront/` 底下**。

---

## 1. 註冊

```
POST auth/register
```

不需 token。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `name` | string | ✅ | 最長 50 |
| `identity_card` | string | ✅ | 一碼英文 + 九碼數字（`A123456789`），大小寫與空白會正規化 |
| `password` | string | ✅ | 8 ～ 100 碼，**需 `password_confirmation` 相符** |
| `phone` | string\|null | — | 最長 20 |
| `email` | string\|null | — | Email 格式、最長 255 |
| `agree_transaction_terms` | boolean | ✅ | **必須為 true** |
| `agree_privacy_policy` | boolean | ✅ | **必須為 true** |

```json
{
  "name": "王小明",
  "identity_card": "A123456789",
  "password": "s3cret-passphrase",
  "password_confirmation": "s3cret-passphrase",
  "email": "ming@example.com",
  "agree_transaction_terms": true,
  "agree_privacy_policy": true
}
```

### 回應 201

```json
{
  "success": true,
  "message": "註冊完成，已為您登入；您的帳號為登入識別，請妥善保存。",
  "data": {
    "account": "PF-000042",
    "token": "42|xxxxxxxxxxxxxxxxxxxxxxxx",
    "token_expires_at": "2026-11-17 10:00:00",
    "token_idle_timeout_minutes": 10080,
    "membership": { "…": "會籍資料，形狀見 buyer/profile" }
  },
  "errors": null,
  "meta": null
}
```

**`account` 由系統產生**，不由你指定 —— 開放傳入等於把全站唯一性交給呼叫端。
使用者從沒輸入過它，因此**前端一定要把它顯示出來並提醒使用者記下**：
那是他**下次**登入唯一能用的識別。

### 註冊直接回 token，不必再登入一次

`token` 就是這一次的登入狀態，拿去打買家端點即可。時限與登入相同，見[第 3 條的 token 時限](#token-時限)。

> 這一條是使用者旅程模擬（plans/24）改出來的。原本註冊只回 `account`
> 與會籍，訊息是「請以帳號登入」—— 而那表示前端只有兩條路：要使用者把剛拿到的
> 一串代碼抄下來、連同三秒前才設好的密碼再打一次，或在背景自動補一次
> `auth/login`。所有前端都會選第二條，也就是把一件後端做一次就好的事，
> 推給每個前端各做一次。

### 兩項同意是硬性的

未同意必要的交易與隱私條款**不得完成註冊**（§B-01），且這兩項事後也**撤不掉**
（見 [buyer/consents](../buyer/01-profile-and-notifications.md#3-同意設定) 的
`revocable`）—— 撤回等於不得使用本服務。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 兩項同意未勾選、密碼太短或兩次不一致、Email 格式錯誤 |
| 422 | 平台預設組織尚未建立（部署未完成，訊息會指出這件事） |


> **為什麼註冊要填身分證號**
>
> 它是日後**跨工會歸戶**的唯一依據：缺了它，這位買家之後被工會建檔時會被當成
> 另一個人而拿到第二組帳號，而兩邊永遠合不起來。
>
> 它**不是購買條件** —— 整條購買流程都不讀它，而且輸出一律遮罩（§2.3），
> 連本人的個人資料頁看到的也是遮罩值。
>
> 同一組身分證號在平台只能註冊一次，重複時回 **422**，訊息會指引登入或重設密碼。
> 已經有工會會籍的人自行註冊時**不會**併入既有帳號：註冊是未經驗證的入口，
> 若比對到就沿用，等於讓任何知道他人身分證號的人在別人帳號上開會籍。

---

## 2. 啟用帳號

```
POST auth/activate
```

不需 token。**工會核發帳號後**，你會收到一個啟用連結（Email 或簡訊），
或由工會當面交付一組一次性初始密碼。

Email 啟用信的連結是 `{APP_URL}/activate#token=…`：權杖在 `#` 片段，瀏覽器不會送到伺服器。
後端附的 `/activate` 頁讀出權杖、讓會員設定密碼後呼叫本端點，成功即顯示 `data.account`。
自建前端時照同一做法：從片段取權杖，不要改成查詢字串。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `token` | string | ✅ | 啟用連結裡的權杖 |
| `password` | string | ✅ | **至少 8 碼且須含英文與數字**，需 `password_confirmation` 相符 |
| `agree_transaction_terms` | boolean | ✅ | **必須為 true**（§B-01：未同意必要條款不得啟用） |
| `agree_privacy_policy` | boolean | ✅ | **必須為 true** |

```json
{ "token": "…", "password": "myPassw0rd", "password_confirmation": "myPassw0rd", "agree_transaction_terms": true, "agree_privacy_policy": true }
```

### 回應 200

```json
{
  "success": true,
  "message": "帳號已啟用，請以此帳號與您設定的密碼登入。",
  "data": { "account": "TPEUNION-000031" },
  "errors": null,
  "meta": null
}
```

### 啟用連結會過期

有效時數由系統參數 `account_activation_hours` 決定（預設 **72 小時**）。
逾時請聯絡所屬工會重發 —— 平台端有重發端點，但那是工會或平台人員操作的。

**密碼規則比註冊嚴格**（要求英文＋數字）：這條路徑的帳號是工會核發的，
初始密碼由第三方交付過一次，因此設定新密碼時的門檻拉高。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 權杖無效、已使用過、已逾期，或給的是重設密碼連結（兩種連結互不通用） |
| 422 | 未勾選兩項必要同意（`errors` 指出欄位）；同意與啟用同一交易，會留下同意快照 |
| 422 | 密碼不符規則或兩次不一致 |

---

## 3. 登入

```
POST auth/login
```

不需 token。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `account` | string | ✅ | 最長 32 |
| `password` | string | ✅ | 最長 100 |

### 回應 200

```json
{
  "success": true,
  "message": "登入成功。",
  "data": {
    "token": "7|xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "token_expires_at": "2026-11-17 10:00:00",
    "token_idle_timeout_minutes": 10080,
    "account": "DEMO-1a258c",
    "must_change_password": false,
    "membership": {
      "id": 8,
      "member_number": "1a258cdd",
      "name": "示範買家",
      "membership_status": "active",
      "marketplace_status": "active",
      "is_tradable": true,
      "organization": { "id": 2, "code": "DEMO", "name": "示範職業工會", "type": "union" },
      "required_consents_granted": true,
      "shops": [
        { "id": 1, "name": "示範工會商店", "slug": "demo-shop", "status": "open" }
      ],
      "…": "完整形狀見 buyer/profile"
    }
  },
  "errors": null,
  "meta": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `token` | 後續請求帶 `Authorization: Bearer {token}`。 |
| `token_expires_at` | 這張 token 的**絕對到期時點**（UTC，至秒，與其他時間欄位相同；顯示時加 8 小時為台北時間）。用得再勤也會在這一刻失效。 |
| `token_idle_timeout_minutes` | **閒置時限**：連續這麼多分鐘沒有任何請求，token 即失效；每次請求重新起算。 |
| `must_change_password` | 為 `true` 表示用的是初始密碼，**前端應強制導向變更密碼**。 |
| `membership` | 這組帳號所屬的會籍。**永遠不是 `null`** —— 帳號就在會籍上，登不進來的會籍不會走到這裡。 |
| `membership.can_open_shop` | 這個組織能不能開店。平台預設組織（自行註冊者）恆為 `false`（§2.2.1）。「我要開店」按鈕看它。 |
| `membership.eligible_for_member_price` | 這個會籍能不能享會員價。由組織類型決定，後端判定（§13.5.2）。 |
| `membership.membership_paid` | 會費繳納狀態，由會務系統同步，商城**唯讀**（§X-05）。 |
| `membership.required_consents_granted` | 必要的交易與隱私條款是否已同意（§B-01）。`false` 時**送出訂單與開店申請會被擋**，請先引導使用者以 `PUT buyer/consents` 同意 `transaction_terms`、`privacy_policy`。臨櫃核發初始密碼的帳號沒有經過任何同意畫面，最常是 `false`。 |
| `membership.shops` | 這個會籍名下的店面（沒有店面時是空陣列）。**賣家後台的第一個畫面靠它** —— 賣家側每一個列表都要 `?shop_id=`，而這是唯一回答得了「我有哪些店」的地方。詳見 [buyer/profile](../buyer/01-profile-and-notifications.md#1-個人資料)。 |

### token 時限

token 有兩道時限（plans/35 35-5），任一到了都會失效，之後的請求一律 **401**，請重新登入：

| 時限 | 會員預設 | 系統參數 |
| --- | --- | --- |
| 絕對時效（自登入起算） | 30 天 | `member_token_lifetime_minutes` |
| 閒置時限（自最後一次請求起算） | 7 天 | `member_token_idle_minutes` |

兩個值在**登入當下**凍結到這張 token 上：平台事後調整參數只影響之後的登入。
失效的 token 會被刪除，不會再變回有效。前端可依 `token_expires_at` 在到期前提示重新登入；
閒置時限無法預告確切時點，收到 401 時導回登入即可。

### 一個工會一組帳號

`account` 由**所屬組織**核發。一個人加入兩個工會就有兩組帳密，登入哪一組
就是哪一個工會的身分 —— 會員價資格、會員限定商品與租戶範圍全部隨之而定。

**沒有切換端點，也不需要。** 一個 token 的身分永遠不變，因此前端不必擔心
「另一個分頁改了身分」這種事。要換身分就用另一組帳密登入，拿到的是另一張 token。

兩組帳號在資料庫裡仍知道是同一個自然人（供日後歸戶用），但那條關聯不對外
輸出，也不共用憑證。

### 失敗

| 狀態 | 情況 | message |
| --- | --- | --- |
| 401 | 帳號不存在或密碼錯誤 | `帳號或密碼不正確。` |
| 401 | 帳號已停用 | `此帳號已停用，請聯絡所屬組織。` |
| 429 | 同帳號、同來源 IP 連續失敗 5 次，鎖 900 秒 | `登入嘗試次數過多，請於 N 秒後再試。` |

> **帳號不存在與密碼錯誤回同一句話**，前端不要嘗試區分 —— 那會變成帳號列舉工具。
>
> 每一次登入（成功與失敗）都會寫入 `authentication_logs`，含 IP 與 User-Agent。

---

## 4. 登出

```
POST auth/logout
```

需要會員 token。無請求 body。

```json
{ "success": true, "message": "已登出。", "data": null, "errors": null, "meta": null }
```

**只註銷當前這個 token**，其他裝置的登入不受影響。

---

## 5. 變更密碼

```
POST auth/password
```

需要會員 token。**這是 `must_change_password` 的出口。**

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `current_password` | string | ✅ | 目前的密碼；不正確回 422 |
| `password` | string | ✅ | 最少 8 碼、須含英文字母與數字，且**不得與目前密碼相同** |
| `password_confirmation` | string | ✅ | 與 `password` 相同 |

```json
{
  "current_password": "Init-9f2a7c",
  "password": "my-new-passphrase-7",
  "password_confirmation": "my-new-passphrase-7"
}
```

### 回應 200

```json
{
  "success": true,
  "message": "密碼已變更，其他裝置上的登入已一併失效。",
  "data": { "must_change_password": false },
  "errors": null,
  "meta": null
}
```

### 三件事同時發生

1. **`must_change_password` 轉為 `false`。** 那個旗標的意義就是「還沒換過」。
2. **其他裝置上的 token 全部失效**，目前這一個保留 —— 換密碼通常發生在
   「我覺得別人知道我的密碼」之後；留著舊 token 等於換了一道沒人走的門。
   而操作者本人不該在成功的那一刻被登出，他什麼也沒做錯。
3. **稽核留一筆**（誰、何時、IP），但**不含任何密碼值**：舊的、新的、雜湊都不寫入
   （§X-02：敏感資料不得進入日誌）。

### 為什麼掛在 `auth/` 而不是 `buyer/profile`

密碼是登入主體 `user` 的憑證，而一個人可能有**多個會籍卻只有一組密碼**
（§2.2、§X-05）。放進個人資料底下，「改哪一個會籍的密碼」就變成一個沒有答案的問題。

### 與啟用帳號的關係

`auth/activate`（第 2 條）**不是替代品**：它需要一次性啟用**連結**，而連結只在
`email` / `sms` 管道核發。`offline` 管道回的是一組明文初始密碼、沒有連結
—— 那組密碼經線下管道交付，交付過程中看過它的人不只本人，因此 §X-05.1
要求「使用該密碼登入後強制變更密碼」。這一條就是那個「變更」。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 401 | 未登入，或以平台管理員 token 呼叫（兩個身分主體在**認證層**就分開，§2.2） |
| 422 | 目前密碼不正確／新密碼與目前相同／強度不足／兩次輸入不一致 |

---

## 6. 忘記密碼（申請重設連結）

```
POST auth/password/forgot
```

不需 token（業主 2026-10-18 決定採會員自助，plans/35 35-3）。
系統把一次性重設連結寄到**這個帳號會籍上**的 Email；Email 不可用（未填或已退信）時改發簡訊。
沒有任何可用聯絡方式的會員，仍須聯絡所屬組織處理。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `account` | string | ✅ | 最長 32 |

```json
{ "account": "TPEUNION-000031" }
```

### 回應 200 —— 永遠是這一句

```json
{
  "success": true,
  "message": "若帳號存在且留有可用的 Email 或手機，重設連結已寄出，請於時效內開啟。",
  "data": null,
  "errors": null,
  "meta": null
}
```

帳號不存在、已停用、尚未開通（還沒啟用過）、沒有可用聯絡方式、冷卻中、寄送失敗 ——
**回應一字不差**。任何差異都能拿來列舉帳號，或確認某個帳號有沒有留信箱。
前端也不要自行推測「寄了沒」：照這句話顯示即可。

- **尚未啟用的帳號不寄**：那條路是組織核發的[啟用連結](#2-啟用帳號)，不從這裡繞過。
- **回應不等寄信**：查帳號與寄送都在回應送出後才做，回應快慢也看不出帳號是否存在。信件或簡訊可能晚幾秒到，畫面請寫「請稍候查收」，不要寫「已寄達」。
- **同一帳號 60 秒內只寄一次**；新連結寄出時，舊的未使用連結立即作廢。
- 以 IP 限流：每分鐘 5 次，超過回 **429**。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 未填 `account` |
| 429 | 同一 IP 請求過於頻繁 |

---

## 7. 重設密碼

```
POST auth/password/reset
```

不需 token。重設信的連結是 `{APP_URL}/reset-password#token=…`：權杖在 `#` 片段，
與[啟用連結](#2-啟用帳號)同一做法。後端附的 `/reset-password` 頁讀出權杖、讓會員設定新密碼後呼叫本端點。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `token` | string | ✅ | 重設連結裡的權杖 |
| `password` | string | ✅ | **至少 8 碼且須含英文與數字**，需 `password_confirmation` 相符 |

```json
{ "token": "…", "password": "newPassw0rd", "password_confirmation": "newPassw0rd" }
```

### 回應 200

```json
{
  "success": true,
  "message": "密碼已重設，所有裝置皆已登出，請以新密碼登入。",
  "data": { "account": "TPEUNION-000031" },
  "errors": null,
  "meta": null
}
```

### 三件事同時發生

1. 密碼換成新的，`must_change_password` 清為 `false`（密碼是會員自己剛設的）。
2. 連結標記為已使用，不能再用第二次。
3. **既有的每一個工作階段都被登出** —— 忘記密碼往往是因為憑證外洩。成功後不簽發 token，請重新登入。

連結時效由系統參數 `password_reset_link_minutes` 決定（預設 **60 分鐘**），
遠短於啟用連結的 72 小時：重設是會員當下自己要的，信箱被別人讀到時，能用的窗口越短越好。

**重設連結與啟用連結互不通用**：把重設權杖送到 `auth/activate`、或把啟用權杖送到這裡，都回 422。

### 失敗

| 狀態 | 情況 |
| --- | --- |
| 422 | 權杖無效、已使用過、已作廢、已逾期，或不是重設連結（一律同一句「重設連結無效或已逾期，請重新申請。」） |
| 422 | 密碼不符規則或兩次不一致 |
| 429 | 同一 IP 每分鐘超過 10 次 |
