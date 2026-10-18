# PAYUNi sandbox 契約實驗

spec §S-10.1h。預設關閉，需 `PAYUNI_CONTRACT_LAB_ENABLED=true`，只讀 sandbox 憑證。
僅有效的平台管理員且具有 `payuni.credential.manage` 可操作；財務預設無權限，會員 token 為 401。
不建立商城商家／訂單／付款／庫存。建店／付款由使用者另外開啟表單手動送出；
明示 `query-trade` 操作會由封測站對原 UPP 交易發動一次 sandbox M3 唯讀查詢；
原結果缺欄位時可另明示 `diagnose-trade`，獨立一次取證，並非自動重試。
詳細測試案例、還原及 Log 交付見 [操作手冊](../../ops/payuni-contract-lab.md)。
可直接用 [封測 Bruno 請求集](../../../tests/BrunoPayuniSandbox/README.md) 操作，不必手填 API body。

## 建立實驗

```http
POST platform/payuni/contract-lab
```

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `kind` | string | 是 | `a2`、`upp`、`upp-grant` 或 `upp-refund`；grant 另需 payout.execute，refund 另需 refund.process |
| `signer` | string | 是 | `agent` 或 `merchant`；a2／upp-grant 只可 agent，upp-refund 只可 merchant |

拒絕其他欄位，不可指定金額、環境、MerID、金鑰或回呼目的地。
201 的 data 為 `run_id`、`form_url`、`a4_url`、`notify_url`。網址是秘密能力權杖，24 小時有效，
不可貼入客服紀錄或共享報告。**每次建立都是新 run**，不應自動重試；建立本身沒有遠端副作用。
同封包／訂單重送須使用原 run，不再建立另一筆。管理回應採 private/no-store。
上句的故意重送只適用舊實驗；`upp-grant`／`upp-refund` 新表單只交付一次，第二次 GET 為 410，
禁止 refresh-time。瀏覽器已取得的表單不可重送；首次 GET 回應失聯也不重新交付。

## 操作原實驗

```http
POST platform/payuni/contract-lab/{labRun}/actions
```

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `action` | string | 是 | `ack`、`refresh-time`、`note`、`close`、`query-trade`、`diagnose-trade`、`verify-query-parser`、`query-capture`、`query-grants`、`grant-once`、`refund-once`、`query-refund` |
| `confirmation` | string | grant-once／refund-once 時 | 分別固定 sandbox_grant_ntd10_once／sandbox_refund_ntd10_once，不可互換；其他 action 禁止 |
| `query_date` | string | query-grants 時 | 有效 YYYY-MM-DD；其他 action 禁止 |
| `page` | integer | query-grants 時 | JSON 正整數，不接受字串或浮點；其他 action 禁止 |
| `query_key` | string | query-grants／query-refund 時 | 正規小寫 UUID；同次重播沿用，其他 action 禁止 |
| `channel` | string | ack 時 | `a4` 或 `notify`；其他 action 禁止 |
| `mode` | string | ack 時 | `fail`=503/空白、`empty`=200/空白、`success`=200/SUCCESS、`one`=200/1、`ok`=200/OK |
| `case` | string | note 時 | 固定案例：a2_created、a2_same_form、a2_interrupted、upp_paid、upp_same_form、upp_after_ten_minutes、credit_enabled、three_d_enabled、auto_capture_enabled、portal_ack_received、portal_no_extra_trade |
| `outcome` | string | note 時 | yes、no、unknown |

200 data 為 run_id/action。ack 重播只重選回應；close 重播仍關閉，不清除紀錄。
refresh-time 僅保留原 UPP 訂單號／金額並更新 Timestamp；不提交表單。
note 僅為人工觀察，並非供應商證據。操作及操作者 ID 記錄在私有去敏 Log。

`grant-once` 是財務寫入實驗而非唯讀查詢，僅限新建 `upp-grant`、已交付一次表單且 run 已關閉。
另需目前有效的 `payout.execute`，建立／查核前／grant 前／回應前重新驗權，不自動補權限。
先持久化一次占用，再查核原交易：唯一、TWD10、完整已付款、信用卡 UPP 一次付清、
請款成功且 CloseAmt／RemainAmt 均 10、無退款跡象。任何缺漏、未知或不符停止，
結果見 `grant_preflight_result` 及 `grant_once_result`，200 只代表已記錄，不代表 grant 成功。
重播只讀原結果、必要時補去敏 Log；不再 M3 或 grant。回條保存失敗或僅有 started 都保持待查。
`grant_accepted`／SUCCESS 只是受理；後續用 query-grants 觀察入帳，不更新商城 payout／費用或銀行狀態。
不得在實驗期間從後台另發退款／撥款；此操作不提供遠端互斥或冪等保證。
grant 未支援 NotifyURL，不添加欄位；新 UPP 付款仍照帶逐筆 NotifyURL。

