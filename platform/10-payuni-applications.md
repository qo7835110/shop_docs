# PAYUNi 商家申請代建與撤回

spec §S-10.1a、§S-10.1c。需要有效**管理員 token** 及獨立的 `payuni.application.manage`。
會員 token 在認證層拒絕；財務、客服及一般開店審核權不隱含這項權限。

初期可由平台代賣家建案，不強制另一名管理員覆核；但代建權**不等於**核實權，
申請本身不呼叫供應商、不產生可收款資格；核實另走 [受控核實 API](12-payuni-verification.md)。

## 提交既有 MerID

```text
POST platform/shops/{shop}/payuni-applications
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 是 | `sandbox` 或 `production`，使用各自受控 AgentID；缺設定拒絕新案 |
| `candidate_merchant_id` | 是 | 1～20 碼 ASCII 英數、底線或連字號；原值保留，不修剪或改大小寫 |
| `submission_key` | 是 | 正規小寫 UUID，同主體／店面／環境的同鍵同內容回原案 |

## 自動配號的新建店申請

```text
POST platform/shops/{shop}/payuni-applications/new-merchant
```

spec §S-10.1e。初期僅開放平台代建，沿用 `payuni.application.manage`；
此入口不要求第二位管理員，不取得核實權，也**不會交付表單或向 PAYUNi 建店**。

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 是 | `sandbox` 或 `production`；環境與序號互相隔離，缺受控 AgentID 設定拒絕新案 |
| `submission_key` | 是 | 正規小寫 UUID，不修剪；同主體／店面／環境重播回同案 |

伺服器按 AgentID 配置持久化流水號，組成候選 MerID；不每日歸零，已提交配置不回收，
耗盡拒絕。配置只屬本地申請，不保證遠端可用、不建立身分／綁定／支付能力。
成功與重播均為 200；`data` 沿用七欄位白名單，`path` 固定 `new_merchant`，
新案 `status=submitted`。不輸出候選、代理商代號、序號、冪等鍵或表單。

禁止提交 `candidate_merchant_id`、`agent_id`、`allocated_sequence`、`path`、`status`、
`shop_id`、`owner_member_id`、`observed_binding_id`、`observed_binding_version`、
`decision_verification_id`，即使是 null 也回 422。店面只從 URL 取得。
既有 MerID 提交入口仍要求原候選，不會改走自動配號。

新舊途徑共用同店／環境唯一未結案限制：另一未結案請求或跨途徑同鍵回 422，
不能藉新路由繞過。正當重播不重新配號，即使設定移除、換代理商或原案已撤回，
仍回原案與原狀態；每次皆重驗目前權限。撤回後用**新鍵**重提才建立新配置。
新案可由以下既有清單／單案／撤回入口管理，店主亦沿用賣家查閱與撤回權限。

錯誤：未登入或會員 token 為 401；無獨立權限為 403；不存在店面為固定封裝 404；
格式、受控欄位、設定缺漏、未結案／冪等衝突及配號耗盡為 422。
Bruno `05-platform/payuni-new-*` 及 J16 驗證同一案的提出、重播、讀回與撤回；
`06-contract` 驗證會員 token 隔離與財務角色拒絕。這些驗收不呼叫 PAYUNi。

## 撤回

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/withdraw
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `withdrawal_key` | 是 | 正規小寫 UUID |
| `reason` | 是 | 1～1000 個 Unicode 字元，不能全為空白；通過後原樣保存 |

平台可撤回他人代建或店主提出的未結案申請，但不得以同鍵冒充另一操作者已完成的撤回。
即使具平台權限，URL 店面與申請不符仍以 403 拒絕。
若新建店申請已有 A2 交付占用，結果待查時撤回回 422，不得換號重送（§S-10.1f）。
目前交付準備僅為內部核心，尚無公開表單交付入口；新建店提交 API 本身不建立占用。
原占用的正向歸屬查核另走 [核實 API](12-payuni-verification.md)，不能用撤回代替。

寫入端點成功與重播均為 200，共通 ApiResponse 的 `data` 只有 `id`、`shop_id`、`environment`、
`path`、`status`、`submitted_at`、`withdrawn_at`；無外部識別碼或可收款欄位。
欄位語意、受控輸入禁止項、冪等及錯誤碼與 [賣家契約](../seller/10-payuni-applications.md) 相同。
一般輸出不是候選原值查閱或完整核實 UI；證據見 [私有證據 API](11-payuni-evidence.md)，預覽、確認及解除見 [核實與綁定 API](12-payuni-verification.md)。

Bruno 對應 `05-platform/payuni-application-*`，同一條鏈覆蓋提交、重播、異內容、撤回與撤回重播。

## 清單與單案查閱

```text
GET platform/shops/{shop}/payuni-applications
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 否 | `sandbox` 或 `production` |
| `status` | 否 | `submitted`／`withdrawn`／`verified`／`rejected` |
| `page` | 否 | 正整數，預設 1 |
| `per_page` | 否 | 正整數，預設 20，上限 100 |

只列 URL 指定店面的本地申請，依 ID 由新到舊排序；`data` 陣列及 `meta.pagination` 分離。

```text
GET platform/shops/{shop}/payuni-applications/{payuniApplication}
```

回上述七欄位；終態可查，不提供候選原值／證據。兩種查閱同樣要求 `payuni.application.manage`，
平台權限不允許把 A 店申請放在 B 店 URL 下取用；巢狀錯配為 403。
