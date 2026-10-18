# PAYUNi 商家日帳查詢

spec §P-08／§14.9.3。僅有效平台財務人員持有 `settlement.reconcile` 可操作；
會員 token 為 401，一般平台管理員與客服預設為 403。來源限原 sandbox 代理商付款 session。
查詢涵蓋該原商家整天的 grant_query 資料，不只 URL 上的一筆付款；不得當作付款前置對帳。

## 建立批次

```http
POST /api/platform/payments/{paymentNo}/payuni-grant-queries
Content-Type: application/json

{"batch_key":"fd02308e-8049-4273-a3bf-3b4373000010","query_date":"2026-10-01"}
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `batch_key` | 是 | 正規小寫 UUID；不修剪，同次操作必須保留原鍵 |
| `query_date` | 是 | 嚴格 YYYY-MM-DD；新批次限台北當日及前 29 天 |

建立只保存本地批次，不外送。新批次須 `PAYUNI_GRANT_QUERY_ENABLED=true`
（預設 false）及可用的原憑證版本。**無需也不得因此開啟收款或撥款。**
同鍵同來源與日期只回原批次，關閉開關或過期仍可重播；改來源／日期回 422。
不存在付款回 404，無 session 回 422，正式環境 session 回 403。

已授權且範圍檢查通過後，若新批次被日帳開關拒絕，422 的 `message` 固定為
`sandbox 商家日帳查詢尚未啟用；請保留原批次鍵，勿自動重送。`。
其他批次範圍錯誤仍為 `原商家查詢批次範圍不符。`，不公開私有底層例外。
此分類只說明拒絕原因，不授權用戶端更換鍵、啟用開關或送出撥款。

## 單頁查詢

```http
POST /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages
Content-Type: application/json

{"page":1}
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `page` | 是 | 正整數 JSON number；拒絕字串、浮點、布林、溢位 |

兩個 POST 均拒絕額外欄位並限制每分鐘 10 次。外部 MerID、環境、金額、網址及憑證不可由呼叫端指定。
grant_query 無 NotifyURL 參數，不自行添加；本功能不恢復 A4 共用通知設定。
新頁先提交永久占用及操作者稽核，再於交易外最多外送一次；提交後及外送前重驗授權／開關。
原頁重播只回已保存結果，缺回條也不重送，不自動跨頁、不換批次鍵重試。
回應在途撤權仍保存加密原件與去敏回條，但回 403 不公開結果。

新頁及占用後外送前的開關拒絕使用上述同一停用訊息；其他頁面範圍錯誤為
`查詢頁面範圍不符。`。占用後停用仍保留原頁，不因重新啟用而取得重送權；
遇到拒絕先 GET 原頁核對，不能僅憑 422 推定沒有占用。

## 讀回

```http
GET /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}
GET /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}
```

只讀本地資料，不外呼或解密原件，不受開關／日期／憑證輪替阻擋，但仍需即時權限。
批次不屬於該付款或頁面不存在回 404；路徑頁碼不合法回 422。請保存 batch_key 供中斷後讀回，勿改鍵。

成功及重播為 200、標準 ApiResponse、`Cache-Control: private, no-store`。`data`：

- `batch`：id、query_date、created_at。
- `coverage`：claimed_pages、received_pages、observed_pages、unknown_pages、unresolved_pages、page_count_conflict、pages_complete、snapshot_consistent、settlement_verified。
- `page`：批次讀回為 null；單頁操作含 id、page_number、claimed_at、receipt。
- `page.receipt`：null 表示尚未確認；否則含 id、outcome（observed／unknown）、failure、provider_status、total_pages、row_count、recorded_at。
  `provider_status` 只在 `failure=provider_error_observed` 時填入 PAYUNi 外層狀態碼，且僅限文件白名單（例 `QUERY03002` 查無符合撥款資料、`QUERY02004` 查詢日期須 30 天內）；未知代碼為 null。空日仍是 unknown，不證明當日沒有入帳。

