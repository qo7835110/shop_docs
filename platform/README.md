# 平台後台 API

原付款 M3 人工查核見 [17 原付款查核](17-payuni-payment-query.md)，需 `settlement.reconcile`；只保存觀察，不入帳或授權重付。

PAYUNi [一次表單交付](../buyer/04-orders-and-payment.md#3b-payuni-一次表單交付)僅原下單會員可用；平台 token 不得代付，前景返回不表示付款成功。

PAYUNi [原付款觀察內部採用](../../../plans/31-3-payment-observation-adoption.md)沒有公開 API／排程；
管理員查詢或執行 lab 不會自動入帳，lab 的觀察也不能補貼到商城訂單。

原 A2 交付紀錄可透過 [核實 API](12-payuni-verification.md) 明列 `delivery_claim_id` 進行正向歸屬查核；不等於遠端建店或可收款。

平台可代建 [A2 自動配號本地申請](10-payuni-applications.md#自動配號的新建店申請)，
仍須獨立的 `payuni.application.manage`；不手填 MerID，也不代表已向 PAYUNi 建店或可收款。

付款讀回的 `payuni_query` 只公開最新本地查核摘要，沿用同一會員授權；matched 不是已付款，GET 不發動金流查詢。

[買家付款讀回](../buyer/04-orders-and-payment.md#3a-查核本地付款) 不接受平台管理員 token（401），不可用平台權限繞過會員身分隔離。

路由前綴：`platform/`（完整路徑為 `/api/platform/...`）
認證：平台管理員 token（`Authorization: Bearer {admin_token}`）

> 先讀 [共通約定](../conventions.md)。回應信封、分頁、錯誤碼、遮罩規則都在那裡，本目錄不再重複。

## 文件分類

| 文件 | 內容 | 對應 spec |
| --- | --- | --- |
| [01 認證與帳號權限](01-auth-and-accounts.md) | 登入／登出／目前身分、管理員帳號、角色與權限授予 | §12.2、P-05、P-06 |
| [02 營運總覽](02-overview.md) | 平台營運總覽與風險摘要 | P-01 |
| [03 組織與會籍](03-organizations-and-members.md) | 組織 CRUD、會籍代建與維護、身分證取用 | P-01、§2.2.1、X-05 |
| [04 店面管理](04-shops.md) | 店面清單／代建／代送審／審核／停權／關閉／撥款帳戶驗證 | P-01、C-01、S-10、P-07.2 |
| [05 商品複審與分類](05-products-and-categories.md) | 平台複審佇列、違規下架、商品分類主檔 | P-09 |
| [06 系統設定](06-settings.md) | 系統參數、費率、運費、年度行事曆 | P-04、§13.1～§13.3 |
| [07 金流作業](07-finance.md) | 退款、對帳、撥款、調整單、拒付 | §14.4、§14.6、§14.8、§14.9、P-08 |
| [08 風控與違規](08-risk.md) | 違規記錄、處置等級、申訴裁定、未取貨申訴 | P-07、§14.7.3 |
| [09 內容處置與個資](09-moderation-and-privacy.md) | 評價隱藏、對話介入、訊息處置、個資權利申請 | B-04.1、X-03、X-04 |
| [10 PAYUNi 商家申請](10-payuni-applications.md) | 代建、撤回、列表與單案查閱；不代表可收款 | S-10.1a、S-10.1c |
| [11 PAYUNi 私有證據](11-payuni-evidence.md) | 真實收件、版本列表、作廢與受控下載 | S-10.1b |
| [12 PAYUNi 核實與綁定](12-payuni-verification.md) | 明列證據預覽／確認、拒絕、綁定歷史及解除 | S-10.1c |
| [15 PAYUNi 付款設定](15-payuni-payment-settings.md) | 最新判讀查閱、明列版本及設定審查 | 12.1 |
| [16 PAYUNi 能力查詢](16-payuni-capability.md) | sandbox 原綁定 A3 同步、工作歷史查核 | S-10.1d |
| [17 PAYUNi 原付款查核](17-payuni-payment-query.md) | 財務單次 M3、原鍵重播及歷史 | P-10、12.1 |
| [18 PAYUNi 商家日帳](18-payuni-grant-query.md) | 批次、單頁、逐列觀察與原交易關聯；不等於已對帳 | P-08、14.9.3 |
| [19 營運告警](19-operation-alerts.md) | 依類別分流列出告警、標記已處理 | §12.1、§15.2 |
| [20 訂單與付款查詢](20-orders.md) | 跨工會訂單列表與詳情、全平台付款清單（唯讀、聯絡資料遮罩） | P-02、P-08 |
| [21 稽核日誌](21-audit-logs.md) | 全平台稽核紀錄查閱（唯讀、原值新值遮罩） | P-03 |
| [13 PAYUNi 憑證版本](13-payuni-credentials.md) | 本地版本清單、受控登錄及不可逆撤銷；不讀回秘密 | §12.2、§15.3 |

## 端點總表

共 146 條路由。除 `platform/auth/login` 外全部需要管理員 token。

### PAYUNi sandbox 實驗

操作細節見 [14 sandbox 契約實驗](14-payuni-contract-lab.md)，預設關閉，不啟用商城付款。

| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| POST | `platform/payuni/contract-lab` | payuni.credential.manage |
| POST | `platform/payuni/contract-lab/{labRun}/actions` | payuni.credential.manage |
| GET | `platform/payuni/contract-lab/{labRun}` | payuni.credential.manage |

### PAYUNi 憑證版本

| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/payuni/credential-versions` | `payuni.credential.manage` |
| POST | `platform/payuni/credential-versions` | `payuni.credential.manage` |
| POST | `platform/payuni/credential-versions/{credentialVersion}/revoke` | `payuni.credential.manage` |

### PAYUNi 商家申請

| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/shops/{shop}/payuni-applications` | `payuni.application.manage` |
| POST | `platform/shops/{shop}/payuni-applications` | `payuni.application.manage` |
| POST | `platform/shops/{shop}/payuni-applications/new-merchant` | `payuni.application.manage`；本地自動配號 |
| GET | `platform/shops/{shop}/payuni-applications/{payuniApplication}` | `payuni.application.manage` |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/withdraw` | `payuni.application.manage` |
| GET | `platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence` | `payuni.evidence.read` |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence` | `payuni.evidence.manage` |
| GET | `platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence/{payuniEvidence}/download` | `payuni.evidence.read` |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/evidence/{payuniEvidence}/invalidate` | `payuni.evidence.manage` |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/verification-context` | `payuni.binding.verify` + `payuni.evidence.read`；每次查閱留理由及稽核 |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/verification/preview` | `payuni.binding.verify` + `payuni.evidence.read`；批准改綁另需 `payuni.binding.manage` |
| POST | `platform/shops/{shop}/payuni-applications/{payuniApplication}/verification` | 同預覽 |
| GET | `platform/shops/{shop}/payuni-bindings` | `payuni.binding.manage` |
| GET | `platform/shops/{shop}/payuni-bindings/{payuniBinding}/payment-settings` | `payuni.binding.verify` + `payuni.evidence.read` |
| POST | `platform/shops/{shop}/payuni-bindings/{payuniBinding}/payment-settings` | `payuni.binding.verify` + `payuni.evidence.read` |
| GET | `platform/shops/{shop}/payuni-bindings/{payuniBinding}/capability-query` | `payuni.binding.manage` |
| POST | `platform/shops/{shop}/payuni-bindings/{payuniBinding}/capability-query` | `payuni.binding.manage` |
| GET | `platform/payments/{paymentNo}/payuni-query` | `settlement.reconcile` |
| POST | `platform/payments/{paymentNo}/payuni-query` | `settlement.reconcile` |
| POST | `platform/payments/{paymentNo}/payuni-reconciliation` | `settlement.reconcile`（sandbox 原付款核對採用；非撥款） |
| POST | `platform/payments/{paymentNo}/payuni-grant-queries` | `settlement.reconcile` |
| GET | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}` | `settlement.reconcile` |
| POST | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages` | `settlement.reconcile` |
| GET | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}` | `settlement.reconcile` |
| POST | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/materialization` | `settlement.reconcile` |
| GET | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows` | `settlement.reconcile` |
| GET | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/{rowId}` | `settlement.reconcile` |
| POST | `platform/payments/{paymentNo}/payuni-grant-queries/{batchKey}/pages/{pageNumber}/rows/{rowId}/link` | `settlement.reconcile` |
| POST | `platform/shops/{shop}/payuni-bindings/{payuniBinding}/release` | `payuni.binding.manage` |

### 認證與帳號權限
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| POST | `platform/auth/login` | 無（公開） |
| POST | `platform/auth/logout` | 僅需登入 |
| GET | `platform/me` | 僅需登入 |
| GET | `platform/permissions` | `role.manage` |
| GET | `platform/roles` | `role.manage` |
| POST | `platform/roles` | `role.manage` |
| PATCH | `platform/roles/{role}` | `role.manage` |
| PUT | `platform/roles/{role}/permissions` | `role.manage` |
| POST | `platform/members/{member}/roles` | `role.manage` |
| DELETE | `platform/member-roles/{memberRole}` | `role.manage` |
| POST | `platform/admins/{admin}/roles` | `role.manage` |
| DELETE | `platform/admin-roles/{adminRole}` | `role.manage` |
| GET | `platform/admins` | `admin.manage` |
| POST | `platform/admins` | `admin.manage` |
| PATCH | `platform/admins/{admin}/disable` | `admin.manage` |

### 營運總覽
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/overview` | `organization.manage` |

### 組織與會籍
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/organizations` | `organization.manage` |
| POST | `platform/organizations` | `organization.manage` |
| GET | `platform/organizations/{organization}` | `organization.manage` |
| PATCH | `platform/organizations/{organization}` | `organization.manage` |
| PATCH | `platform/organizations/{organization}/status` | `organization.manage` |
| GET | `platform/organizations/{organization}/sync-binding` | `organization.sync_binding` |
| POST | `platform/organizations/{organization}/sync-binding/preview` | `organization.sync_binding` |
| POST | `platform/organizations/{organization}/sync-binding` | `organization.sync_binding` |
| DELETE | `platform/organizations/{organization}/sync-binding` | `organization.sync_binding` |
| GET | `platform/members` | `member.manage` |
| POST | `platform/members` | `member.manage` |
| GET | `platform/members/{member}` | `member.manage` |
| PATCH | `platform/members/{member}` | `member.manage` |
| PATCH | `platform/members/{member}/membership-status` | `member.manage` |
| PATCH | `platform/members/{member}/marketplace-status` | `member.manage` |
| POST | `platform/members/{member}/account` | `member.account_issue`（`offline` 另需 `member.credential.offline`） |
| POST | `platform/members/{member}/access-suspension` | `member.access_suspend` |
| DELETE | `platform/members/{member}/access-suspension` | `member.access_suspend` |
| GET | `platform/members/{member}/identity` | `member.identity.reveal` |

### 店面
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/shops` | `shop.review` |
| GET | `platform/shops/{shop}` | `shop.review` |
| PATCH | `platform/shops/{shop}` | `shop.review` |
| GET | `platform/shops/{shop}/disposition` | `shop.review` |
| POST | `platform/shop-applications` | `shop.review` |
| POST | `platform/shop-applications/{application}/submit` | `shop.review` |
| POST | `platform/shop-applications/{application}/decision` | `shop.review` |
| PATCH | `platform/shop-applications/{application}/bank-account/verify` | `shop.review` |
| PATCH | `platform/shops/{shop}/close` | `violation.manage` |
| PATCH | `platform/shops/{shop}/suspend` | `violation.manage` |
| PATCH | `platform/shops/{shop}/resume` | `violation.manage` |
| PATCH | `platform/shops/{shop}/bank-account/verify` | `shop.review` |

> **代建、代送審與審核的對象是「申請單」，不是「店面」**（§11.2）。
> 店面於申請核准的同一交易內才被建立，因此核准之前沒有店面可以指向。
> `shops.status` 只有 `approved` / `open` / `suspended` / `closed` 四態，
> 審核期的狀態全在 `shop_applications` 上。

### 商品與分類
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/product-reviews` | `product.review` |
| POST | `platform/product-reviews/{product}/decision` | `product.review` |
| POST | `platform/products/{product}/violation-delist` | `violation.manage` |
| GET | `platform/categories` | `category.manage` |
| POST | `platform/categories` | `category.manage` |
| PATCH | `platform/categories/{category}` | `category.manage` |
| PATCH | `platform/categories/{category}/status` | `category.manage` |

### 系統設定
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/system-parameters` | `system_parameter.manage` |
| PATCH | `platform/system-parameters/{key}` | `system_parameter.manage` |
| GET | `platform/fee-rules` | `fee_rule.manage` |
| POST | `platform/fee-rules` | `fee_rule.manage` |
| PATCH | `platform/fee-rules/{feeRule}/status` | `fee_rule.manage` |
| GET | `platform/shipping-rates` | `fee_rule.manage` |
| POST | `platform/shipping-rates` | `fee_rule.manage` |
| PATCH | `platform/shipping-rates/{shippingRate}/status` | `fee_rule.manage` |
| GET | `platform/business-days` | `system_parameter.manage` |
| POST | `platform/business-days/import` | `system_parameter.manage` |

### 營運告警
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/operation-alerts` | `operation_alert.all`／`.finance`／`.fulfillment`（依類別） |
| PATCH | `platform/operation-alerts/{operationAlert}/acknowledge` | 同上，限告警類別在範圍內 |

### 稽核日誌
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/audit-logs` | `audit.view` |

### 訂單查詢
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/orders` | `order.view` |
| GET | `platform/orders/{orderNo}` | `order.view` |
| GET | `platform/payments` | `order.view` |

### 金流
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/refunds` | `order.view` |
| POST | `platform/refunds/{refundNo}/dispute-decision` | `refund.dispute`（客服） |
| POST | `platform/refunds/{refundNo}/process` | `refund.process`（**僅財務**） |
| GET | `platform/refunds/{refundNo}/payuni-query` | `refund.process` + `settlement.reconcile`（sandbox 原退款歷史） |
| POST | `platform/refunds/{refundNo}/payuni-query` | `refund.process` + `settlement.reconcile`（sandbox 原占用 M3 查回；獨立啟用後可自動採用） |
| GET | `platform/refunds/{refundNo}/payuni-correlation` | `refund.process` + `settlement.reconcile`（純本地核對，不結案） |
| POST | `platform/refunds/{refundNo}/payuni-manual-completion` | `refund.process` + `settlement.reconcile`（sandbox 同額退款後台人工核對結案，不外送） |
| POST | `platform/refunds/{refundNo}/payuni-origin` | `refund.process`（原 sandbox 退款來源凍結，不外送） |
| GET | `platform/refunds/{refundNo}/payuni-submission` | `refund.process`（原占用及首次回條，不外呼） |
| POST | `platform/refunds/{refundNo}/payuni-submission` | `refund.process`（明示單次 M4；預設關閉，不結案） |
| GET | `platform/settlement-files` | `settlement.reconcile` |
| POST | `platform/settlement-files` | `settlement.import` |
| POST | `platform/settlement-files/{settlementFile}/reconcile` | `settlement.reconcile` |
| GET | `platform/reconciliation-discrepancies` | `settlement.reconcile` |
| PATCH | `platform/reconciliation-discrepancies/{discrepancy}` | `settlement.reconcile` |
| GET | `platform/payouts` | `payout.view` |
| POST | `platform/payouts/{payoutNo}/execute` | `payout.execute`（**僅財務**） |
| GET | `platform/adjustments` | `payout.view` |
| POST | `platform/adjustments` | `adjustment.create` |
| POST | `platform/adjustments/{adjustment}/approve` | `adjustment.approve` |
| GET | `platform/chargebacks` | `chargeback.manage` |
| POST | `platform/chargebacks` | `chargeback.manage` |
| POST | `platform/chargebacks/{chargeback}/resolve` | `chargeback.manage` |

### 風控
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/violations` | `violation.view` |
| POST | `platform/violations` | `violation.manage` |
| POST | `platform/violations/{violation}/revoke` | `violation.manage` |
| GET | `platform/violation-appeals` | `violation.view` |
| POST | `platform/violation-appeals/{appeal}/decision` | `violation.manage` |
| GET | `platform/shops/{shop}/disposition` | `violation.view` |
| GET | `platform/pickup-failure-appeals` | `violation.manage` |
| POST | `platform/pickup-failures/{pickupFailure}/decision` | `violation.manage` |

### 內容處置與個資
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/reviews` | `review.moderate` |
| POST | `platform/reviews/{review}/hide` | `review.moderate` |
| GET | `platform/flagged-messages` | `message.moderate` |
| POST | `platform/conversations/{conversation}/join` | `refund.dispute` |
| POST | `platform/conversations/{conversation}/leave` | `refund.dispute` |
| GET | `platform/conversations/{conversation}/messages` | **本人已加入且尚未退出**該對話 |
| POST | `platform/conversations/{conversation}/messages` | **本人已加入且尚未退出**該對話 |
| POST | `platform/messages/{message}/hide` | `message.moderate` |
| GET | `platform/data-requests` | `audit.view` |
| POST | `platform/data-requests/{dataRequest}/decision` | `audit.view` |
| POST | `platform/data-requests/{dataRequest}/execute` | `audit.view` |

## 職責分離：三個預設角色能做什麼

前端做選單／按鈕的顯示隱藏時請依這張表，不要以「是管理員就都給」處理。

| 權限 | 平台管理者 `platform_admin` | 財務人員 `platform_finance` | 客服人員 `platform_cs` |
| --- | :---: | :---: | :---: |
| `organization.manage`（組織、營運總覽） | ✅ | — | — |
| `member.manage` / `member.view` | ✅ | — | — |
| `member.marketplace_dispose`（商城側停權／恢復會員） | ✅ | — | — |
| `member.identity.reveal` | ✅ | ✅ | — |
| `shop.review`（店面審核、代建） | ✅ | — | — |
| `product.review` | ✅ | — | — |
| `category.manage` / `fee_rule.manage` / `system_parameter.manage` | ✅ | — | — |
| `role.manage` / `admin.manage` / `audit.view` | ✅ | — | — |
| `operation_alert.all`（全部營運告警，§12.1） | ✅ | — | — |
| `operation_alert.finance`（付款、退款、撥款、對帳告警） | — | ✅ | — |
| `operation_alert.fulfillment`（出貨告警） | — | — | ✅ |
| `payuni.application.manage`（PAYUNi 商家申請的提出與撤回，§S-10.1a） | ✅ | — | — |
| `payuni.evidence.manage`（登錄與作廢 PAYUNi 私有證據，§S-10.1b） | ✅ | — | — |
| `payuni.evidence.read`（讀取 PAYUNi 私有證據原件，§S-10.1b） | ✅ | — | — |
| `payuni.binding.manage`（解除 PAYUNi 生效綁定，§S-10.1c） | ✅ | — | — |
| `payuni.binding.verify`（核實／拒絕 PAYUNi 商家歸屬，§S-10.1c） | ✅ | — | — |
| `payuni.credential.manage`（保存受控憑證及本地撤銷，§12.2；內部服務，不提供讀秘密 API） | ✅ | — | — |
| `violation.view` / `violation.manage`（含停權、違規下架） | ✅ | — | 僅 `violation.view` |
| `violation.record` / `shop.dispose` | — | — | — |
| `order.view` | ✅ | ✅ | ✅ |
| `refund.dispute`（裁定爭議、加入對話） | — | — | ✅ |
| `refund.process`（**執行退款**） | — | ✅ | ❌ 不可授予 |
| `payout.view` | ✅ | ✅ | — |
| `payout.execute`（**執行撥款**） | — | ✅ | ❌ 不可授予 |
| `settlement.import` / `settlement.reconcile` / `chargeback.manage` | — | ✅ | — |
| `adjustment.create` | — | ✅ | — |
| `adjustment.approve` | ✅ | — | — |
| `review.moderate`（隱藏評價） | ✅ | — | — |
| `message.moderate`（隱藏訊息） | ✅ | — | ✅ |

> `violation.record`（建立違規記錄）與 `shop.dispose`（暫停店面）**三個平台預設角色
> 都沒有** —— 它們是為工會管理者切出來的（§P-07.4）。平台端走的是更完整的
> `violation.manage`。三者的分野見
> [01 §4 三組看起來相似、但刻意分開的權限](01-auth-and-accounts.md#4-權限清單)。

三條硬規則（spec §15.3，**不可由設定放寬**）：

1. **裁定 ≠ 執行**：客服裁定「該不該退」，財務執行「退出去」。退款與撥款的執行權限
   不得授予客服類角色。
2. **建立 ≠ 核准**：調整單的建立者不得核准自己開的單（服務層另外檢查）。
3. **送審 ≠ 審核**：代為送審的人不得再擔任該件的審核者。

付款未知事件留存為 `unconfirmed` 並告警，缺金額的成功通知轉差異；不得以人工改付款狀態
繞過查核。詳見 [本地付款觀察](../../integration/payment-observations.md)；這不是 PAYUNi ACK 契約。