`query-trade` 僅限 UPP，從原 run 取得商家、交易短碼及 agent／merchant 簽署方式，
不接受外部識別或金額。先持久化一次查詢占用，再於檔案鎖外單次呼叫 M3；
同 run 重播只回操作受理，不再次外呼，逾時／中斷不自動重試。
已關閉／過期 run 仍可查詢，不重開表單或回呼；原設定身分／金鑰摘要不符拒絕外送，
A2 不支援（503）。不依此寫入商城財務或允許重付，Notify URL 不需修改。
200 不是查詢結果：請讀 report 的 `trade_query_result`，`matched` 只代表唯一列的
原商家／訂單／TWD 10／信用卡 UPP 關聯相符。保留 trade_status、data_source、payment_type、
gateway 及有條件的 trade_fee；只有 A＋已付款＋確有費用才保存數值，否則 null，不能當成零元。
簽章／關聯不符、多列、查無及傳輸失敗均 unknown；不提供退款／撥款完成證據。

`diagnose-trade` 只在原 `query-trade` 已有唯一一筆 `unknown / missing_envelope_field`
結果時可用，否則 503 且不外送。另持久化一次診斷占用，沿用原商家、訂單、憑證及所有核對規則；
不刪除原占用、不覆寫原結果、不因診斷成功而入帳。重播、提交後逾時／程序中斷都不重送。
新事件為 `trade_diagnostic_started`／`trade_diagnostic_result`；
缺欄位結果的 `missing_field` 只記固定 enum：`envelope.Status/MerID/Version/EncryptInfo/HashInfo`、
`plaintext.Result`、`row.Status/TradeStatus/DataSource/TradeNo/TradeAmt/PaymentType/Gateway`
（斜線表示不同候選欄位），其他失敗或舊結果可能為 null／未提供。
這是解析器第一個缺少或空值的必要欄位位置，不是完整欄位盤點，不含原文、任意鍵名或值。
不跳過外層版本／商家、驗簽解密及逐列核對；外層缺欄位時不能據此宣稱已驗簽。

`verify-query-parser` 是新版解析器的固定一次驗收，不是按版本號無限重查。
只接受原 query 唯一 `unknown / missing_envelope_field` 且原診斷唯一同錯誤、
`missing_field=row.Status` 的 UPP run；缺結果／其他失敗／重複結果均 503，零外送。
另須已載入 reader 的語意版本 `m3-signed-status-v2`；不能只靠磁碟檔案雜湊放行。
占用、原交易、憑證摘要、關閉後不重開、鎖外送出、重播／中斷不重送與結果核對規則同上。
原 query／diagnose 事件不改；新增 `trade_parser_verification_started`／`trade_parser_verification_result`，
兩者保存 `query_parser_revision`（載入版本）與 `query_reader_sha256`（開始時磁碟檔案 SHA-256）。
200 仍只表示操作受理，不寫訂單／付款事件或解除重付限制，不驗 Notify URL。

### 原交易的請款觀察

`query-capture` 限已關閉／過期 UPP run，且先前 `trade_query_result`、`trade_diagnostic_result`
或 `trade_parser_verification_result` 至少有一筆 matched。未符合為 503，零外呼。
需載入 `credit_capture_revision=m3-credit-capture-v1` 及
`credit_refund_revision=m3-credit-refund-observation-v1`，另以 `query_reader_sha256` 核對部署；
不能只憑磁碟檔案已換版判定 OPcache 已更新。
每 run 固定一次 `credit_capture_query_started` 占用，鎖外唯讀查原交易；
`credit_capture_query_result` 追加結果，不改原三種查詢，重播／中斷不再次外送。

