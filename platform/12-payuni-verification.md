# PAYUNi 核實與綁定歷史

spec §S-10.1c。僅有效平台管理員，預覽／確認需同時具有 `payuni.binding.verify` 與
`payuni.evidence.read`；批准替換既有綁定另需 `payuni.binding.manage`。
同一管理員可代建、收件及核實，不強制第二人，但每項授權／稽核獨立。
本地核實通過只證明平台記錄的歸屬判斷，不證明 PAYUNi 開通或可收款。
新建店申請若已有 A2 交付占用，一般預覽／確認不得結案，回 422（§S-10.1f）。
只有明列原 `delivery_claim_id` 的正向歸屬查核可核准，詳見下節（§S-10.1g）；撤回與拒絕仍不能解除占用。
目前交付準備僅為內部核心，尚無公開表單交付入口；本功能不代送表單。

## 核實原值受控查閱

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/verification-context
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `reason` | 是 | 1～1000 字元，不能全為 Unicode 空白；原樣記錄於讀取稽核 |

需 `payuni.binding.verify` 與 `payuni.evidence.read`，即使案件曾有綁定也不因此要求解除權。
200 共通信封；`data` 恰六欄：`application_id`、`shop_id`、`environment`、
`owner_member_id`（提交時店主）、`candidate_merchant_id`（候選原值）、`agent_id`（申請時快照）。
不從現在設定推導 AgentID；不回傳 Key／IV、IP、冪等鍵、原件或收款資格。

每次查閱都重新授權並先提交獨立稽核，包含重複請求；沒有省略留痕的冪等快取。
回應 `Cache-Control: private, no-store`。稽核或提交失敗不交付原值。
可查終態歷史，不會復活申請；店主後續變更不改寫上述提交快照。
錯配店面／缺權限 403，不存在 404，理由不合法 422，均使用共通信封。
一般申請、預覽、核實回條及綁定列表仍不揭露原值。

操作順序：開案 → 受控核對原值 → 選擇管理員確認，或收齊並閱讀文件 → 預覽 → 確認。
遇補件或作廢使舊預覽失效，須重新核對，不自動選取新版本或重按確認。

### 管理員確認（不強制圖片）

依 2026-09-30 業主決策，既有 `existing_merchant` 可在下列預覽及確認請求明列
`administrator_confirmed: true`、`approved: true`、`selected: []`，確認商家／代理商歸屬、
店主授權及本地店面對應。確認仍需 revision、UUID 與理由，授權要求不變。
無需上傳圖片，不製造文件或原件讀取紀錄。預覽／確認回條額外提供
`confirmation_basis: "admin_confirmation"`；此依據及操作者、時間、IP、理由不可改寫。
不能混入文件、拒絕、新商家或 A2 `delivery_claim_id`。省略或 false 維持文件模式，空集合仍拒絕。
核實後只是本地歸屬綁定，不開啟收款、Notify 或財務操作。

## 核實預覽

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/verification/preview
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `selected` | 是 | 文件模式為 JSON 陣列 2～100 筆；至少各一份有效歸屬與授權證據；管理員確認須明列空陣列 |
| `selected[].id` | 是 | JSON 正整數，不能重複，必須屬本案 |
| `selected[].version` | 是 | JSON 正整數，與該 ID 的版本完全一致；子物件只接受 id／version |
| `approved` | 是 | JSON `true` 或 `false`；不可省略、不接受字串或 0／1 |
| `administrator_confirmed` | 否 | JSON 布林，預設 false；true 僅限既有商家正向確認，不能有文件或 delivery_claim_id |
| `delivery_claim_id` | 否 | 原 A2 占用的 JSON 正整數；提供時不接受 null、字串或布林，且只能搭配 `approved=true` |

200 共通信封；`data` 包含 `application_id`、`shop_id`、`environment`、`approved`、
依 ID 排序的 `selected`、`observed_binding_id`（可 null）、`observed_binding_version`、`revision`。
不提供候選 MerID、AgentID、原件或可收款旗標。預覽不保留名額、不建立核實結果。
拒絕也須固定兩用途的有效證據；資料不足應補件或撤回，不假造核實拒絕。
`payment_settings` 是付款設定原件，不得選入本歸屬核實集合，包含在兩種必要文件之外額外加入也回 422。

## 確認核實或拒絕

