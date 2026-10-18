# 工會後台 API

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
原 UPP run 的 `query-trade` 亦僅限平台人工唯讀查詢，不授予工會管理員金流查詢或實驗管理權限。
缺欄位後的 `diagnose-trade` 亦僅限平台人工取證，不提供工會入口。
新版解析器的 `verify-query-parser` 同樣僅限平台一次唯讀驗收，不提供工會入口。

原 A2 交付歸屬查核僅供具獨立權限的平台管理員操作，工會沒有核實或解除交付占用入口。

[A2 自動配號申請](../platform/10-payuni-applications.md#自動配號的新建店申請) 初期由平台代建；
工會無配號入口，管理工會的權限不等於平台的 `payuni.application.manage`。

付款讀回的 `payuni_query` 只公開最新本地查核摘要，沿用同一會員授權；matched 不是已付款，GET 不發動金流查詢。

[買家付款讀回](../buyer/04-orders-and-payment.md#3a-查核本地付款) 只限下單會員本人；工會管理權不授予讀取所屬會員付款的權利。

工會管理權不隱含 PAYUNi 商家申請、核實、核實原值查閱或證據讀取權。
申請由 [店主](../seller/10-payuni-applications.md) 或 [有獨立授權的平台管理員](../platform/10-payuni-applications.md) 操作，工會沒有對應路由。

路由前綴：`union/`（完整路徑為 `/api/union/...`）
認證：**會員 token**（`Authorization: Bearer {member_token}`）

> 先讀 [共通約定](../conventions.md)。回應信封、分頁、錯誤碼、遮罩規則都在那裡，本目錄不再重複。

## 工會管理者是「會員」，不是「平台管理員」

這是串接工會後台最容易搞錯的一件事：

- 工會管理者用的是**會員側登入**（`POST auth/login`），拿到的是**會員 token**。
- 平台管理員的 token 打 `union/*` 會在**認證層**被拒（401），反之亦然。
- 「他是工會管理者」不是一種帳號類型，而是**某個會籍被授予了 `union_admin` 角色**，
  且該角色的範圍（`scope_id`）指向某一個組織。

因此同一個人可能同時是某工會的管理者、另一家店的賣家、以及買家 —— **同一筆會籍的三個面向**，
用的是同一組帳號密碼、同一個 token。

### 登入的那組帳號決定你看得到哪個工會

工會端所有端點都以**這個 token 所屬會籍的組織**為租戶範圍（§X-05.1）。

一個自然人在兩個工會都有幹部身分時，會有**兩組帳密**，各自登入 —— 兩張 token 同時有效，
互不影響，也不會有「另一個分頁改掉了我的身分」這回事。以 A 工會的帳號去打 B 工會的資料
一律 403，那是租戶隔離，不是需要先切換。

| 端點 | 用途 |
| --- | --- |
| `POST auth/login` | 登入。回應的 `membership` 就是這組帳號的身分與所屬組織 |
| `GET buyer/profile` | 帳號、會籍、名下店面與兩項資格，一次取得 |

> 這兩支屬於買家中心／認證，不在 `union/` 前綴底下，但工會後台的前端一定會用到。
>
> **沒有切換會籍的端點**：帳密由各組織核發，登入的那一刻身分就定了（§X-05.1）。
> 身兼兩個工會幹部的人有兩組帳密，各自登入。

---

## 租戶邊界（§2.3）

**工會管理者不得檢視其他工會的會員資料或交易紀錄。** 這條約束在每個端點都成立，
而且是三道閘門的第三道（身分主體 → 權限 → 租戶範圍）。

實務上的意思：

- 大部分端點**根本不接受組織參數**，一律用你自己的工會。
- 少數接受 `organization_id` 的端點（總覽、會員清單、違規記錄），
  帶了別的工會 ID 只會得到 403，不會得到資料。
- 403 的訊息會明確說是「不得跨工會存取」還是「沒有這個授權」，可直接顯示。

## 工會端做不到的事

前端不要留這些按鈕 —— 它們在工會端**沒有端點**：

| 想做的事 | 為什麼不行 | 誰能做 |
| --- | --- | --- |
| 建立／修改**會籍主檔** | `membership_status` 是會務端認定的會籍事實，商城唯讀（§X-05.2） | 平台（`platform/members`） |
| **關閉**店面 | 關閉是不可逆處置，工會只能暫停並轉平台（§P-07.4） | 平台（`platform/shops/{shop}/close`） |
| 解除**平台**所下的停權 | 越級解除以 403 拒絕並留稽核（§P-07.4） | 平台（`platform/shops/{shop}/resume`） |
| 裁定違規、撤銷違規記錄 | 「不得裁定」（§P-07.3） | 平台（`platform/violations`） |
| 標記違規為**重大**（`is_major`） | 重大違規不受點數門檻限制、直接停權，等於繞過分工（§P-07.1） | 平台 |
| 執行退款與撥款 | 工會不碰金流（§P-07.4） | 平台財務 |
| 審核法人機構的開店申請 | 法人申請由平台審核（§C-01） | 平台（`platform/shop-applications/{application}/decision`） |
| 代賣家上架／下架商品 | 商品由賣家維護 | 賣家（`seller/products`） |
| 改自己工會的統一編號 | 「不得由工會端自行變更」（§2.2.1） | 平台（`platform/organizations/{id}`） |

### 工會**做得到**的處置（容易被誤以為不行）

§P-07.4 給了工會三項處置權，前端請保留這些按鈕：

| 做得到 | 端點 | 邊界 |
| --- | --- | --- |
| 暫停旗下店面、解除**自己所下**的暫停 | `PATCH union/shops/{shop}/suspend`｜`resume` | 解不掉平台所下的停權 |
| 會員的**商城側**停權／恢復 | `PATCH union/members/{member}/marketplace-status` | 不改寫 `membership_status` |
| **核發旗下會員的商城帳號**（含重發憑證） | `POST union/members/{member}/account` | 只發帳號，仍不改寫會籍主檔；`offline` 管道需平台另行授予 |
| 建立違規記錄 | `POST union/violations` | 強制 `source = manual`，不得標 `is_major`；點數**不計入停權門檻** |

---

## 端點總表

共 28 條路由，全部需要會員 token。

### 查詢（唯讀）
| 方法 | 路徑 | 需要權限 | 文件 |
| --- | --- | --- | --- |
| GET | `union/overview` | `order.view` @ 本工會 | [01](01-overview-and-queries.md#1-工會商城總覽) |
| GET | `union/orders` | `order.view` @ 本工會 | [01](01-overview-and-queries.md#2-本工會店面的訂單) |
| GET | `union/members` | `member.view` @ 本工會 | [01](01-overview-and-queries.md#3-本工會會員清單) |
| GET | `union/violations` | `violation.view` @ 本工會 | [01](01-overview-and-queries.md#4-本工會店面的違規記錄唯讀) |

### 審核
| 方法 | 路徑 | 需要權限 | 文件 |
| --- | --- | --- | --- |
| GET | `union/shop-applications` | `shop.review` @ 本工會 | [02](02-reviews.md#1-店面申請清單) |
| POST | `union/shop-applications/{application}/decision` | `shop.review` @ 本工會 | [02](02-reviews.md#2-店面審核決議) |
| GET | `union/product-reviews` | `product.review` @ 本工會 | [02](02-reviews.md#3-商品審核佇列) |
| POST | `union/product-reviews/{product}/decision` | `product.review` @ 本工會 | [02](02-reviews.md#4-商品審核決議) |

### 商城內容
| 方法 | 路徑 | 需要權限 | 文件 |
| --- | --- | --- | --- |
| GET | `union/announcements` | `announcement.manage` @ 本工會 | [03](03-storefront-content.md#1-公告列表) |
| POST | `union/announcements` | `announcement.manage` @ 本工會 | [03](03-storefront-content.md#2-建立公告) |
| PATCH | `union/announcements/{announcement}` | `announcement.manage` @ 本工會 | [03](03-storefront-content.md#3-更新公告) |
| DELETE | `union/announcements/{announcement}` | `announcement.manage` @ 本工會 | [03](03-storefront-content.md#4-下架公告) |
| GET | `union/featured-slots` | `featured_slot.manage` @ 本工會 | [03](03-storefront-content.md#5-推薦版位列表) |
| POST | `union/featured-slots` | `featured_slot.manage` @ 本工會 | [03](03-storefront-content.md#6-加入推薦版位) |
| PATCH | `union/featured-slots/{featuredSlot}` | `featured_slot.manage` @ 本工會 | [03](03-storefront-content.md#7-調整排序) |
| DELETE | `union/featured-slots/{featuredSlot}` | `featured_slot.manage` @ 本工會 | [03](03-storefront-content.md#8-移出推薦版位) |

### 店面與會員管理（U-05）
| 方法 | 路徑 | 需要權限 | 文件 |
| --- | --- | --- | --- |
| GET | `union/shops` | `shop.review` @ 本工會 | [04](04-shop-and-member-management.md#1-旗下店面清單) |
| GET | `union/shops/{shop}` | `shop.review` @ 本工會 | [04](04-shop-and-member-management.md#2-店面詳情) |
| GET | `union/shops/{shop}/products` | `product.review` @ 本工會 | [04](04-shop-and-member-management.md#3-店面商品查詢) |
| PATCH | `union/shops/{shop}/suspend` | `shop.dispose` @ 本工會 | [04](04-shop-and-member-management.md#4-暫停店面) |
| PATCH | `union/shops/{shop}/resume` | `shop.dispose` @ 本工會 | [04](04-shop-and-member-management.md#5-解除暫停) |
| GET | `union/members/{member}` | `member.view` @ 本工會 | [04](04-shop-and-member-management.md#6-會員詳情與訂單摘要) |
| PATCH | `union/members/{member}/marketplace-status` | `member.marketplace_dispose` @ 本工會 | [04](04-shop-and-member-management.md#7-會員商城側停權與恢復) |
| POST | `union/members/{member}/account` | `member.account_issue` @ 本工會（`offline` 另需 `member.credential.offline`） | [04](04-shop-and-member-management.md#8-核發商城帳號) |
| POST | `union/violations` | `violation.record` @ 本工會 | [04](04-shop-and-member-management.md#9-建立違規記錄) |
| POST | `union/members/export` | `member.view` @ 本工會 | [04](04-shop-and-member-management.md#10-匯出會員商城統計) |
| GET | `union/members/exports` | `member.view` @ 本工會 | [04](04-shop-and-member-management.md#11-匯出紀錄) |
| GET | `union/members/exports/{export}/download` | `member.view` @ 本工會 | [04](04-shop-and-member-management.md#12-下載匯出檔) |

## `union_admin` 預設權限

系統預設角色 `union_admin`（工會管理者）帶**十個**權限，範圍為 `organization`：

| 權限 | 用途 |
| --- | --- |
| `shop.review` | 店面申請審核 |
| `shop.dispose` | 暫停旗下店面、解除自己所下的暫停（**不含關閉**，§P-07.4） |
| `product.review` | 商品審核 |
| `order.view` | 總覽與訂單查詢 |
| `member.view` | 會員清單、詳情與統計匯出 |
| `member.marketplace_dispose` | 會員的商城側停權／恢復（**不改寫會籍主檔**，§U-05.2） |
| `announcement.manage` | 公告 |
| `featured_slot.manage` | 推薦版位 |
| `violation.view` | 違規記錄唯讀 |
| `violation.record` | **建立**違規記錄（**不含**撤銷與裁定，§P-07.4） |

> `violation.record` 與 `violation.manage` 是刻意分開的兩個權限。後者的語意是
> 「處置違規與停權」，含撤銷與裁定 —— 合成一個，工會就連帶取得裁定申訴的能力，
> 而申訴一律由平台裁定（§P-07.3）。`shop.dispose` 與 `member.marketplace_dispose`
> 是同一個切法。

權限可由平台調整組合，前端不要寫死「有 `union_admin` 就顯示全部選單」，
而應該依實際被拒（403）或由平台端提供的角色設定決定。

> `union_admin` **不得授予平台預設組織的會籍**（該組織不承載店面，也沒有工會管理職能）。

付款未知不列為成功營收，也不授予工會改寫付款結果的權限；詳見
[本地付款觀察](../../integration/payment-observations.md)。
> 退款執行界線（§14.4.3）：平台現有執行入口只允許 local／testing 的 demo 模擬；PAYUNi 等真實付款不得靠手填識別結案。demo 完成文案會明示未向金流送出退款，不代表實際入帳；本角色不取得財務執行權。詳見[平台退款說明](../platform/07-finance.md#3-執行退款財務)。

撥款同樣不以模擬結果代替外部入帳（§14.9.3）：現有財務 API／排程只接受本地 demo；本角色不取得執行權。
撥款讀回的 `execution_notice` 區分模擬／待核實／非正額遞延；真實付款不顯示模擬預計入帳日，見[撥款說明](../seller/07-payouts.md)。
