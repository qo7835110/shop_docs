# 賣家中心 API

路由前綴：`seller/`（完整路徑為 `/api/seller/...`）
認證：**會員 token**（`Authorization: Bearer {member_token}`）

> 先讀 [共通約定](../conventions.md)。回應信封、分頁、錯誤碼、遮罩規則、金額與天數單位都在那裡，本目錄不再重複。

## 賣家是「會員」，不是另一種帳號

與工會管理者同一件事：**「賣家」不是帳號類型，而是某個會籍被授予了 `seller` 角色**，
而該角色的範圍（`scope_id`）指向**一間特定的店面**。

同一個人可以同時是某店的賣家、某工會的管理者、以及買家 —— 同一組帳號密碼、
同一個 token —— 那是**同一筆會籍的多個面向**，差別只在他持有哪些角色（§X-05.1）。

### 角色是在**核准那一刻**才拿到的

開店申請核准的同一交易內，系統做兩件事：建立店面，然後把 `seller` 角色授予店主、
範圍設為那間店（[SellerRoleGranter](../../../app/Services/Shops/SellerRoleGranter.php)）。

因此**送審中的申請人打不了商品與庫存端點** —— 那些走權限判定，而他還沒有權限。
403 的訊息會寫成「店面尚未通過審核，或您沒有管理商品的授權」，兩種情形共用一句話
是刻意的：對呼叫端而言下一步相同。

## 兩種閘門，缺一不可

賣家端的授權有**兩層**，看端點而定：

| 層 | 問的問題 | 例子 |
| --- | --- | --- |
| **擁有權** | 這間店是不是你的？ | 店面維護、開業、關閉、撥款帳戶、報表 |
| **擁有權 ＋ 權限** | 是你的店，而且你有這個授權嗎？ | 商品、庫存、訂單履約、評價回覆、撥款查詢 |

只看擁有權的那一組（`ShopPolicy` 的 `update` / `close` / `viewOwn` /
`manageBankAccount`）**不檢查 `shop.manage`**：那些是店主對自己店面的固有權利，
不是可以被收回的授權。要收回請停權店面，不是拿掉權限。

`seller` 角色的五個權限對應到端點：

| 權限 | 管到哪些端點 |
| --- | --- |
| `product.manage` | 商品與 SKU 的全部寫入（[03](03-products-and-skus.md)） |
| `inventory.manage` | 庫存的全部寫入與查詢（[04](04-inventory.md)） |
| `order.fulfill` | 訂單查詢、出貨、退貨處理、取消回應、對話、拒付抗辯（[05](05-orders-and-shipping.md)、[06](06-returns-and-refunds.md)、[07](07-payouts.md)、[08](08-reviews-and-messages.md)） |
| `payout.view` | 撥款查詢（[07](07-payouts.md)） |
| `review.reply` | 評價列表與回覆（[08](08-reviews-and-messages.md)） |
| `shop.manage` | **目前沒有端點檢查它** —— 店面維護走擁有權。保留給日後的多人店面 |

> 一人多店在第一階段**不支援**（單店、單倉，§1.5）。但授權模型已經是「角色綁店面」，
> 因此端點一律要求 `shop_id` 或以路由參數帶店面，不從 token 推。

---

## 賣家端做不到的事

前端不要留這些按鈕 —— 它們在賣家端**沒有端點**：

