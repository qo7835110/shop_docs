# PAYUNi 原付款查核

spec §P-10／§12.1。僅有效平台管理員持有 `settlement.reconcile`（預設財務角色）；
一般平台管理員與客服不隱含取得此權限，會員 token 為 401。既有 sandbox 代理商 session 為唯一來源。
停用新收款不阻擋歷史查核，無需開啟 PAYUNI_ENABLED，也不要求目前商家綁定仍生效。

## 最新工作歷史

```http
GET /api/platform/payments/{paymentNo}/payuni-query
```

paymentNo 為商城付款編號，不是 PAYUNi TradeNo。找不到付款回 404；原付款尚無 session 或無工作回 data=null。
只讀本地最新建立的工作，不外呼，不延長期限。正式環境 session 回 403。

## 單次原交易查核

```http
POST /api/platform/payments/{paymentNo}/payuni-query
Content-Type: application/json

{"job_key":"fd02308e-8049-4273-a3bf-3b4373000001"}
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `job_key` | 是 | 正規小寫 UUID；不修剪；同次查核重播必須用原鍵 |

拒絕所有其他欄位，包括商家、金額、環境、憑證版本或網址。限每分鐘 10 次。
無原 session 回 422，不自行建立 session、付款或表單。工作與去敏 requested 稽核同交易提交，
提交後、外送前再次查核即時授權；沿原商家、原永久短碼、原憑證版本單次 M3，保存驗簽後的觀察。
回應在途撤權仍保存觀察但不公開回條。原鍵重播只回原工作，不重送／不重複 requested 稽核，
仍需即時授權；同鍵改指其他付款回 422。新 UUID 不算重播，不得超時自動改鍵。

成功及合法重播皆 200，標準 ApiResponse，`Cache-Control: private, no-store`。
data 為 null（僅 GET）或恰含：query_id（本地整數）、started_at（帶時區時間）、
observation（null 表示尚未確認；否則為 id、outcome、failure、recorded_at）。
outcome 為 matched／unknown；failure 為固定去敏代碼或 null，不回原始錯誤。
回條不公開外部交易號、商家代號、工作鍵、session ID、金鑰、憑證版本、金額／費用或可收款旗標。
內部完整觀察仍保存供受控採用，公開入口不允許直接套用到付款。

matched 只表示原交易相符，不等於已付款、已請款、對帳相符或可以退款／重付。
查無／逾時／憑證不可得保留 unknown；提交後中斷保留 pending，不自動送第二次。
本入口不入帳、不寫 payment_events、不修改訂單或庫存、不交付表單，也不測 Notify。
封測尚無商城原 session，不能將 lab run 交易補貼成商城付款。
本地 Bruno 以合成歷史及未登錄版本驗證 unknown／原鍵重播，不外呼；不是 31-7 真實金流證據。
