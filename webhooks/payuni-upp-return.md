# UPP 前景返回及返回商店

spec §B-02／§12.1；M1 ReturnURL 為 Form POST，BackURL 為返回商店按鈕。

```
POST webhooks/payuni/upp/return/{orderNo}
GET webhooks/payuni/upp/back/{orderNo}
POST webhooks/payuni/upp/return
GET webhooks/payuni/upp/back
```

付款表單的 ReturnURL／BackURL 綁原訂單編號（`{orderNo}` 為 `M` 加 16 位數字，不符為 404）；不帶編號的兩條供 2026-10-15 前已交付的舊表單使用。

**前端返回網址（業主 2026-10-15）**：部署設定 `PAYUNI_FRONTEND_RETURN_URL`（https；本機開發可 `http://localhost`／`127.0.0.1`；不得帶 query、fragment 或帳密）時，回 **303** 導回該網址並附 `?order_no={orderNo}`（舊路由不附）。前端頁以原買家 token 呼叫 `GET buyer/orders/{order_no}/payment-status` 確認結果。設定缺漏或不合格時維持下方 JSON 提示。導向目的地只取部署設定，不採用 query 或 body 內的任何網址。

未設定前端返回網址時：無需登入，回 ApiResponse 200，data 固定為 `{"payment_confirmation_required":true}`。
完全不取用／回顯 body、query、Header 內的交易內容或網址；不查 DB、不呼叫金流，
不建立通知、工作或付款事件，也不取消訂單。回應 no-store／no-referrer。
這不是 Notify ACK，不表示付款成功。前端須回原商城視窗，經原買家授權讀取付款狀態。
