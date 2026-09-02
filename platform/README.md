# 平台後台 API

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

## 端點總表

共 92 條路由。除 `platform/auth/login` 外全部需要管理員 token。

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
| GET | `platform/members` | `member.manage` |
| POST | `platform/members` | `member.manage` |
| GET | `platform/members/{member}` | `member.manage` |
| PATCH | `platform/members/{member}` | `member.manage` |
| PATCH | `platform/members/{member}/membership-status` | `member.manage` |
| PATCH | `platform/members/{member}/marketplace-status` | `member.manage` |
| POST | `platform/members/{member}/account` | `member.account_issue`（`offline` 另需 `member.credential.offline`） |
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

### 金流
| 方法 | 路徑 | 需要權限 |
| --- | --- | --- |
| GET | `platform/refunds` | `order.view` |
| POST | `platform/refunds/{refundNo}/dispute-decision` | `refund.dispute`（客服） |
| POST | `platform/refunds/{refundNo}/process` | `refund.process`（**僅財務**） |
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
