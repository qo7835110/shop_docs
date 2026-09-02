# 買家中心 API

路由前綴：`buyer/`（完整路徑為 `/api/buyer/...`）
認證：**會員 token**（`Authorization: Bearer {member_token}`）

> 先讀 [共通約定](../conventions.md)。回應信封、分頁、錯誤碼、遮罩規則、金額與天數單位都在那裡，本目錄不再重複。

## 買家不是一種角色，是預設狀態

任何有會籍的人都是買家。`buyer` 系統角色帶四個權限（`order.place`、`order.view`、
`return.request`、`review.write`），範圍為 `platform` —— 不綁店面、不綁組織。

同一個人可以同時是某店的賣家、某工會的管理者與買家；用的是同一組帳號密碼、
同一個 token。

## 登入的那組帳號決定你買得到什麼價格

一個自然人在多個工會有會籍時，**每個工會各有一組帳密**（§X-05.1）；
登入哪一組，這個 token 就是那個工會的身分，而它決定三件事：

| 決定什麼 | 怎麼影響 |
| --- | --- |
| 會員價資格 | `member_price_scope` 為 `own_organization` 的商品，只有該工會的會籍算數（§13.5.2） |
| 會員限定商品 | `member_only` 的規格買不買得到 |
| 訂單的租戶歸屬 | 下單當下寫入快照，**事後以另一個身分登入不會改寫既有訂單** |