`pages_complete` 僅表示一致總頁數下的頁碼收齊，**不證明跨頁快照一致、已入帳、已對帳或可撥款**。
snapshot_consistent 與 settlement_verified 固定 false。缺頁、未知、總頁數變動不得下缺帳結論。
不回商家／交易外部識別、批次鍵、原封包、密文、金鑰、費用金額；不修改付款、退款、庫存或撥款。
逐列保存／來源關聯入口見下節；不因關聯而將任何金融狀態改為完成。

本地 Bruno 使用 DemoPayuniGrantQuerySeeder 的合成歷史 unknown，開關維持 false，驗讀回與重播而不外呼。
正向單頁外送以 PHPUnit 合成 dispatcher 驗證；不是封測站或供應商驗收證據。

## 保存原頁逐列觀察

```http
POST /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/materialization
```

不帶 body／query。從固定原頁回條解密、核對長度／摘要、原憑證版本並重新驗簽，全部列與完成紀錄原子保存。
同回條重播只讀完成紀錄，不再解密或查詢。沒有回條、unknown 回條回 422；私有原件／簽章不符不產生部分結果。
只需本地 `settlement.reconcile` 授權，**不需開啟收款或外送查詢開關**；新解析仍需可用原憑證，
已完成解析的歷史重播／讀回不因輪替被阻擋。不是 Lab 原件匯入，也不接受人工列、金額、來源或外部網址。

data 含 id、row_count、recorded_at、settlement_verified（固定 false）；只有本地單頁解析完成。

## 逐列清單與單列

```http
GET /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `page` | 否 | 本地清單頁碼，預設 1；與路徑中的 PAYUNi 原頁碼無關；正整數且不超過 PHP 整數上限除以 100 |
| `per_page` | 否 | 預設 20；正整數，上限依共用規則壓至 100 |

data 為純清單，按原 row_index 遞增；相同 TradeNo 的多列全保留，不依交易號去重。
meta.pagination 為共用分頁，meta.observation 含 receipt_id、materialization（未解析為 null，否則與上述完成回應同形）、
settlement_verified=false。未解析時清單為空，並不代表供應商當天零交易；尚無回條回 422。

```http
GET /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/{rowId}
```

data 為單列。每列只含 id、row_index（0-based）、recorded_at、link、settlement_verified=false。
link 為 null 或 {id, source_id, payment_no, created_at}；payment_no 是已關聯的商城原付款編號，
可能與發起整日查詢的付款不同。沒有外部交易識別、PaymentInfo、未核實費用原字串或原件。
跨付款／批次／頁面套用 rowId、未完整保存或不存在的列回 404，不洩漏其他來源。
路徑頁碼／列 ID 必須是正規正整數，拒絕前導零、浮點或溢位（422）。

## 尋找並關聯原交易

```http
POST /api/platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/{rowId}/link
```

不帶 body／query，不能指定 source_id、交易號或金額。伺服器僅比對已固定原交易來源的
商家身分、環境及兩個交易識別，唯一符合才建立不可變關聯；不要求每列屬於發起查詢的那筆付款。
成功與合法未配對皆回 200，data 為單列：link=null 表示尚未配對，可在來源稍後建立後再次查核。
已關聯的重播只讀原紀錄；未知金額／收支語意不能藉此變成已核實，亦不更新 reconcile_status／payout。

上述兩個 POST 各使用現有每分鐘 10 次節流並拒絕任何額外欄位；所有讀寫都重驗即時財務權限。
等原 session 鎖期間撤權不得解析或關聯；提交後撤權保留已提交紀錄，但不公開回條。
materialized／linked 稽核各隨首次保存原子提交；人工未配對查核另外記 source_missing（每次操作留稽核）。
稽核包含原 admin／時間／IP 與本地參照，不記為 system，也不記私有原值。
全數沿標準 ApiResponse 與 private, no-store；不外呼、不授權退款、對帳結案或撥款。

完整 HTTP 鏈先在 00-payuni-grant-rows 解析未保存的合成原件，後續 05-platform 驗重播、分頁與未配對。
備料沿既有本地通知 session，不改其付款／庫存狀態；實際關聯既存原交易的正向與晚到來源由 PHPUnit 驗證。