| 想做的事 | 為什麼不行 | 誰能做 |
| --- | --- | --- |
| 直接建立「店面」 | 建立的是**申請單**；店面於核准的同一交易內才存在（§11.2） | 送審後由工會或平台核准 |
| 自行核准開店申請 | 申請人與審核人不得為同一帳號（§15.3） | 工會（§U-02）或平台（法人，§C-01） |
| 標記撥款帳戶「驗證通過」 | 自己核准自己 | 平台（`platform/.../bank-account/verify`） |
| 解除**工會或平台**下的停權 | 越級解除以 403 拒絕並留稽核（§11.2、§P-07.4） | 處置者或更高層級 |
| 手動扣減庫存來「出貨」 | 出貨才扣 `actual`，且必須留異動紀錄（§11.4） | 走 `ship`，不是庫存端點 |
| 把瑕疵品改回可售 | 瑕疵品不得在同一 SKU 上復原（§S-05.4） | 只能 `defective-disposal` 退回買家或報廢 |
| 修改或刪除買家評價 | 評價不得由賣家移除（§B-04.1） | 回覆或檢舉；隱藏只有平台做得到 |
| 執行退款、觸發撥款 | 賣家不碰金流（§14.4、§14.9.1） | 平台財務；撥款由排程產生候選 |
| 向買家收退貨運費 | 退貨運費由賣家負擔，不自退款金額扣除（§14.5.1） | —— 這是規則，不是設定 |

---

## 端點總表

共 80 條路由，全部需要會員 token。

