# 專案規則

## 溝通原則
- 指令中有任何不確定或不清楚的地方，**必須先向用戶確認**，再開始執行。

## Git 工作流程
- 分支完成後，直接 merge 到 main 並推送，不需要等用戶另外指示。
- 推送前確認 remote URL 的 token 有效；若失效請向用戶索取新 token。
- 每次修改 `travel-app.html` 後，必須同步更新 `index.html`（Vercel 以 `index.html` 為入口）。
