# 13 PAYUNi 憑證版本管理

spec P-10／§12.2／§15.3。所有端點限有效平台管理員且即時持有 `payuni.credential.manage`，
初期僅平台管理者具有此權，財務／客服預設沒有。會員 token 在認證層回 401。
這是本地版本保存／撤銷，不外呼 PAYUNi；不修改部署設定、不授權付款，也不代表供應商開通或停用。
成功回應 `Cache-Control: private, no-store`。沒有秘密上傳、下載、解密讀取或復活端點。

## 1. 已保存版本清單

```
GET platform/payuni/credential-versions
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `environment` | string | — | `sandbox`／`production`；省略列出全部本地版本 |
| `page` | integer | — | 從 1 開始 |
| `per_page` | integer | — | 最小 1，最大採用 100 |

回應 200；`data` 清單按 id 新到舊、分頁在 `meta.pagination`。
單筆只包含 `id`、`version`（非秘密 UUID）、`kind`（目前 agent）、`environment`、
`created_at`、`revoked_at`（未撤銷為 null）。不列 AgentID／MerID、Key／IV、密文、操作者或理由。
此清單不是目前部署設定預覽；版本是否 active 或可用仍由每次解析核對，沒有 `can_pay` 旗標。

## 2. 登錄目前部署版本

```
POST platform/payuni/credential-versions
```

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `expected_environment` | string | 是 | `sandbox`／`production`，須與目前受控部署環境相同 |
| `expected_version` | string | 是 | 正規小寫 UUID，須與該環境的受控代理商版本相同，不修剪前後空白 |
| `reason` | string | 是 | 最長 1000、不可全空白，原樣保存；只填作業理由，禁止貼入秘密 |

拒絕任何額外欄位，不接受透過 API 提供金鑰。Key／IV 只能由受控部署設定取得，
在既有 Vault 中加密保存；版本、稽核、操作者與 IP 同交易建立。
回應 201，`data: { "id": 1 }`。同 UUID／相同憑證的重播回原 id，不重複保存或新增稽核；
同 UUID 換金鑰、錯環境、缺受控設定、已撤銷皆拒絕，不能用重新登錄復活。
回應遺失先依 UUID 查清單；不得為了重試任意另產新版本。

## 3. 本地撤銷指定版本

```
POST platform/payuni/credential-versions/{credentialVersion}/revoke
```

`{credentialVersion}` 是清單或登錄回傳的正整數 id，不是 UUID。

| 欄位 | 型別 | 必填 | 規則 |
| --- | --- | --- | --- |
| `expected_environment` | string | 是 | 核對指定版本的原環境，不要求等於目前部署環境 |
| `expected_version` | string | 是 | 指定版本的正規小寫 UUID，須與鎖後原紀錄相同 |
| `reason` | string | 是 | 最長 1000、不可全空白；同一操作重播須使用原理由 |

回應 200，`data: { "id": 1, "revoked": true }`。
即使目前設定已移除或輪替，仍可撤銷舊版本；撤銷與稽核原子提交。
相同操作者／相同理由重播回原結果，不新增撤銷；不同操作者或理由回 422，不改寫原事實。
撤銷不可逆，不刪版本、不修改原付款，亦不能回收已送至瀏覽器／供應商的內容。

## 錯誤

- 401：未登入或拿會員 token。
- 403：無目前管理權限、帳號失效；不揭露目標是否存在。
- 404：有權限但指定 id 不存在，或路徑 id 格式不合法。
- 422：輸入不合法、環境／版本核對不符、部署憑證不可用或撤銷重播衝突；不回聲秘密或內部例外。
- 500：查詢或撤銷結果無法確認時只回安全錯誤；先讀清單查證，不刪資料或換號重試。

部署作業與限制見 [操作手冊](../../ops/payuni-credential-management.md)。
