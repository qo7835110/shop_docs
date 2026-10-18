# PAYUNi 開通（平台代建商店）

spec §S-10.1i、[Plan33](../../../plans/33-payuni-seller-onboarding.md)。工會核准開店後，店主填寫 PAYUNi 開通資料並同意，
平台以代理商身分替本店建立 PAYUNi 商店、自動綁定並代申請物流。
已綁定 PAYUNi 的店面不需辦理；既有商家請改用 [商家申請](10-payuni-applications.md)。

需要有效會員 token、店主本人及本店 `shop.manage`；店面須為 `approved` 或 `open`。目前僅測試環境（sandbox）。

## 提交

```text
POST seller/shops/{shop}/payuni-onboarding
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `submission_key` | 是 | UUID；同店同鍵同內容回原案，異內容 422 |
| `identity_type` | 是 | `individual`（以會員姓名與身分證字號開通，兩者須已存在）或 `company` |
| `company_ubn` | 公司必填 | 8 位數字統編且檢查碼正確（PAYUNi 會以 `AGENT03004` 拒絕）；個人不得帶 |
| `company_name` | 公司必填 | 1～50 字，須與登記名稱相同；個人不得帶 |
| `email` | 是 | PAYUNi 會員與商店客服信箱，最多 50 字 |
| `contact_name` | 是 | 商店聯絡人，最多 20 字 |
| `contact_mobile` | 是 | `09` 開頭 10 碼 |
| `merchant_name` | 是 | 商店中文名稱，最多 20 字且須為 BIG5 可表示字元 |
| `merchant_english_name` | 是 | 最多 50 字；英數、半形空白與 `~!@#$%^&*()_-+={}[]\:;"<,>.?/'` |
| `address` | 是 | 商店聯絡地址，最多 50 字 |
| `address_postcode` | 是 | 3 碼郵遞區號 |
| `mcc` | 是 | PAYUNi 行業別代碼 1～999（例：64 百貨商店、19 服裝、3 食品） |
| `unit_price` | 是 | 平均客單價（元），1～99,999,999 |
| `c2c_enabled` | 是 | 是否開 7-ELEVEN 店到店常溫 |
| `c2c_return_store_id` | 開超商時必填 | 6 碼 7-ELEVEN 退貨門市代碼；**PAYUNi 建立後不得修改** |
| `home_enabled` | 是 | 是否開黑貓宅配 |
| `home_sender_name` | 開黑貓時必填 | 寄件人，最多 15 字 |
| `home_sender_mobile` | 開黑貓時必填 | `09` 開頭 10 碼 |
| `home_sender_address` | 開黑貓時必填 | 寄件地址，最多 120 字，須含縣市、鄉鎮市區與門牌號（「台」「臺」皆可）且真實存在；**開通後無法自行修改，需洽 PAYUNi 客服**，前端須在欄位旁明示 |
| `home_product_type` | 開黑貓時必填 | `0001`～`0015`（例：`0009` 生活用品、`0015` 其他） |
| `consent` | 是 | 須為 true：同意將上述資料及身分資料提供給 PAYUNi |
| `consent_version` | 是 | 目前同意書版本 `payuni-onboarding-2026-10-16`；前端須顯示該版本文字 |

每店每環境同時只有一件辦理中。未啟用的物流欄位一律不保存。個人身分證字號不寫入開通案，送 PAYUNi 時才從會員資料讀取。

## 建店

受理後（部署已開啟 `PAYUNI_ONBOARDING_ENABLED` 時）立即以代理商金鑰呼叫 PAYUNi 新增合作商店一次；未開啟時只受理資料、維持 `submitted`（商店代號由系統配發；開信用卡一次付清，寄件費由平台代理商統一設定）。
建店成功狀態轉 `created`；PAYUNi 明確拒絕（例如郵遞區號不存在）轉 `failed`，可修正資料後以新提交鍵重提；
網路逾時或代號已存在時維持 `submitted`，由排程 `payuni:process-onboardings` 以商店狀態查詢確認，**不會重送建店**。
提交的回應即反映這次嘗試後的狀態。