matched 結果新增 `authorization_type`（AuthType）、`close_status`（CloseStatus）、
`close_amount`（CloseAmt）：僅信用卡碼表，缺漏／空字串／破折號保留 null；零金額原樣保留，
不能補成全額請款。CloseStatus：1 申請中、2 成功、3 取消、7 處理中、9 未申請。
仍須分別看 TradeStatus、DataSource、原金額及工具；這不是自動入帳、退款成功或重付許可。
報告不含卡片、授權碼或供應商原文；不操作請款、不重新付款、不改 Notify／ACK。
snapshot 新增 `credit_capture_query_attempted`；true 無結果仍為未知，不能清占用重送。
同次 matched 結果亦保存信用卡 `refund_type`（2 退款／3 預計退款）、`refund_status`
（1 申請中／2 成功／3 取消／8 處理中）、`refund_amount`、`refund_day`、`remaining_refund_amount`。
日期僅保存供應商原日期文字，不假定時區，不當成退款完成時間；缺漏為 null，合法零元保持 0。
M3 只提供信用卡最後一筆退款，不能據此認領任一本地退款單、補造外部退款 ID 或認定完整歷史。
DataSource B／日期缺漏／剩餘額度不足等仍須另核對，任何觀察不授權退款送出。
本次沒有新增 action 或重設占用；舊版已執行過者只交回原 Log，不清除占用再跑。
操作步驟見 [請款取證工作單](../../../plans/31-3-credit-capture-observation.md)。

### 原商家的撥款單頁查詢

`query-grants` 只對已關閉／過期、且先前 M3 有 matched 的原 UPP run 查詢 sandbox
`trade/grant_query`；沿用原 signer／商家與憑證摘要，不接受覆寫商家、簽署者、金鑰或 URL。
Bruno 操作直接編輯 JSON Body 的 query_date／page／query_key；不再以 grant_query_* 環境或請求變數覆寫。
新查詢日期限台北今天及前 29 日。輸入格式錯誤為 422；原 run／憑證／窗口或儲存不符為 503，不外送。
不得在呼叫端 DB 交易內執行。這是原商家的整頁觀察，可能含其他交易，不宣稱只含原 UPP 訂單。

每個 run + query_key 先以私有不可覆寫 claim 檔保存日期／頁碼，再於檔案鎖外送一次。
同鍵相同範圍只讀回／補 Log；同鍵改日期或頁碼拒絕。只見 started 無 result 表示未知，
不能清 claim 或換鍵盲目重送。不同頁面須另以明示新鍵取證，目前不自動循頁或重試。
原實驗關閉狀態、付款表單與 Notify／ACK 均不改變，不寫商城財務。

HTTP 200 原始 body（包含解析 unknown 者）以 0600 私有檔保存，Log 只記摘要／大小及去敏觀察。
傳輸失敗／截斷沒有可用 body，不造假原件。此為實驗證據，不是正式財務觀察資料表或對帳來源。
claim／結果檔採獨占建立及檔案 fsync；沒有目錄 fsync，不宣稱斷電後仍有完整 crash-durable 協定。
已保存結果而 Log 缺失時，同鍵可補 Log，不重新外送。

事件 `grant_query_started`／`grant_query_result` 帶 query_key、query_date、requested_page、
grant_query_revision／grant_query_reader_sha256。結果另有 classification、failure、observed_status、
current_page、total_pages、row_count、response_saved／response_bytes／response_sha256。
batch_complete、settlement_verified、payment_ready、acceptance_complete 固定 false。
200 只是操作受理，accepted 只是受限形狀的已驗簽單頁，不代表已入帳、撥款完成或整批查齊。
空日／未知分頁形狀保留 unknown；原始檔不得貼入回報或從公開路徑下載。
snapshot 可先核對 grant_query_revision=`grant-query-page-observation-v2`，不憑檔案 SHA 推定 OPcache 已更新。
v2 補齊撥款查詢文件的 10 個 QUERY 狀態碼白名單，新結果可保留 observed_status；
其他未知代碼仍為 null，不輸出供應商 Message。外層錯誤仍是 unknown／provider_error_observed，
不是已驗簽明細或完整空日證據。此端點 QUERY03001 為網路異常、QUERY03002 為查無符合撥款資料，
不得沿用其他查詢 API 的同碼解釋，也不授權重送 grant。舊回條／歷史 null 不回填，
同鍵重播沿用保存結果，升版不新增外送機會。

對新 `upp-grant`，部署單頁關聯摘要後，snapshot／查詢事件另帶
`grant_query_summary_revision=grant-experiment-page-summary-v1` 與 `grant_query_summary_sha256`。
新查詢結果多一個 `original_trade`：`status` 為 source_unavailable／page_unknown／
not_observed_on_page／ambiguous_on_page／matched_on_page；matched_row_count 可為 null，
僅單頁唯一且 MerID、MerTradeNo、原私有 grant 來源 TradeNo 三者全符才提供 row_index 與
execute_amount／fee_amount／funds_amount（整數或 null）、amounts_are_integers（布林或 null）。
金額須為 PHP／JavaScript 共同安全範圍的正規整數；超界、小數或非正規格式保留 null，不能當成零。
direction 只將文件的入帳／出帳映為 credit／debit，其餘 unknown，不輸出原 Type／Action 或識別碼。
scope 固定 page_only、settlement_verified 固定 false；不加總多列，不把未觀察到視為沒有入帳。
來源缺失／損壞／symlink 只回 source_unavailable，仍保存查詢回條；相同 query_key 不外呼。
舊實驗不附此摘要，已保存的歷史結果不回填或改寫；新版不能成為換鍵重查或重新 grant 的理由。

