# PAYUNi 既有商家申請

spec §S-10.1a、§S-10.1c。只處理本地申請與撤回，不呼叫 PAYUNi；申請通過格式驗證
不代表核實、綁定或可收款。這也不是新商家 A2 建店入口。

需要有效會員 token、店主身分及該店 `shop.manage`；交易暫停不阻擋受控申請管理。
平台可代建，賣家也可撤回平台代建的本店未結案申請；不同操作者不能冒充原撤回的重播。

## 提交

```text
POST seller/shops/{shop}/payuni-applications
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 是 | `sandbox` 或 `production`，兩環境獨立；目前只確認 sandbox 已開通 |
| `candidate_merchant_id` | 是 | 1～20 碼 ASCII 英數、底線或連字號；保留大小寫，前後空白直接拒絕 |
| `submission_key` | 是 | 正規小寫 UUID；同鍵同內容回原案，即使原案已結案 |

AgentID 從該環境受控設定取得，缺設定以 422 拒絕新案，既有同內容重播仍可回原案。
不接受非空的 `agent_id`、`path`、`status`、`shop_id`、`owner_member_id`、綁定觀察或決定欄位。
提交不可改寫；同店／環境只容許一件未結案申請。填錯時先撤回，再用新鍵提出新案。

成功及重播均為 **200**，使用共通 ApiResponse。`data` 僅有：

| 欄位 | 說明 |
| --- | --- |
| `id`、`shop_id` | 本地申請及店面識別碼 |
| `environment`、`path` | 環境及途徑；本入口提交固定 `existing_merchant`，查閱／撤回平台代建的新案可能為 `new_merchant` |
| `status` | `submitted`／`withdrawn`／`verified`／`rejected`；重播回目前終態 |
| `submitted_at`、`withdrawn_at` | ISO 8601 時間；未撤回時後者為 null |

一般回應不含 MerID、AgentID、冪等鍵、IP、理由原文、證據或支付資格欄位。
前端保存本地申請 ID 與自己送出的冪等鍵以供重試；目前沒有原值受控查閱或證據端點，
不得把此回應當成已可完成整份人工核實操作。

## 撤回

```text
POST seller/shops/{shop}/payuni-applications/{payuniApplication}/withdraw
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `withdrawal_key` | 是 | 正規小寫 UUID |
| `reason` | 是 | 1～1000 個 Unicode 字元，不得全為 Unicode 空白；原樣保存，不修剪 |

成功／同操作者同鍵同理由重播為 200，回應同上述七欄位。巢狀申請與店面錯配以 403 拒絕。
撤回只收回申請，不解除既有綁定；核實／拒絕終態不能再撤回，回 422。
新建店申請若已有 A2 交付占用，結果待查時同樣回 422，不得撤回再換號（§S-10.1f）。
目前沒有公開表單交付入口，提交本地申請不會建立此占用。

未登入或跨 guard token：401；無權或跨店：403；不存在：固定 404；格式、同鍵異內容或狀態衝突：422。

Bruno 對應 `03-seller/payuni-application-*`；只能對測試資料操作，合成候選不作真實歸屬證據。

## 清單與單案查閱

```text
GET seller/shops/{shop}/payuni-applications
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 否 | 限定 `sandbox` 或 `production` |
| `status` | 否 | `submitted`／`withdrawn`／`verified`／`rejected` |
| `page` | 否 | 正整數，預設 1 |
| `per_page` | 否 | 正整數，預設 20，超過 100 壓至 100 |

依本地申請 ID 由新到舊排序，`data` 為上述七欄位的陣列；分頁位於 `meta.pagination`。
同樣需要店主與本店 `shop.manage`，不是只要有會員 token 就能查。

```text
GET seller/shops/{shop}/payuni-applications/{payuniApplication}
```

回傳上述七欄位。可查終態歷史；不修改狀態、不回吐候選原值、不列證據或收款資格。
不存在以固定 404 回應，跨店或巢狀識別不相符以 403 拒絕。

平台以 [自動配號入口](../platform/10-payuni-applications.md#自動配號的新建店申請) 代建的
`new_merchant` 申請，同樣可由具本店管理權的店主查閱與撤回；欄位白名單及遮蔽規則相同。
賣家沒有新建店提交路由，不能用現有提交入口指定途徑或要求重新配號。
新舊途徑共用同店／環境唯一未結案限制；撤回不回收已配置號碼，也不直接解除綁定。
