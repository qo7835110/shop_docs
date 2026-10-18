# PAYUNi 付款設定判讀

spec §12.1。有效平台管理員須同時具 `payuni.binding.verify`、`payuni.evidence.read`；
不要求第二人覆核，不需要解除綁定權。會員 token 在認證層拒絕。
支援文件判讀，或 sandbox 管理員本人直接確認（免附文件／另行審查）；
兩者都不是供應商 API 自動驗證，更不是可收款旗標。
成功回應皆 `Cache-Control: private, no-store`。

## 查閱最新審查

```text
GET platform/shops/{shop}/payuni-bindings/{payuniBinding}/payment-settings
```

沒有審查時 `data=null`，不能推定啟用。有審查回最新一筆（不退回舊成功），
含下節回條所有欄位及 `selected`（按原件 ID 遞增的 `{id, version}` 陣列）。
若有未封存的內部歷史，status 為 preparing 仍原樣回查，不拿前一筆 recorded 替代。
過期、原件作廢或綁定解除後仍可查歷史；本 GET 不檢查收款資格、不讀原件、不外呼，
也不建立查核工作、付款或原件讀取稽核。讀回 true 不表示當下能用。
取得的 `data.id` 用作提交的 `observed_review_id`；data=null 時明示 null，不可省略。

## 留存設定判讀

```text
POST platform/shops/{shop}/payuni-bindings/{payuniBinding}/payment-settings
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `administrator_confirmed` | 否 | JSON true 表示 sandbox 管理員直接確認；省略／false 採文件模式，不接受字串、0／1 或 null |
| `selected` | 是 | 直接確認明示空陣列 `[]`；文件模式為 1～20 份 `payment_settings` 原件，含目前最新版本；禁止混用 |
| `selected[].id` | 是 | JSON 正整數，不可重複；子物件只接受 id／version |
| `selected[].version` | 是 | JSON 正整數，與原件 ID 相符 |
| `observed_review_id` | 是 | 查閱時的最新審查 JSON 正整數；尚無審查明示 null；不接受空字串 |
| `credit_enabled` | 是 | 信用卡一次付清的 JSON true／false；不接受 0／1 或字串 |
| `three_d_enabled` | 是 | 3D 的 JSON true／false |
| `auto_capture_enabled` | 是 | 自動請款的 JSON true／false |
| `grant_enabled` | 否 | 撥款指示能力的 JSON true／false；省略為未知，不沿用前次啟用，不接受 null、0／1 或字串 |
| `valid_until` | 是 | UTC `YYYY-MM-DD HH:mm:ss`，晚於審查時刻、不超出原件已知效期；無默認期限 |
| `review_key` | 是 | 正規小寫 UUID；同次重播保留原鍵、內容與 observed_review_id |
| `reason` | 是 | 1～1000 字非全空白理由，原樣留存於審查與稽核 |

拒絕多餘欄位；所有字串不修剪。採用截止是本次本地有效上限，不冒稱 PAYUNi 文件效期。
直接確認順序：核對 sandbox 原綁定與設定 → 查最新判讀 → 明示
`administrator_confirmed=true`、`selected=[]`、三項設定、理由、截止及畫面版本。
不需上傳／下載文件，也不建立原件讀取稽核；正式環境拒絕此模式。
文件模式操作順序：原申請受控查閱商家原值 → 收件／逐份下載確認設定原件 → 查最新判讀 →
明列原件版本、三項判讀與截止提交。若畫面審查 ID 過期，須重新核對而非自動換 ID 重送。
原件必須屬於綁定原申請、店面、環境及目前店主；原綁定須生效，原申請 verified。
讀檔前後各自鎖後重驗權限及依據；讀檔期間新審查提交亦拒絕舊工作晚到覆蓋。

成功／合法重播均 200，共通信封 `data` 恰有：`id`、`binding_id`、`application_id`、
`shop_id`、`environment`、`status`（recorded）、`confirmation_basis`（documents／admin_confirmation）、`credit_enabled`、`three_d_enabled`、
`auto_capture_enabled`、`grant_enabled`（boolean 或 null）、`recorded_at`、`valid_until`；時間均 UTC 秒精度字串。
不回傳理由、操作者／店主、IP、工作鍵、摘要、私有路徑、MerID、金鑰或 payment_ready。
否定也留存並成為最新判讀，不可默默退回舊成功。三項信用卡設定決定付款設定因素；
`grant_enabled` 獨立判讀，不影響既有信用卡因素，亦不能由三項全 true 推定。
省略時舊請求的摘要與合法重播保持不變；同鍵新增或更改 grant 值是不同內容，回 422。

同人同鍵同內容僅回原歷史，不重讀原件／重複判讀；撤權後連重播也拒絕。
已提交後回應遺失，保留完整原請求重播；變更內容／操作者或畫面版本不是重播。
格式、過期、錯件、作廢、舊畫面及重播衝突回 422；無登入／會員 token 回 401；
缺任一權限／跨店綁定回 403；店面／綁定不存在回 404。失敗遵循共同錯誤封裝。

## 邊界

spec §14.9.1：grant 確認僅是撥款設定因素。採用須在原交易持鎖範圍重讀最新封存判讀，
重驗原綁定／店主／來源與期限；false、未知、過期皆拒絕。不是 A3 的 API 核實結果，
不證明原交易現在可撥金額或已入帳，也不會啟用外送或移除五項撥款條件。
既有記錄保留 null，不將過去信用卡確認自動改為 grant 開通。sandbox 沿用本人確認，不要求補圖片。

未新增金流外呼、沒有付款交付或入帳；本 API 不修改訂單、庫存、費率或工會分潤。
新付款交付占用會強制保存原設定確認參照；既有付款交付 Service／買家表單仍須通過其他守衛，
A3、憑證、付款契約及可靠結果接收仍各自必要。舊占用缺參照不可回填成功或釋放重送。
Bruno 沿既有合成核實鏈造出可到達狀態，沒有新增不可由 API 取得的 seed。