### 單筆退款實驗

`upp-refund` 是專用新商家簽署 NT$10 UPP，不帶 GrantExport；不升級舊 run、不動等待撥款交易。
建立、`refund-once`、`query-refund` 需目前有效的 payuni.credential.manage + refund.process；
外呼前及回應前重驗，未授權 403，不自動補權。不得在 DB 交易內外呼。
先選逐筆 Notify ACK，再人工付款，確認請款後 close；不設定共用 Notify URL。

`refund-once` 必須表單已交付、run 已關閉、建立未滿 24 小時；先永久占用，再 M3 查核唯一原
信用卡一次付清、DataSource A、已付款 10、請款成功 10、RemainAmt 10、無退款跡象，
才保存私有 TradeNo 並送一次 M4 CloseType=2／TradeAmt=10。外送前再驗期限，不接受外部識別或金額。
M4 不帶 NotifyURL；簽章與原商家／TradeNo／CloseType 核對成功僅為 refund_accepted。
Log 為 refund_once_started、refund_preflight_result、refund_once_result；未知／逾時均不重送。
200 僅表示已記錄，不表示退款受理或成功。重播只回原結果，必要時補 Log；不得刪占用。

`query-refund` 不要求原 M4 結果已保存，但必須有原 claim 及已保存來源。
Body 僅 action/query_key，不能帶日期、頁碼、金額或新交易號。每 run 最多 10 個新查詢、至少
間隔 60 秒；人工檢視前次結果後才明示另查，不自動輪詢。相同 key 重播不外送，也不受新查詢間隔影響。
查詢原訂單且核對保存的 TradeNo；完整已付且請款 10，RefundType=2／RefundStatus=2／
RefundAmt=10／RemainAmt=0 且 RefundDay 有值，記 refund_success_observed，否則保持未完成／不符／未知。
M3 僅最後一次退款，不宣稱完整歷史或商城退款完成。日期保留原文字，不猜時區。
事件 refund_query_started/refund_query_result；snapshot 帶 refund_experiment、refund_attempted、
refund_query_count、refund_experiment_revision=refund-experiment-v1、refund_experiment_sha256。
refund_completion_verified、payment_ready、acceptance_complete 固定 false；原始卡片、交易識別、
商家號及 Message 不輸出，不修改商城財務或庫存。操作者不得同步從後台另行退款／取消／撥款。
儲存採檔案 fsync 及互斥鎖，但未提供目錄 fsync，不宣稱斷電後完整 crash-durable。

## 取得去敏報告

```http
GET platform/payuni/contract-lab/{labRun}
```

200 data.report 為 JSONL 字串，包含 run ID、時間、驗簽結果、原內容 HMAC 指紋、回覆選擇、人工觀察及操作人員 ID。
可在 close 後取得；總開關須仍開啟。報告不含 Key／IV、原通知、表單權杖或完整商家／交易號。
payment_ready、acceptance_complete 固定 false；沒有重送不等於 ACK 契約已確認。
snapshot 的 `trade_query_attempted` 表示已取得一次查詢權，不保證外送或保存結果成功；
true 而沒有 `trade_query_result` 必須保留未知，不可刪除占用或另造付款重查。
`trade_diagnostic_attempted` 同樣只表示已占用人工診斷；沒有結果時不得再外送。
`trade_parser_verification_attempted` 表示新版解析器驗收已占用，意義同上。
snapshot 另含 `query_parser_revision`（舊載入版本可能為 null）及 `query_reader_sha256`；
前者不能由後者推定，檔案同步不保證 OPcache 中的舊類別已失效。唯讀 GET 不呼叫 M3。

未登入 401、無權限 403、關閉開關／不符 run 路徑格式 404、輸入錯誤 422、缺憑證／run 不存在／儲存不可用 503。
503 不回傳內部例外，勿以自動重送建立另一實驗。

外部入口位於 `/__payuni_contract_lab/{labRun}/{labToken}/{channel}`，不是管理 API。
form 為 GET，a4／notify／return 為 POST；直接回原始 HTML／ACK，無 session cookie 或登入重導。
這些入口僅觀察，不向正式付款服務傳送事件。
