# 商城職行家 API 文件（前端串接用）

本目錄是給前端串接的 API 說明。**規格的唯一真實來源仍是 [spec.md](../../spec.md)**；
本文件只描述目前 `routes/api.php` 上真正存在的端點與其請求／回應形狀。
兩者若有出入，以 spec 為準並回報修正。

## 閱讀順序

1. [共通約定](conventions.md) —— 回應信封、認證、錯誤碼、分頁、遮罩、金額與天數單位。
   **所有端點都套用這一份，各分類文件不再重複。**
2. 各角色的端點文件：

| 角色 | 路由前綴 | 文件 |
| --- | --- | --- |
| 平台後台 | `/api/platform/*` | [platform/](platform/README.md) |
| 前台（訪客／買家） | `/api/storefront/*`、`/api/auth/*`、`/api/health` | [storefront/](storefront/README.md) |
| 買家中心 | `/api/buyer/*` | [buyer/](buyer/README.md) |
| 賣家中心 | `/api/seller/*` | [seller/](seller/README.md) |
| 工會後台 | `/api/union/*` | [union/](union/README.md) |
| 回調 | `/api/webhooks/*` | **不寫前端文件**（金流／物流商用，非前端串接） |

## 與測試、Bruno 集合的關係

API 是本專案唯一的公開介面，因此三份東西必須同時成立：

- `tests/Feature/Api/` —— 行為的驗收依據。
- `tests/Bruno/` —— 可直接執行的請求範例（買家側在 `02-buyer/`、賣家側在 `03-seller/`、平台側在 `05-platform/`）。
- 本目錄 —— 給前端看的說明。

端點有異動時三者一起改。若本文件與 Bruno 集合不一致，以程式碼與測試為準。

### 端點總表由測試釘住，不由人維護

`Feature/Integration/ApiDocsCoverageTest` 兩個方向都比對：文件列出的端點必須真的
註冊過（否則前端串下去是 404），而每一條路由都必須出現在端點總表。
「共 N 條路由」那句話與內部相對連結也一併檢查。

它不驗欄位與語意 —— 那些要靠人讀，而人只會讀他相信還沒過期的文件。

**五個角色的文件都已撰寫，全部納入該測試。** 唯一的例外是 `webhooks/` ——
它是金流與物流商用的回調，契約由 `tests/Bruno/07-webhooks/` 與驗簽測試把關。

那個例外本身也被釘住：`test_未撰寫文件的前綴僅剩_webhooks` 會在有人新增
任何其他前綴而沒有補文件時變紅。