建店確認後同一流程自動綁定：平台建立的商店歸屬由建立事實證明，不需上傳授權文件；付款設定（信用卡、3D、自動請款、撥款指示）視為 PAYUNi 預設已開。
收款仍須 PAYUNi 商店狀態查詢顯示會員、商店、信用卡皆啟用（能力同步）。

綁定後依選擇代申請 7-ELEVEN 店到店常溫與黑貓宅配，帶入平台的貨態推送網址；每個通路只送一次（超商建立後不得修改），
所選通路都有結果即轉 `completed`，並觸發一次能力同步。結帳是否提供超商／宅配，仍依商店狀態查詢中該物流為啟用。

## 查看

```text
GET seller/shops/{shop}/payuni-onboarding
```

回最近一件開通案；沒有時 `data=null`。

## 回應

成功與重播皆 200，`data`：`id`、`shop_id`、`environment`、`status`（`submitted` 已受理／`created` 已建店並綁定、物流處理中／`completed` 物流代申請已送出／`failed` 建店失敗）、
`identity_type`、`c2c_enabled`、`home_enabled`、`c2c_set_status`／`home_set_status`（PAYUNi 物流開通狀態：`1` 正常、`9` 待審查、`8` 審核中、`2` 審查失敗；未送出或結果不明為 null）、`failure_code`／`failure_message`（僅 `failed` 時：PAYUNi 拒絕碼與賣家可讀說明，例如 `AGENT03004` 統編格式有誤）、`consent_version`、`submitted_at`。不回吐統編、手機、地址或提交鍵。

未登入 401；非店主或無 `shop.manage` 403；格式錯誤、未同意、店面未核准或停權、已綁定、已有辦理中、非 sandbox 422。

## 黑貓寄件資料

PAYUNi 黑貓開通後，寄件人姓名、電話、手機與商品類別可由平台重打黑貓建店 API 更新；**寄件地址只能洽 PAYUNi 客服**（service@payuni.com.tw）。
限經本流程開通黑貓（`completed` 且黑貓建店成功）且綁定仍有效的店面。

```text
GET seller/shops/{shop}/payuni-home-sender
```

未開通黑貓時 `data=null`。否則 `data`：`shop_id`、`home_set_status`、`sender_name`、`sender_mobile`（遮罩，例 `09***21`）、
`sender_tel_area_code`、`sender_tel`（遮罩）、`product_type`、`address_editable`（恆為 false）、`address_change_contact`（PAYUNi 客服信箱）、
`latest_update`（最近一次更新，格式同下方回應；沒有時 null）。不輸出寄件地址。目前值為最近一次成功的更新，否則為開通時所填。

```text
POST seller/shops/{shop}/payuni-home-sender
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `submission_key` | 是 | UUID；同店同鍵同內容回原紀錄、不重送，異內容 422 |
| `sender_name` | 是 | 寄件人，最多 15 字 |
| `sender_mobile` | 是 | `09` 開頭 10 碼 |
| `sender_tel_area_code` | 與 `sender_tel` 同時 | 2～3 碼數字 |
| `sender_tel` | 與 `sender_tel_area_code` 同時 | 1～8 碼數字 |
| `product_type` | 是 | `0001`～`0015` |

送出時一律帶開通時的原地址（PAYUNi 要求必帶、但不會因此改地址）與平台貨態推送網址，只送一次。
同店同時只有一件處理中；逾 5 分鐘未記錄結果者視為不明，可再送。

回應 200，`data`：`id`、`status`（`succeeded` 已更新／`failed` PAYUNi 拒絕／`unknown` 結果不明，可換新提交鍵重送，內容相同重打無副作用）、
`status_code`、`failure_message`（僅 `failed`）、`sender_name`、`product_type`、`requested_at`、`recorded_at`。
稽核留新舊值（電話遮罩）。未登入 401；非店主或無 `shop.manage` 403；格式錯誤、未開通黑貓、上一筆處理中、代建開關未開或非 sandbox 422。每分鐘限 10 次。