```text
POST platform/shops/{shop}/payuni-applications/{payuniApplication}/verification
```

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `selected` | 是 | 與預覽相同；文件模式 2～100 筆，管理員確認為空陣列 |
| `selected[].id` | 是 | JSON 正整數，不重複 |
| `selected[].version` | 是 | JSON 正整數；子物件只接受 id／version |
| `approved` | 是 | 與預覽相同的 JSON 布林意圖 |
| `administrator_confirmed` | 否 | 與預覽相同的 JSON 布林模式；true 表示管理員明確確認三項歸屬／授權事實 |
| `revision` | 是 | 預覽取得的 64 碼小寫十六進位；不修剪 |
| `verification_key` | 是 | 正規小寫 UUID |
| `reason` | 是 | 1～1000 字元，不能全為 Unicode 空白；原樣保存 |
| `delivery_claim_id` | 否 | 正向交付歸屬查核時須與預覽相同的原占用 JSON 正整數；重播也不得省略或替換 |

確認前重新授權、CAS 申請／店主／全部證據與作廢歷史／綁定觀察，並受控探測所選原件。
過期預覽或檔案不符 422，不會只靠前次預覽放行。成功與同人同鍵同內容重播皆 200：
文件模式的 `data` 只有 `id`、`application_id`、`shop_id`、`environment`、`status`（`approved`／`rejected`）、`decided_at`。
管理員確認另有 `confirmation_basis=admin_confirmation`；重播保留相同模式，不能改成文件核實。
批准將申請結案為 verified 並原子建立下一版綁定；拒絕結案為 rejected，不建立身分或綁定。
重播須保留原 revision、選取集合、意圖與理由；更換鍵或改內容不是重播。撤權後連重播也拒絕。

### 原交付占用的正向歸屬查核

spec §S-10.1g。僅限 sandbox 的 `new_merchant` 原申請，原店主及綁定觀察必須仍相符；
`delivery_claim_id` 必須屬同一申請，不從目前設定或呼叫端 MerID 重建範圍。
兩種用途的證據均須在原占用之後或同秒登錄；所選供應商歸屬紀錄的取得時間也不得早於占用。
管理員仍須透過原值查閱與原件下載核對實際內容；時間、收件與檔案完整性不保證內容真實。

預覽／確認／回應白名單沿用上節；修訂及冪等內容另固定明示占用，補件、作廢或綁定異動須重新預覽。
成功將原占用與核實決定建立不可變關聯，和原申請結案、原候選身分及綁定同交易提交；不刪除交付歷史。
這只記錄原商家歸屬已核實，不證明該次瀏覽器表單或 PAYUNi 建店動作成功，也不提供可收款資格。
查無、負向狀態、`approved=false` 或缺少證據不能用來解除占用／換號。
Bruno `payuni-delivery-review-*` 與 J17 使用獨立合成 seed，占用之外的證據及核實均經 API 建立。

## 綁定歷史

```text
GET platform/shops/{shop}/payuni-bindings
```

需 `payuni.binding.manage`，不由一般開店審核或申請管理權隱含取得。

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `environment` | 否 | `sandbox` 或 `production` |
| `page` | 否 | 正整數，預設 1 |
| `per_page` | 否 | 預設 20，上限 100 |

只列 URL 店面，ID 新到舊排序，含已解除版本；分頁在 `meta.pagination`。
每列十欄：`id`、`shop_id`、`environment`、`application_id`、`verification_id`、`version`、
`active`、`created_at`、`released_at`、`release_reason`。未解除時後兩欄 null。
`active` 只表示本地綁定未解除，絕非收款資格。無外部身分識別、IP、冪等鍵或原件。

## 解除指定版本

```text
POST platform/shops/{shop}/payuni-bindings/{payuniBinding}/release
```

需 `payuni.binding.manage`，巢狀店面錯配 403，版本不存在 404。

| 欄位 | 必填 | 規則 |
| --- | --- | --- |
| `release_key` | 是 | 正規小寫 UUID |
| `reason` | 是 | 1～1000 字元，非全 Unicode 空白，原樣保存 |

成功及同人同鍵同理由重播 200，`data` 與綁定列表相同十欄。不同鍵或內容 422。
只追加解除事實、不刪歷史、不復活舊版，不改寫舊交易快照。

以上已提供原值受控查閱、證據及核實 API，但不是完整核實 UI；一般回應仍不能推導外部身分。
A2 交付待查案另受 §S-10.1f 限制，不得因既有商家核實 API 可用就繞過交付結果查核。
