# 前台 API（訪客與買家共用）

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

共 13 條路由。除登出與變更密碼外都不需要 token。

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

### 商品與店面
| 方法 | 路徑 | 需要 token | 文件 |
| --- | --- | :---: | --- |
| GET | `storefront/products` | 選填 | [02](02-catalog.md#1-商品搜尋) |
| GET | `storefront/products/{product}` | 選填 | [02](02-catalog.md#2-商品詳情) |
| GET | `storefront/shops/{slug}` | — | [02](02-catalog.md#3-店面頁) |
| GET | `storefront/shops/{slug}/products` | 選填 | [02](02-catalog.md#4-店面商品列表) |

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
