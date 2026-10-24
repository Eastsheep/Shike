# 食刻熱量日記

手機用的飲控紀錄網頁：拍照或文字描述交給 DeepSeek 估算熱量，也能手動記錄。紀錄存在你自己的私人 GitHub repo。

## 架構

| 部分 | 放哪裡 |
|---|---|
| 網頁（這個 repo） | GitHub Pages，公開 |
| 飲食紀錄 | 另一個**私人** repo（例如 `shike-data`），`data/YYYY-MM.json` 每月一個檔，`profile.json` 存目標 |
| AI 估算 | DeepSeek API，模型 `deepseek-flash` |

GitHub token 和 DeepSeek API key 只存在手機瀏覽器的 localStorage，不會寫進這個 repo。

## 設定步驟

1. **網頁 repo**：建立公開 repo（例如 `shike`），上傳 `index.html`、`manifest.webmanifest`、`icon.svg`、`README.md`。
   到 Settings → Pages → Branch 選 `main`、資料夾 `/ (root)` → Save。網址會是 `https://<帳號>.github.io/shike/`。
2. **資料 repo**：建立**私人** repo `shike-data`，勾選 *Add a README file*。
3. **GitHub token**：Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token。
   - Repository access：Only select repositories → `shike-data`
   - Permissions → Repository permissions → Contents：Read and write
   - 有效期限自己決定，到期後在 App 設定裡換新的
4. **DeepSeek API key**：到 platform.deepseek.com 建立 API key，儲值少量金額即可。
5. 手機打開網址 → 右上角齒輪 → 填入帳號、`shike-data`、token、DeepSeek key → 測試連線 → 儲存。
6. 加入主畫面：iPhone 用 Safari 的「分享 → 加入主畫面」；Android 用 Chrome 選單的「加到主畫面」。

## 注意

- 換手機或清除瀏覽器資料後，要重新貼 token 和 key。紀錄本身在 GitHub 上，不會不見。
- token 外流的話，到 GitHub 把它撤銷（Revoke）再建一個新的。
- AI 估算只是參考，加入前可以修改數字。
