# 食刻熱量日記（Shike）

給自己和家人用的手機飲控網頁 App。部署在 GitHub Pages：https://eastsheep.github.io/Shike/ （repo `Eastsheep/Shike`，網址的 `Shike` 大小寫要一致）。

**和使用者溝通一律用繁體中文。** 使用者不是工程師，說明要具體到「點哪裡、看到什麼」。

## 檔案

| 檔案 | 用途 |
|---|---|
| `index.html` | 整個 App（HTML + CSS + JS 都在裡面，無建置步驟、無框架） |
| `foods.js` | 台灣常見食物清單 `window.SHIKE_FOODS`，每筆 `[名稱, 類別, "g"/"ml", 份量名稱, 份量g或ml, kcal, 蛋白質, 碳水, 脂肪]`，數值是「一份」的值 |
| `manifest.webmanifest`, `icon.svg`, `icon-*.png` | 加入主畫面用（iOS 不吃 SVG 圖示，所以 apple-touch-icon 用 `icon-180.png`） |
| `splash/` | iOS 加到主畫面後的原生啟動畫面，`light-/dark-寬x高.png`，各 iPhone 尺寸一張，版面和 App 內的 `#splash` 一樣 |
| `README.md` | 給使用者看的設定步驟（含 iOS 捷徑步驟） |

部署方式：使用者自己在 GitHub 網頁「Add file → Upload files」覆蓋檔案。改完程式要提醒他上傳哪些檔。

## 資料與同步

- 紀錄存在使用者的**私人** repo（例如 `shike-data`），用 GitHub Contents API 讀寫，token 是 fine-grained PAT（只開該 repo 的 Contents 讀寫）。
  - `data/YYYY-MM.json`：`{days:{"YYYY-MM-DD":[item...]}, activity:{"YYYY-MM-DD":{active,steps,src,t}}}`
  - `profile.json`：`{goal:{kcal,p,c,f}, calc:{bmr,tdee,factor,sel,adj,kcal,formula,lbm}, body:{sex,age,h,w,bf,act,goal}, dyn:{on,mode,ratio}}`（`mode` 為 `lazy`/`precise`；舊資料只有 `on`，`on:true` 視為精打細算）
- 本機快取在 localStorage `shike:cache`，設定（GitHub 帳號/repo/token、DeepSeek key/model）在 `shike:cfg`。token 和 key 只存在瀏覽器，絕不寫進 repo。
- 食物 item：`{id,meal(b/l/d/s),src(photo/text/manual),name,portion,kcal,p,c,f,t, q?,u?,fn?}`；`fn` 是 foods.js 的名稱，`q/u` 是數量與單位，編輯時用來還原。
- 同步：`dirtyMonths` 記錄待上傳月份；PUT 遇 409/422 會重抓再合併。**活動消耗用時間戳 `t` 取較新的那筆**（`newerAct`）。
- 食物以 item 為單位合併（`mergeDays`）：本頁沒有未上傳修改時直接用遠端；有的話依 id 聯集、`mt||t` 較新者勝、刪除記在月檔 `del:{id:{k,t}}`（保留 60 天），讓舊頁面不能把刪掉的東西帶回來、也不會覆蓋別頁新增的項目。頁面回到前景／取得焦點／bfcache 還原時會重新拉資料（`refreshFromGitHub`，5 秒節流）。
- 月檔格式：`{days, activity, del}`。
- 健康數據新管道：捷徑直接 PUT `health/YYYY-MM-DD_yyyyMMddHHmmss.json`（內容 `{"d","a","s"}`，免 sha）。`pullHealth` 列出資料夾、每天取最新一檔套用（手動值較新則保留手動）、清掉舊檔；`healthSeen` 記錄已處理的時間。舊的 `#d=&a=&s=` 網址匯入仍保留。
- 家人共用：同一個網址，各自在設定裡填不同的資料 repo 和 token。
- 我的食物：資料 repo 的 `myfoods.json` = `{foods:[{id,name,kcal,p,c,f,portion,q,u,fn,photo,t}]}`，刪除留 `{id,deleted:true,t}` 墓碑，合併依 `t`（`mergeMy`）。同名存檔會覆蓋同一筆；沒有數量的（如 AI 結果）存成 1 份。從手動視窗頂端或搜尋結果帶入，照片沿用原 id 不重傳；帶入時「存到我的食物」預設不勾，避免改掉基準份量。
- 照片：item 的 `photo` 是 id（`YYYY-MM/<uid>`），檔案在資料 repo `photos/<id>.jpg`（約 1080px、JPEG 0.75）。本機存在 IndexedDB `shike-photos`，待上傳清單在 `cache.pendingPhotos`，由 `sync()` 上傳；其他裝置用 `Accept: application/vnd.github.raw` 按需下載（失敗 60 秒內不重試）。AI 拍照估算加入時，同一張照片會掛在該次所有食物上。刪除食物不會刪 GitHub 上的照片檔。

