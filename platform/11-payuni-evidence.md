# PAYUNi 私有證據

spec §S-10.1b、§13.7。僅有效平台管理員 token 可用；賣家、工會、財務預設無權。
`payuni.evidence.manage`（收件／作廢）與 `payuni.evidence.read`（列表／下載）互不隱含。
URL 店面、申請、版本須一致；錯配 403、不存在 404。每次重播仍重新授權。
收件不等於核實或開通收款，不呼叫 PAYUNi。`submitted` 可收三種用途；`verified` 僅可新增
`payment_settings`，供後續付款設定審查；其他終態不得新收件。原操作同內容重播仍可讀回。

## 收件

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence
```

使用 `multipart/form-data`，不是檔案路徑、URL 或 base64 JSON。

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `purpose` | 是 | `provider_record`、`owner_authorization` 或 `payment_settings` |
| `source_kind` | 是 | 歸屬資料／付款設定搭配 `provider_portal`／`provider_document`；授權文件只接受 `owner_document` |
| `file` | 是 | 實際 PDF／JPEG／PNG；伺服器辨識 bytes，大小上限由 `payuni_evidence_max_kb` 決定（預設 5120 KB） |
| `acquired_at` | 是 | 嚴格 `Y-m-d H:i:s`，**UTC**（不轉換台北時間，見 [conventions](../conventions.md#時間)），來源取得時間；不是系統核實時間 |
| `valid_until` | 否 | 嚴格 `Y-m-d`；未提供或 null 代表未聲稱期限 |
| `idempotency_key` | 是 | 正規小寫 UUID；同主體、同案、同鍵及相同內容／metadata 重播回同一版本 |

成功及重播 200。`data` 僅有 `id`、`application_id`、`shop_id`、`environment`、`purpose`、
`version`、`registered_at`；同鍵異內容 422。不含物件鍵、摘要、原始檔名、IP 或原件。
不自動修剪 UUID／日期；來源與效期是登錄者聲稱，不保證內容真實或無惡意程式。

`payment_settings` 用於信用卡、3D、自動請款等設定原件，不能選入商家歸屬核實集合。
其版本與作廢獨立留存，不覆寫已結案的決定或綁定；收件未核實設定值、未建立採用紀錄，
也不直接放行付款。即使舊綁定已解除仍可補充歷史原件，不表示能用於目前或其他商家的收款。

## 版本列表

```text
GET platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `page` | 否 | 正整數，預設 1 |
| `per_page` | 否 | 預設 20，上限 100 |

限定同案、按 ID 新到舊；`data` 為陣列、`meta.pagination` 為分頁。
各列包含收件回條七欄，加上 `owner_member_id`、`source_kind`、`acquired_at`、`valid_until`、
`byte_size`、`mime_type`、`registered_by_type`、`registered_by_id`、`invalidated`、
`invalidated_at`、`invalidated_by_type`、`invalidated_by_id`、`invalidation_reason`。
無作廢時後四欄為 null；不回原件、私有路徑、SHA-256 或冪等鍵。

## 作廢

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence/{payuniEvidence}/invalidate
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `invalidation_key` | 是 | 正規小寫 UUID |
| `reason` | 是 | 1～1000 個 Unicode 字元，不得全空白；通過後原樣保存 |

成功及同主體同內容重播 200；`data` 為 `id`、`evidence_version_id`、`invalidated_at`。
不同鍵、理由或冒充他人重播為 422。只新增作廢事實，原件／版本保留且仍可受控查閱。
已結案申請的證據也可記錄作廢，不會自動改寫既有核實或綁定。

## 受控原件下載

```text
GET platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence/{payuniEvidence}/download
```

成功為 **200 二進位，不包 ApiResponse**；錯誤仍是共通 JSON 信封。
先提交當下授權／讀取稽核，再有界讀取及核對大小、SHA-256；缺件／被替換拒絕交付。
`Content-Type` 採已驗格式，`Content-Disposition: attachment` 使用伺服器產生的安全檔名；
`Cache-Control: private, no-store, max-age=0`、`X-Content-Type-Options: nosniff`、
`Content-Security-Policy: sandbox; default-src 'none'`。不提供公開或時效性旁路 URL。

Bruno `05-platform/payuni-evidence-*` 以同案合成 PDF 驗證收件、重播、衝突、列表、下載與作廢；
整鏈腳本為每次執行建立獨立私有目錄，不使用開發用原件。
