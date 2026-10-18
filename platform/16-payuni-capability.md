# PAYUNi sandbox 能力查詢

spec §S-10.1d。僅有效平台管理員且具 `payuni.binding.manage`，固定 URL 店面與 sandbox 綁定。會員 token 401、財務／客服及錯配店面 403、不存在識別 404。GET／POST 回應皆為標準 ApiResponse，帶 `Cache-Control: private, no-store`。

## 最新歷史

```http
GET /api/platform/shops/{shop}/payuni-bindings/{payuniBinding}/capability-query
```

不呼叫 PAYUNi、不讀原封包。沒有工作時 `data=null`；已有工作但結果未知時 `data.observation=null`。已解除的 sandbox 綁定仍可查歷史。讀取不延長有效期限。

## 單次 A3 查詢

```http
POST /api/platform/shops/{shop}/payuni-bindings/{payuniBinding}/capability-query
Content-Type: application/json

{"job_key":"12345678-1234-4234-8234-123456789abc"}
```

| 欄位 | 必填 | 說明 |
| --- | --- | --- |
| `job_key` | 是 | 正規小寫 UUID；由操作者為一次查詢建立並保留。不得帶前後空白。 |

不接受 `merchant_id`、`agent_id`、`environment`、`credential_version` 或 `url`。原身分、憑證版本、固定 A3 端點全由伺服器取得。POST 限流每分鐘 10 次。

前置條件：原生效 sandbox 綁定、已保存且有效的代理商憑證版本、正值資料表參數 `payuni_capability_ttl_seconds` 與 `payuni_capability_query_timeout_seconds`，並開啟 `PAYUNI_CAPABILITY_SYNC_ENABLED=true`。能力查詢不要求 `PAYUNI_ENABLED=true`；只做能力取證時保持收款關閉。已另行核准的商城 sandbox 付款交付則依[部署前置](../../ops/payuni-buyer-payment-delivery.md)與當批授權決定，不因 A3 成功自動開啟收款。

先提交工作與稽核，再於交易外單次外呼，最後留存觀察。相同工作鍵重播只回原結果（含尚未確認工作），不再外呼／新增稽核；不同綁定使用原鍵回 422。同鍵重播仍重驗即時權限。已關閉同步開關仍可查核原工作；新鍵則回 422。

`data` 欄位：`query_id`、`binding_id`、`binding_version`、`shop_id`、`environment`、`sequence`、`started_at`、`fresh_until`、`deadline_at`、`observation`。時間皆帶時區。`observation` 為 null 或包含 `id`、`outcome`（accepted／unknown）、`failure`、三層狀態 `member_status`／`merchant_status`／`credit_status`、物流開通 `logistics_c2c_status`（7-ELEVEN 店到店常溫 `SPC2C`）／`logistics_home_status`（黑貓宅配 `HOME`）、`received_at`、`adopted`。

物流兩欄的碼值同 `credit_status`（1 啟用、2 不啟用、3 未申請、4 強制關閉、9 審核中、92 審核失敗、99 其他關閉）；**null 代表本次未觀察到可信值**（欄位缺漏或文件外碼值），不是停用，也不得以信用卡狀態推論。物流欄位不參與 `adopted` 判定；買家結帳時以採用中且新鮮的觀察判定物流通路是否可選（spec §13.2.2），`received_at` 即該狀態的最後同步時間。

`adopted` 只表示**當時**採用，不表示目前可收款；過期、後續失效或更新工作都可能使其不再有效。HTTP 200 只代表查詢工作已記錄，不保證供應商啟用。unknown／空觀察不可解讀為成功，應保留原鍵查核，禁止因超時自動改用新鍵或自動重送。

此入口不變更付款、訂單、庫存、退款或撥款，不測 Notify，也不能解鎖恢復待查狀態。生效三層狀態仍不能取代付款設定及端到端驗收。
