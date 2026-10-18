# PAYUNi UPP 通知接收

spec §P-10／§12.1。這是 UPP M1／M2 通知，不是 A4 共用通知或既有 demo HMAC 回調。

`POST /api/webhooks/payuni/upp/notify/{paymentNo}`

`paymentNo` 是原商城付款編號，用於選取不可變 session，**不是驗證密鑰**。
付款交付準備在受控 `PAYUNI_DELIVERY_ORIGIN` 下產生此 NotifyURL，放進 UPP 加密區；
不得由買家指定網址、商家、憑證或原交易金額。只接受既有 sandbox／agent／信用卡一次付清交付。
停用新收款、原期限已過不拒絕歷史通知；查不到原 session 或原交付則不自動建立。

不使用會員／平台 token，以原 session 的環境、原商家及原代理商憑證版本驗簽。
只接受原始 `application/x-www-form-urlencoded`，最多 65,536 bytes；外層必須是
`Status`、`MerID`、`Version=2.0`、`EncryptInfo`、`HashInfo`。重複鍵／百分比別名不得覆蓋。
解密後再核對原 MerID、MerTradeNo、整數 TradeAmt 及工具／狀態，不能輪流嘗試其他金鑰。
詳見[解析規則](../../integration/payuni-upp-results.md)。

回應沿用商城 `ApiResponse` 的五欄 JSON 封裝，均 no-store，不公開付款識別、密文或通知內容：

- **200**：本次通知已可靠保存，或同一通知已有不可變紀錄；`success=true`、`data=null`。
- **422**：封包或原交易不符；固定訊息、`success=false`、`data=null`。不改成付款失敗。
- **503**：原憑證不可得或無法確認提交；不承諾已接收，保留原交易待查核。

200 是本系統的保存回條，**不是付款收據，也不證明 PAYUNi 已接受 ACK**。
M2 快照未明訂回應 body／重試規則；官方 WooCommerce `receive_response()` 範例處理後直接 exit，
不提供固定 SUCCESS／OK 字串契約。這裡不虛構供應商 ACK 字串；沿用既有 API 封裝，
仍須以封測實際通知及供應商端行為核實接受性，不推定 503 必然重送或特定重試間隔。
參考：[PAYUNi 官方模組](https://raw.githubusercontent.com/payuni/PAYUNi_for_WooCommerce/main/class-payuni.php)。

## 保存、重播與查核界線

`payuni_upp_notifications` 固定原 delivery 參照，僅保存已核對的有限交易／狀態欄位與 UTC 收件時間。
不保存卡號、授權碼、Message、原文、密文、Key／IV。DB 外鍵、原交易 scope trigger、不可更新／刪除
及通知事實唯一鍵為最後防線；空 AuthType 也透過生成鍵去重，不用 NULL 唯一鍵漏洞。
不同狀態另附一筆，晚到 UNKNOWN 不覆寫 SUCCESS；同一欄位事實的 Message／wire 順序改變不另造事件。
這是本地觀察去重，不宣稱供應商提供了事件 ID。

接收不改付款、訂單、期限、庫存、退款或撥款，不建立 payment_events；HTTP 接收本身不外呼 M3。
另有預設關閉的[通知查核 command／排程](../../ops/payuni-upp-notification-query.md)，從持久 inbox 查原交易。
後續須由原交易查核確認完整請款／費用，並走獨立採用流程；未知工作補救及受控採用入口仍待完成。
前景 Return 不可直接入帳；目前 Return／Back 尚無商城 handler，不可宣稱完整買家交付已可用。

一般 Laravel Log 的 `payuni_upp_notification` 事件只含去敏原因或本地通知／交付 ID、duplicate 與
`payment_ready=false`。可靠證據為資料庫紀錄；Log 故障不抹除已保存觀察。
部署須先 migrate。請保留新收款關閉，等交付、查核、採用與封測閉環通過再開放。

## 本地正向 HTTP 驗收

`tests/Bruno/run-full-chain.sh` 在原憑證撤銷測試前先執行 `00-payuni-notify`：
讀取同一原付款 → 合成通知 → 原通知重送 → 再讀取同一原付款。
runner 額外直接核對資料庫：收件前為零筆、重送後僅一筆，付款／訂單／庫存／期限／付款事件不變。

此資料夾只允許 localhost；使用獨立 Node AES-GCM／SHA256 與公開 M7 測試向量，
CLI 此段需 `--sandbox developer`，其他資料夾保持預設 safe。專用 seeder 只准在
`marketplace_pro_chain`／`marketplace_pro_test` 建立獨立商店與未付款交易，並恢復暫時設定；
不包含在一般 DatabaseSeeder，不呼叫真實 PAYUNi、不向瀏覽器交付付款表單。
請勿將這份合成請求指向封測站，也不可把本地通過視為供應商通知／ACK 通過。
