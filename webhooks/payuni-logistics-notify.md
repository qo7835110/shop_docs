# PAYUNi 貨態推送接收

spec §S-12.1；[詢問工作單](../../integration/payuni-provider-confirmation.md) L-N（寄送第 14 題）。
這是商店物流設定的貨態 Notify（超商 `logistics/setting`、黑貓 `home_delivery/setting` 的 `NotifyURL`），
不是代理商專區共用通知（A4，業主無法設定，未使用）。

`POST /api/webhooks/payuni/logistics/notify`

網址固定、不含秘密；驗簽即接收授權，不使用會員／平台 token。限 `payuni.environment=sandbox`，每分鐘 120 次。

## 接收規則

- 接受原始 `application/x-www-form-urlencoded` 或 `application/json`，最多 65,536 bytes；
  外層為 `Status`、`MerID`、`Version`、`EncryptInfo`、`HashInfo`（文件未載明傳輸格式，兩種都收）。
- 先以代理商金鑰（受控版本）驗簽解密；不通過時，若外層 `MerID` 正是設定中的 sandbox 測試商店，再試該商店金鑰。
  記錄是哪一把通過（`agent`／`merchant`），以回答「推送用哪把金鑰加密」。解密後 `MerID` 須與外層相同。
- **每次到達都保存一列，重送不去重**，以觀察 PAYUNi 是否推送、重送次數與間隔。

## 回應

沿用 `ApiResponse` 五欄 JSON，均 no-store：

- **200**：已驗簽並保存；`success=true`、`data=null`。PAYUNi 是否把它當成接收成功尚未確認（第 14 題）。
- **422**：無法驗簽、格式不符或非 sandbox。無法驗簽時仍留一列到達紀錄（內容指紋、外層 MerID、失敗代碼），
  不保存原文與貨態；非 sandbox 不保存。

## 保存與界線

`payuni_logistics_notifications`：收件時間（UTC）、傳輸格式、原文 SHA-256、驗簽金鑰、失敗代碼、外層 MerID、
物流型態、UNi 物流序號、貨態代碼與貨態時間；驗簽成功者另以 APP_KEY 加密保存原文，供日後採用時重新驗簽。
不可更新／刪除。接收本身**不寫 `logistics_events`、不推進出貨／訂單／撥款**。

## 推送觸發查詢（業主 2026-10-17）

推送內容沒有原 `MerTradeNo`／`TradeNo`／`Odno`，不足以核對原單，所以只當喚醒訊號：排程
`payuni:sweep-logistics-notifications` 每分鐘取已驗簽、帶物流序號與貨態的新推送，以原環境＋外層 `MerID`
的商店身分＋物流序號找已核實的原包裹關聯，立即發一次物流單查詢（不等輪詢間隔，但為之後的輪詢保留一個間隔），
再走既有採用。工作鍵由關聯＋貨態＋貨態原始時間推導：PAYUNi 重送同一貨態只讀回原查詢、不重送。
每則推送一個處理決定，存於 `payuni_logistics_notification_triggers`（`queried`／`replayed`／`unconfirmed`／
`unmatched`／`ineligible`，不可更新／刪除）。限 sandbox，需 `PAYUNI_LOGISTICS_NOTIFICATION_TRIGGER_ENABLED`
與查詢、輪詢開關；採用開關開啟時當場採用，否則由採用排程接手。

查看：`php artisan payuni:logistics-notifications --limit=20`（去敏欄位，不含原文）。
