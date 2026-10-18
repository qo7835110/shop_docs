# 前台 API（訪客與買家共用）

平台財務的 [PAYUNi 商家日帳查詢](../platform/18-payuni-grant-query.md)需對帳權；本角色無批次、逐頁、逐列解析或原交易關聯入口，頁面收齊或關聯成功不代表已對帳或可撥款。

平台財務另有 [PAYUNi 原付款查核](../platform/17-payuni-payment-query.md)，本角色不得使用；讀回付款狀態不外呼、不入帳或授權重付。

PAYUNi [一次表單交付](../buyer/04-orders-and-payment.md#3b-payuni-一次表單交付)僅原下單會員可用；本角色沒有代付權，前景返回不表示付款成功。

PAYUNi 原付款觀察另有內部採用元件，但本角色無採用入口；讀取狀態不會觸發入帳或重新查詢。

原 sandbox 綁定的[能力查詢／歷史](../platform/16-payuni-capability.md)僅限具綁定管理權的平台管理員；本角色不能同步或宣告商家可收款。

既有商家的[管理員確認核實](../platform/12-payuni-verification.md)可免圖片，但仍僅限平台具權限管理員；本角色不能自行核准，也不因此取得收款資格。

PAYUNi sandbox 契約實驗僅供具權限的平台管理員使用；本角色無管理入口。
`query-capture` 亦只供平台查原交易請款、最後退款與剩餘額度觀察，不授予本角色外呼權，不改付款或退款狀態。
付款設定[判讀查閱／留存](../platform/15-payuni-payment-settings.md)僅供有權限的平台管理員使用；sandbox 可由管理員直接確認免附文件，本角色不能代為確認或藉此啟用收款。
付款設定原件 `payment_settings` 由平台沿[私有證據 API](../platform/11-payuni-evidence.md)收件；本角色無原件權限，收件不代表可付款。
詳見 [平台實驗 API](../platform/14-payuni-contract-lab.md)，它不會更新商城訂單或付款。
原 UPP run 的 `query-trade` 是平台人工唯讀查詢，不向未登入前台提供金流查詢入口。
缺欄位後的 `diagnose-trade` 亦僅限平台人工取證，不提供前台入口。
新版解析器的 `verify-query-parser` 同樣僅限平台一次唯讀驗收，不提供前台入口。

原 A2 交付歸屬查核僅由平台後台處理，不開放前台，不代表店面已具收款資格。

[A2 自動配號申請](../platform/10-payuni-applications.md#自動配號的新建店申請) 不開放前台；
候選 MerID 不公開，本地申請完成不代表商店已開通收款。

付款讀回的 `payuni_query` 只公開最新本地查核摘要，沿用同一會員授權；matched 不是已付款，GET 不發動金流查詢。

[付款讀回](../buyer/04-orders-and-payment.md#3a-查核本地付款) 不是公開查詢；需要下單會員的 token，不可憑訂單編號查詢他人付款。

前台不提供 PAYUNi 商家申請、核實或原值查閱入口，也不公開外部商家識別碼／證據。
本地商家管理見 [賣家 API](../seller/10-payuni-applications.md)；申請存在不等於商店已具支付能力。

路由前綴：`storefront/`、`auth/`，外加健康檢查 `health`
認證：**多數不需要 token**；帶了會員 token 則內容會依你的會籍調整（見下）

> 先讀 [共通約定](../conventions.md)。回應信封、分頁、錯誤碼、金額與遮罩規則都在那裡，本目錄不再重複。

## 同一支端點，訪客與會員看到的不一樣

前台的商品端點**接受選填的會員 token**。帶與不帶的差別不是「能不能看」，
而是**看到什麼價格**：

| | 訪客（不帶 token） | 會員（帶 token） |
| --- | --- | --- |
| `price_from` | 原價 | 若適用，**會員價** |
| `skus[].price` | 原價 | 若適用，會員價 |
| `skus[].price_type` | `normal` | `member` |
| 會員限定 SKU | `is_purchasable: false`，附原因 | 資格符合則 `true` |
| 工會公告 | 只看得到 `public` | 加上本工會的 `members_only` |

實測同一件商品：訪客的 `price_from` 是 180，DEMO 工會的會員是 160。

> **會員價依 token 所屬的會籍判定**（§13.5.2）。一個人在多個工會有會籍時，
> 各有一組帳密 —— 以哪一組登入，就看到哪一個工會的價格
> （[一個工會一組帳號](../buyer/01-profile-and-notifications.md#2-一個工會一組帳號)）。

## 前台看得到的東西是**一層一層篩過的**

商品出現在前台需要五個條件同時成立（§11.3、§F-01）：

1. 店面已核准**且**營業中（`open`）
2. 商品已核准**且**上架（`listed`）
3. SKU 可售庫存 > 0
4. 會員限定 SKU：viewer 的會籍資格符合
5. 配送方式的交集非空（平台 → 工會 → 店面 → 商品，§13.2.2）

前三項在 SQL 篩，後兩項在 SQL 之外判定 —— 因此清單類端點會**先取多一點候選再過濾**，
`per_page` 是上限不是保證筆數。

## 找不到就是 404，不是 403

草稿、審核中、已下架、違規下架的商品，以及從未開業過的店面，
**對前台而言不存在**：回 404 而不是 403 —— 後者會洩漏它曾經存在（§F-01、§2.3）。

**平台預設組織不在前台**（§2.2.1、§12.3-51）：它不承載店面、沒有商城頁，
因此 `storefront/organizations` 的清單排除它，`storefront/organizations/PF` 回 404。

---

## 端點總表

共 16 條路由。除登出與變更密碼外都不需要 token。

### 健康檢查
| 方法 | 路徑 | 需要 token | 文件 |
| --- | --- | :---: | --- |
| GET | `health` | — | [01](01-auth.md#0-健康檢查) |

### 認證
| 方法 | 路徑 | 需要 token | 文件 |
| --- | --- | :---: | --- |
| POST | `auth/register` | — | [01](01-auth.md#1-註冊) |
| POST | `auth/activate` | — | [01](01-auth.md#2-啟用帳號) |
| POST | `auth/login` | — | [01](01-auth.md#3-登入) |
| POST | `auth/logout` | ✅ | [01](01-auth.md#4-登出) |
| POST | `auth/password` | ✅ | [01](01-auth.md#5-變更密碼) |
| POST | `auth/password/forgot` | — | [01](01-auth.md#6-忘記密碼申請重設連結) |
| POST | `auth/password/reset` | — | [01](01-auth.md#7-重設密碼) |

### 商品與店面
| 方法 | 路徑 | 需要 token | 文件 |
| --- | --- | :---: | --- |
| GET | `storefront/products` | 選填 | [02](02-catalog.md#1-商品搜尋) |
| GET | `storefront/products/{product}` | 選填 | [02](02-catalog.md#2-商品詳情) |
| GET | `storefront/shops/{slug}` | — | [02](02-catalog.md#3-店面頁) |
| GET | `storefront/shops/{slug}/products` | 選填 | [02](02-catalog.md#4-店面商品列表) |
| GET | `storefront/categories` | — | [02](02-catalog.md#5-分類導覽) |

### 首頁與工會商城
| 方法 | 路徑 | 需要 token | 文件 |
| --- | --- | :---: | --- |
| GET | `storefront/home` | 選填 | [03](03-home-and-organizations.md#1-首頁) |
| GET | `storefront/organizations` | — | [03](03-home-and-organizations.md#2-工會商城列表) |
| GET | `storefront/organizations/{code}` | 選填 | [03](03-home-and-organizations.md#3-工會商城頁) |

---

## 前台**沒有**的東西

| 想做的事 | 為什麼沒有 | 走哪裡 |
| --- | --- | --- |
| 訪客加入購物車 | 購物車綁會籍 | 先登入，走 [buyer/cart](../buyer/03-cart-and-checkout.md) |
| 看賣家的成本或費率 | 那是賣家的營業資料（§2.3） | —— |
| 依「銷量」以外的方式看賣得多好 | 只暴露排序，不暴露銷量數字 | `sort=sales` |
| 看到未上架商品的任何痕跡 | 對前台不存在 | —— |
| 用 Email 或手機登入 | **Phase 1 唯一的登入識別碼是 `account`**（§X-05.1） | —— |

訪客不處理付款結果；未知付款不得由前台推定失敗或成功，登入後依買家訂單顯示。
詳見 [本地付款觀察](../../integration/payment-observations.md)。
> 退款執行界線（§14.4.3）：平台現有執行入口只允許 local／testing 的 demo 模擬；PAYUNi 等真實付款不得靠手填識別結案。demo 完成文案會明示未向金流送出退款，不代表實際入帳；本角色不取得財務執行權。詳見[平台退款說明](../platform/07-finance.md#3-執行退款財務)。

撥款同樣不以模擬結果代替外部入帳（§14.9.3）：現有財務 API／排程只接受本地 demo；本角色不取得執行權。
撥款讀回的 `execution_notice` 區分模擬／待核實／非正額遞延；真實付款不顯示模擬預計入帳日，見[撥款說明](../seller/07-payouts.md)。