會籍由**登入的帳號**決定（§X-05.1）：一個人加入兩個工會就有兩組帳密，
沒有切換端點。詳見[一個工會一組帳號](01-profile-and-notifications.md#2-一個工會一組帳號)。

---

## 買家端做不到的事

| 想做的事 | 為什麼不行 | 走哪裡 |
| --- | --- | --- |
| 自己付款 | 付款由**第三方金流商**處理；平台只建立付款單並記錄結果（§12.1） | 建立付款單後導向金流商 |
| 取消已出貨的訂單 | 已出貨只能退貨 | [退貨申請](05-returns-and-refunds.md#1-申請退貨退款) |
| 修改或刪除自己的評價 | 評價不可修改、不可刪除；只能**追評**一次（§B-04） | [追評](06-reviews.md#4-追評) |
| 要求賣家分擔退貨運費以外的事 | 退貨運費本來就由賣家負擔（§14.5.1） | —— |
| 刪除自己的訂單紀錄 | 交易紀錄依法定保存義務保留 | [去識別化](10-privacy.md) |
| 關閉交易類通知 | 訂單、付款、出貨、退款、帳號安全、違規等**不可關閉**（§X-02） | 只有行銷推播關得掉 |
| 直接讀取賣家的成本或費率 | 那是賣家的營業資料（§2.3） | —— |

---

## 端點總表

共 50 條路由，全部需要會員 token。

### 個人資料、會籍與通知
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/profile` | [01](01-profile-and-notifications.md#1-個人資料) |
| GET | `buyer/consents` | [01](01-profile-and-notifications.md#3-同意設定) |
| PUT | `buyer/consents` | [01](01-profile-and-notifications.md#3-同意設定) |
| GET | `buyer/notification-preferences` | [01](01-profile-and-notifications.md#4-通知設定) |
| PUT | `buyer/notification-preferences` | [01](01-profile-and-notifications.md#4-通知設定) |
| GET | `buyer/notifications` | [01](01-profile-and-notifications.md#5-通知列表) |
| PATCH | `buyer/notifications/{notification}/read` | [01](01-profile-and-notifications.md#6-標記已讀) |

### 地址簿與發票設定
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/addresses` | [02](02-addresses-and-invoices.md#1-地址簿) |
| POST | `buyer/addresses` | [02](02-addresses-and-invoices.md#2-新增地址) |
| PATCH | `buyer/addresses/{address}` | [02](02-addresses-and-invoices.md#3-修改地址) |
| DELETE | `buyer/addresses/{address}` | [02](02-addresses-and-invoices.md#4-刪除地址) |
| GET | `buyer/invoice-profiles` | [02](02-addresses-and-invoices.md#5-發票設定) |
| POST | `buyer/invoice-profiles` | [02](02-addresses-and-invoices.md#6-新增發票設定) |
| PATCH | `buyer/invoice-profiles/{invoiceProfile}` | [02](02-addresses-and-invoices.md#7-修改發票設定) |
| DELETE | `buyer/invoice-profiles/{invoiceProfile}` | [02](02-addresses-and-invoices.md#8-刪除發票設定) |

### 購物車與結帳
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/cart` | [03](03-cart-and-checkout.md#1-購物車內容) |
| POST | `buyer/cart` | [03](03-cart-and-checkout.md#2-加入購物車) |
| PATCH | `buyer/cart/items/{item}` | [03](03-cart-and-checkout.md#3-調整數量) |
| DELETE | `buyer/cart/items/{item}` | [03](03-cart-and-checkout.md#4-移除品項) |
| POST | `buyer/checkout/preview` | [03](03-cart-and-checkout.md#5-結帳試算) |
| POST | `buyer/checkout` | [03](03-cart-and-checkout.md#6-送出訂單) |

### 訂單與付款
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/orders` | [04](04-orders-and-payment.md#1-訂單列表) |
| GET | `buyer/orders/{orderNo}` | [04](04-orders-and-payment.md#2-訂單詳情) |
| POST | `buyer/orders/{orderNo}/payments` | [04](04-orders-and-payment.md#3-建立付款單) |
| POST | `buyer/orders/{orderNo}/cancel` | [04](04-orders-and-payment.md#4-取消尚未付款的訂單) |
| POST | `buyer/sub-orders/{orderNo}/confirm-receipt` | [04](04-orders-and-payment.md#5-確認收貨) |

### 退貨、取消與退款
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| POST | `buyer/sub-orders/{orderNo}/return-requests` | [05](05-returns-and-refunds.md#1-申請退貨退款) |
| POST | `buyer/return-requests/{returnRequest}/shipment` | [05](05-returns-and-refunds.md#2-登錄寄回物流單號) |
| POST | `buyer/sub-orders/{orderNo}/cancel-requests` | [05](05-returns-and-refunds.md#3-申請取消訂單) |
| POST | `buyer/cancel-requests/{cancelRequest}/response` | [05](05-returns-and-refunds.md#4-回應賣家的取消申請) |
| GET | `buyer/refunds` | [05](05-returns-and-refunds.md#5-退款單列表) |
| GET | `buyer/refunds/{refundNo}` | [05](05-returns-and-refunds.md#6-退款單詳情) |
| POST | `buyer/refunds/{refundNo}/dispute` | [05](05-returns-and-refunds.md#7-提出爭議) |

### 評價
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/reviews` | [06](06-reviews.md#1-我的評價) |
| GET | `buyer/reviews/pending` | [06](06-reviews.md#2-待評價清單) |
| POST | `buyer/order-items/{orderItem}/review` | [06](06-reviews.md#3-撰寫評價) |
| POST | `buyer/reviews/{review}/append` | [06](06-reviews.md#4-追評) |

### 收藏
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/favorites` | [07](07-favorites.md#1-收藏清單) |
| POST | `buyer/favorites` | [07](07-favorites.md#2-切換收藏) |
| DELETE | `buyer/favorites` | [07](07-favorites.md#3-取消收藏) |

### 訊息
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/conversations` | [08](08-messages.md#1-對話列表) |
| POST | `buyer/conversations` | [08](08-messages.md#2-發起對話) |
| GET | `buyer/conversations/{conversation}/messages` | [08](08-messages.md#3-讀取訊息) |
| POST | `buyer/conversations/{conversation}/messages` | [08](08-messages.md#4-發送訊息) |

### 未取貨紀錄
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/pickup-failures` | [09](09-pickup-failures.md#1-我的未取貨紀錄) |
| POST | `buyer/pickup-failures/{pickupFailure}/appeal` | [09](09-pickup-failures.md#2-提出申訴) |

### 個資權利
| 方法 | 路徑 | 文件 |
| --- | --- | --- |
| GET | `buyer/data-requests` | [10](10-privacy.md#1-我的申請) |
| GET | `buyer/data-requests/eligibility` | [10](10-privacy.md#2-刪除申請的可受理性) |
| POST | `buyer/data-requests` | [10](10-privacy.md#3-提出申請) |
| GET | `buyer/data-requests/{dataRequest}/download` | [10](10-privacy.md#4-下載匯出檔) |

---

## 三個貫穿買家流程的規則

1. **金額一律由後端重算。** 結帳時送的 `expected_payable_amount` **只用於比對**，
   不一致即 422，絕不作為入帳依據（§11.3）。前端請把它送出去 ——
   那是防止「停在結帳頁太久，價格已經變了」的唯一機制。

2. **付款失敗不取消訂單、不釋放庫存。** 一次失敗的付款嘗試什麼都不會發生，
   你在期限內重試即可（§14.2）。只有**待付款逾期**或**你自己取消**才會釋放預留。

3. **自動確認收貨與退貨期限是兩條線。** 訂單自動完成之後，你**依然**可以在
   退貨期限前申請退貨 —— 兩者互不影響（§14.3）。