### 開店申請（S-01、S-10）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| POST | `seller/shop-applications` | 開店資格 | [01](01-shop-application.md#2-建立申請草稿) |
| GET | `seller/shop-applications` | 本人 | [01](01-shop-application.md#1-我的申請紀錄) |
| GET | `seller/shop-applications/{application}` | 申請人本人 | [01](01-shop-application.md#3-申請詳情) |
| PATCH | `seller/shop-applications/{application}` | 申請人本人 | [01](01-shop-application.md#4-修改草稿) |
| POST | `seller/shop-applications/{application}/submit` | 申請人本人 | [01](01-shop-application.md#7-送出審核) |
| GET | `seller/shop-applications/{application}/documents` | 申請人本人 | [01](01-shop-application.md#5-附件) |
| POST | `seller/shop-applications/{application}/documents` | 申請人本人 | [01](01-shop-application.md#5-附件) |
| GET | `seller/shop-applications/{application}/bank-account` | 申請人本人 | [01](01-shop-application.md#6-撥款帳戶) |
| PUT | `seller/shop-applications/{application}/bank-account` | 申請人本人 | [01](01-shop-application.md#6-撥款帳戶) |

### 店面（S-02、S-03、S-10）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/shops/{shop}/overview` | 店主 | [02](02-shop.md#1-賣家中心總覽) |
| PATCH | `seller/shops/{shop}` | 店主 | [02](02-shop.md#2-維護店面資料) |
| POST | `seller/shops/{shop}/open` | 店主 | [02](02-shop.md#3-開始營業) |
| GET | `seller/shops/{shop}/announcements` | 店主 | [02](02-shop.md#9-店家公告) |
| POST | `seller/shops/{shop}/announcements` | 店主 | [02](02-shop.md#9-店家公告) |
| PATCH | `seller/shops/{shop}/announcements/{announcement}` | 店主 | [02](02-shop.md#9-店家公告) |
| POST | `seller/shops/{shop}/announcements/{announcement}/unpublish` | 店主 | [02](02-shop.md#9-店家公告) |
| POST | `seller/shops/{shop}/images/{slot}` | 店主 | [02](02-shop.md#8-店面識別圖像) |
| DELETE | `seller/shops/{shop}/images/{slot}` | 店主 | [02](02-shop.md#8-店面識別圖像) |
| POST | `seller/shops/{shop}/pause` | 店主 | [02](02-shop.md#4-暫停營業) |
| POST | `seller/shops/{shop}/close` | 店主 | [02](02-shop.md#5-關閉店面) |
| GET | `seller/shops/{shop}/favorite-count` | 店主 | [02](02-shop.md#6-被收藏數) |
| GET | `seller/shops/{shop}/bank-account` | 店主 | [02](02-shop.md#7-撥款帳戶) |
| PUT | `seller/shops/{shop}/bank-account` | 店主 | [02](02-shop.md#7-撥款帳戶) |

### 商品與規格（S-04）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/products` | `product.manage` @ 本店 | [03](03-products-and-skus.md#1-商品列表) |
| POST | `seller/products` | `product.manage` @ 本店 | [03](03-products-and-skus.md#2-建立商品草稿) |
| GET | `seller/products/{product}` | 店主 | [03](03-products-and-skus.md#3-商品詳情與送審檢核) |
| PATCH | `seller/products/{product}` | `product.manage` @ 本店 | [03](03-products-and-skus.md#4-修改商品) |
| POST | `seller/products/{product}/submit` | `product.manage` @ 本店 | [03](03-products-and-skus.md#5-送出審核) |
| POST | `seller/products/{product}/list` | `product.manage` @ 本店 | [03](03-products-and-skus.md#6-上架) |
| POST | `seller/products/{product}/delist` | `product.manage` @ 本店 | [03](03-products-and-skus.md#7-下架) |
| GET | `seller/products/{product}/images` | 店主 | [03](03-products-and-skus.md#12-商品圖片) |
| POST | `seller/products/{product}/images` | `product.manage` @ 本店 | [03](03-products-and-skus.md#13-上傳圖片) |
| PUT | `seller/products/{product}/images/order` | `product.manage` @ 本店 | [03](03-products-and-skus.md#14-圖片排序) |
| DELETE | `seller/products/{product}/images/{image}` | `product.manage` @ 本店 | [03](03-products-and-skus.md#15-刪除圖片) |
| GET | `seller/products/{product}/skus` | 店主 | [03](03-products-and-skus.md#8-規格列表) |
| POST | `seller/products/{product}/skus` | `product.manage` @ 本店 | [03](03-products-and-skus.md#9-新增規格) |
| PATCH | `seller/skus/{sku}` | `product.manage` @ 本店 | [03](03-products-and-skus.md#10-修改規格) |
| PATCH | `seller/skus/{sku}/deactivate` | `product.manage` @ 本店 | [03](03-products-and-skus.md#11-停用規格) |

### 庫存（S-05）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/inventories` | `inventory.manage` @ 本店 | [04](04-inventory.md#1-庫存列表) |
| GET | `seller/inventories/{sku}/movements` | `inventory.manage` @ 本店 | [04](04-inventory.md#2-庫存異動明細) |
| POST | `seller/inventories/{sku}/stock-in` | `inventory.manage` @ 本店 | [04](04-inventory.md#3-入庫) |
| POST | `seller/inventories/{sku}/stock-out` | `inventory.manage` @ 本店 | [04](04-inventory.md#4-出庫) |
| POST | `seller/inventories/{sku}/stocktake` | `inventory.manage` @ 本店 | [04](04-inventory.md#5-盤點) |
| POST | `seller/inventories/{sku}/defective-disposal` | `inventory.manage` @ 本店 | [04](04-inventory.md#6-瑕疵品處置) |
| PATCH | `seller/inventories/{sku}/safety-stock` | `inventory.manage` @ 本店 | [04](04-inventory.md#7-安全庫存) |

### 訂單與出貨（S-06、S-12）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/orders` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#1-訂單列表) |
| GET | `seller/orders/{orderNo}` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#2-訂單詳情) |
| POST | `seller/orders/{orderNo}/shipping-label` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#3-建立物流單號印單) |
| POST | `seller/orders/{orderNo}/ship` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#4-確認出貨) |
| PATCH | `seller/orders/{orderNo}/shipment` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#5-更正物流單號) |
| POST | `seller/orders/{orderNo}/handover` | `order.fulfill` @ 本店 | [05](05-orders-and-shipping.md#6-自取交付) |

### 退貨、取消與退款（S-07、B-03、B-06）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/refunds` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#1-退款單列表) |
| POST | `seller/refunds/{refundNo}/dispute` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#4-對驗收結果提出爭議) |
| POST | `seller/return-requests/{returnRequest}/decision` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#2-回應退貨申請) |
| POST | `seller/return-requests/{returnRequest}/inspection` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#3-退貨驗收) |
| POST | `seller/orders/{orderNo}/confirm-return-received` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#5-確認收到退回商品配送異常超商未取) |
| POST | `seller/orders/{orderNo}/return-inspection` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#6-退回商品驗收配送異常超商未取) |
| POST | `seller/orders/{orderNo}/cancel-requests` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#7-賣家提出取消申請) |
| POST | `seller/cancel-requests/{cancelRequest}/response` | `order.fulfill` @ 本店 | [06](06-returns-and-refunds.md#8-回應買家的取消申請) |

### 撥款與拒付（S-08、§14.8）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/payouts` | `payout.view` @ 本店 | [07](07-payouts.md#1-撥款列表) |
| GET | `seller/payouts/{payoutNo}` | `payout.view` @ 本店 | [07](07-payouts.md#2-撥款單詳情) |
| GET | `seller/chargebacks` | `order.fulfill` @ 本店 | [07](07-payouts.md#3-我的拒付) |
| POST | `seller/chargebacks/{chargeback}/response` | `order.fulfill` @ 本店 | [07](07-payouts.md#4-拒付抗辯) |

### 評價與對話（S-11、X-03）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/reviews` | `review.reply` @ 本店 | [08](08-reviews-and-messages.md#1-評價列表) |
| POST | `seller/reviews/{review}/reply` | `review.reply` @ 本店 | [08](08-reviews-and-messages.md#2-回覆評價) |
| POST | `seller/reviews/{review}/report` | `review.reply` @ 本店 | [08](08-reviews-and-messages.md#3-檢舉評價) |
| GET | `seller/conversations` | `order.fulfill` @ 本店 | [08](08-reviews-and-messages.md#4-對話列表) |
| GET | `seller/conversations/{conversation}/messages` | `order.fulfill` @ 本店 | [08](08-reviews-and-messages.md#5-讀取訊息) |
| POST | `seller/conversations/{conversation}/messages` | `order.fulfill` @ 本店 | [08](08-reviews-and-messages.md#6-發送訊息) |

### 報表（S-09）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/shops/{shop}/reports/sales` | 店主 | [09](09-reports.md#1-銷售報表) |
| GET | `seller/shops/{shop}/reports/inventory` | 店主 | [09](09-reports.md#2-庫存報表) |
| GET | `seller/shops/{shop}/reports/orders` | 店主 | [09](09-reports.md#3-訂單與出貨報表) |
| GET | `seller/shops/{shop}/reports/refunds` | 店主 | [09](09-reports.md#4-退款報表) |
| GET | `seller/shops/{shop}/reports/payouts` | 店主 | [09](09-reports.md#5-撥款報表) |
| POST | `seller/shops/{shop}/reports/export` | 店主 | [09](09-reports.md#6-匯出) |
| GET | `seller/shops/{shop}/reports/exports` | 店主 | [09](09-reports.md#7-匯出紀錄) |
| GET | `seller/shops/{shop}/reports/exports/{export}/download` | 店主 | [09](09-reports.md#8-下載匯出檔) |

### 違規與申訴（P-07.3）
| 方法 | 路徑 | 閘門 | 文件 |
| --- | --- | --- | --- |
| GET | `seller/violations` | 店主 | [10](10-violations.md#1-本店面的違規記錄) |
| GET | `seller/violations/disposition` | 店主 | [10](10-violations.md#2-處置現況) |
| POST | `seller/violations/{violation}/appeal` | 店主 | [10](10-violations.md#3-提出申訴) |

---

## 三個貫穿全站的規則，賣家端最常踩到

1. **可售庫存是算出來的，不是存的**：
   `sellable = actual − reserved − awaiting_return_inspection − defective`（§11.3）。
   畫面上要顯示「還能賣幾件」一律讀 `sellable_quantity`，不要自己減。

2. **出貨才扣 `actual`，付款不扣**（§11.4）。付款成功只是把預留從「待付款階段」
   換成「待出貨階段」，數量一件都沒有移動。因此**出貨前的任何取消都不會有「回補庫存」
   的異動紀錄** —— 因為它根本沒有離開過。

3. **應付金額在付款成功後凍結**（§11.3）。全額退款也不會讓它變小，
   所以**任何營收數字都不可以由訂單列表自行加總**。要金額請用
   [報表](09-reports.md)，那裡只有一條算式。