## AI 估算

- DeepSeek `https://api.deepseek.com/chat/completions`，預設模型 `deepseek-flash`（支援圖片，image_url 用 base64 data URL），瀏覽器可直接呼叫（CORS 可用）。
- 照片先用 canvas 縮到 1280px JPEG。要求回傳 JSON `{items:[{name,portion,kcal,protein,carbs,fat}],note}`，解析時容忍 code fence。
- 結果讓使用者勾選、改數字、選 ×0.5～×2 後才加入；加入後清空照片與文字。

## 目標計算

- 計算機：有體脂率用 Katch-McArdle（370 + 21.6 × 去脂體重），否則 Mifflin-St Jeor；TDEE = BMR × 活動係數；目標 = TDEE + 調整（−500/−300/0/+250），取整到 10，有最低值。蛋白質依體重（或去脂體重）。按「算出並套用建議值」會直接儲存。
- 「熱量估算方式」二選一（`isPrecise`）：
  - 懶人估算：TDEE 用所選活動量，每天目標固定，活動紀錄不加到目標。
  - 精打細算：基礎目標以久坐（×1.2）計算，活動量選單鎖住；有活動紀錄（健康 App 或手動）的日子加 動態能量 × 計入比例；沒有紀錄的日子**不估算、不加**（使用者覺得估算會在沒運動時高估）。多出的熱量依基礎目標的蛋白質／碳水／脂肪熱量比例分配。
  - 在目標頁切換模式時，若已有計算紀錄且資料齊全會自動重算並套用。
  - 「查看計算過程」（`renderBreak`）跟著目標頁目前選的模式／活動量／目標即時重畫（用已存的 BMR 重算）；和已套用的目標不一致時提示要按「算出並套用」。
  - 近 7 天圖表參考線（`tdeeFor`）：懶人＝固定 TDEE；精打細算＝久坐 TDEE ＋ 當天完整活動消耗。
- 計算過程只顯示在「目標」頁，預設收合（使用者不要在首頁看到公式）。

## iOS 捷徑「食刻健康」串接

- App 的活動視窗按鈕開 `shortcuts://run-shortcut?name=食刻健康&input=text&text=YYYY-MM-DD`（目前檢視的日期）。
- 捷徑讀健康 App「動態能量」和「步數」該日總和，打開 `https://eastsheep.github.io/Shike/#d=日期&a=活動kcal&s=步數`。
- `importFromHash` 寬鬆解析日期（2026-10-22、2026/10/22、2026年10月22日、後面帶時間都可），看不懂就報錯，**不可默默改用今天**；數字會去掉千分位逗號與單位。
- 捷徑從 Safari 開啟，Safari 也要設定 GitHub token 才會同步。

## 設計與 UI 慣例

- 開啟畫面 `#splash`：logo＋「食刻」，第一次畫面完成且字型載好就淡出，最晚約 1 秒，不做人為等待；JS 壞掉時 CSS 動畫 1.6 秒後也會自己藏起來。「食刻」兩字的字型子集以 base64 內嵌，不會跳字。改 logo 或配色時要重產 `splash/*.png` 與 `icon-*.png`（用 Playwright 依 `#splash` 版面截圖）。

- 手機優先單欄（max 560px），色彩全用 CSS 變數並支援深色模式；字體 Noto Serif TC（標題）、Noto Sans TC（內文）、IBM Plex Mono（數字）。
- 三大營養素用彩色實心圓點表示（紅＝蛋白質、黃＝碳水、藍＝脂肪），不用 P/C/F 字母。
- 使用者試過在餐別／營養素前加圖示後要求移除，不要再加。
- 手動加入：食物清單只在點搜尋框後出現，選完自動收起；份量用「數量＋單位」選單（g/oz/台兩、ml/cc/fl oz/量杯，加上該食物的常見份量），切換單位時保持同樣份量。

## 本機測試

這台電腦沒有 Node/Python 時，用 PowerShell `System.Net.HttpListener` 寫個小靜態伺服器提供 `shike-app/`，在瀏覽器以手機尺寸（375×812）測試。需要 GitHub / DeepSeek 時用覆寫 `window.fetch` 的假回應測，不要打真的 API。測完清掉 localStorage。
