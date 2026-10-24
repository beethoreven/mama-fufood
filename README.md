# mama-fufood

## 中文

「媽媽記副食品」寶寶副食品紀錄 App 的專案總覽 repo,透過 git submodule 把三個子系統掛在一起,方便一次 `clone` 齊全部程式碼。這個 repo 本身**不會被部署**——各子系統各自連到自己的 repo 做部署或打包(見下表)。各子系統的技術細節與架構決策,請見各自 repo 的 README。

紀錄只存在手機與使用者自己的 Google 雲端硬碟;後端只管公用資料(食材清單)、每日檢查與意見回饋,不碰任何寶寶的資料。本 App 僅供記錄與參考,非基於醫學知識做的判斷。

### 子系統

| 子系統 | Repo | 部署/產出 |
|---|---|---|
| Android App | [`mama-fufood-app`](https://github.com/beethoreven/mama-fufood-app) | 本機打包成 APK,用 `adb` 安裝到手機(還沒有上架 Google Play) |
| 後端 API | [`mama-fufood-backend`](https://github.com/beethoreven/mama-fufood-backend) | Render(預計 `https://mama-fufood-backend.onrender.com`) |
| 網站與管理後台 | [`mama-fufood-web`](https://github.com/beethoreven/mama-fufood-web) | GitHub Pages(`https://beethoreven.github.io/mama-fufood-web/`) |

資料庫是 Neon Postgres 18,開在 **ap-southeast-1(Singapore)**,與 Render 同區。這不是效能微調而是前提:姊妹專案 zh-cn-to-tw 實測,兩端跨區時單次查詢 1.76 秒,同區 0.03 秒,差 59 倍。Neon 的區域建好之後不能改。

### 使用方式

**第一次拿到全部程式碼:**

```bash
git clone --recurse-submodules git@github.com-beethoreven:beethoreven/mama-fufood.git
```

(`github.com-beethoreven` 是開發者自己電腦上的 SSH 別名;其他人改用 `https://github.com/beethoreven/mama-fufood.git`。)

**已經 clone 過,只是要把子模組抓齊:**

```bash
git submodule update --init --recursive
```

**更新某個子模組到它自己 repo 的最新版本:**

```bash
cd mama-fufood-backend   # 或 mama-fufood-app、mama-fufood-web
git pull origin main
cd ..
git add mama-fufood-backend
git commit -m "Update mama-fufood-backend submodule pointer"
```

最後這個 `git add` + `commit` 不能省略——沒做的話,子模組本機檔案雖然是新的,但這個 wrapper repo 記錄的「該指向哪個 commit」還是舊的,別人 `clone` 這個 repo 還是會抓到舊版本。

---

### 上線 SOP

★ 上線步驟刻意放在這裡而不是各子系統的 README,因為它**跨越三個子系統與 Google Cloud**,有幾條交叉引用,拆開之後就沒有人能照著做完:

1. Google Cloud「網頁應用程式」用戶端 ID → 後端的 `GOOGLE_CLIENT_ID`、網站 `js/config.js` 的 `googleClientId`、App `env/prod.json` 的 `MFF_WEB_CLIENT_ID`(三個地方是同一個值)。
2. 網站的網址 → 後端的 `ALLOWED_ORIGINS`、Google 用戶端的「已授權的 JavaScript 來源」、Google 同意畫面的「已授權網域」與連結。
3. 後端的網址 → 網站 `js/config.js` 的 `apiBase`、App `env/prod.json` 的 `MFF_API_BASE`、cron-job.org 的保活網址。
4. App 簽署金鑰的 SHA-1 → Google Cloud 的 Android 用戶端。

照 Part 0、A~J 的順序做。外部服務的帳號由你自己登入、自己設定;畫面上的選單名稱可能和這裡寫的略有不同(Google、Render、Neon 偶爾會改版),意思對得上就可以。每個 Part 最後都有「確認」,沒有通過就先不要往下做。

#### Part 0. 開始前準備

**1. 帳號**

| 服務 | 用來做什麼 | 建議 |
|---|---|---|
| GitHub | 放程式;網站用 GitHub Pages | 用已經設好 SSH 別名 `github.com-beethoreven` 的帳號 |
| Google | 建 Google Cloud 專案(登入與雲端硬碟);也是管理後台的第一位管理員 | 用你自己常用的 Google 帳號;不需要付款資料 |
| Neon | 資料庫 | 用 GitHub 帳號登入 |
| Render | 後端 | 用 GitHub 帳號登入(之後連 repo 比較方便) |
| cron-job.org | 每 10 分鐘叫醒後端 | 用 Email 註冊 |

**2. 準備一個安全的地方記下這些值**(例如密碼管理器)。標「秘密」的不能貼到聊天、不能進 git。

| 值 | 在哪一步產生 | 要填到哪裡 | 秘密 |
|---|---|---|---|
| 網站網址 `https://beethoreven.github.io/mama-fufood-web/` | 已知(repo 名稱決定) | Google 同意畫面的連結;來源只填 `https://beethoreven.github.io` | |
| Web Client ID(`….apps.googleusercontent.com`) | B6 | Render 的 `GOOGLE_CLIENT_ID`、網站 `js/config.js`、App `env/prod.json` | |
| 除錯金鑰的 SHA-1 | B6 | Google Cloud 的 Android 用戶端(開發用) | |
| Neon 連線字串 | C3 | Render 的 `DATABASE_URL` | **秘密** |
| `SESSION_SECRET` | D5 | Render | **秘密** |
| 後端網址(`https://….onrender.com`) | D7 | 網站 `js/config.js`、App `env/prod.json`、cron-job.org | |
| 簽署金鑰檔與它的密碼 | G1 | `android/key.properties`;**一定要備份** | **秘密** |
| 正式金鑰的 SHA-1 | G4 | Google Cloud 的 Android 用戶端(正式版) | |

**3. 本機工具**

- Flutter 與 Android Studio:照 `mama-fufood-app` README 的 Part A 裝好。
- **`keytool` 要用 Android Studio 內附的那一支。** macOS 內建的 `keytool` 沒有 Java 可以跑,會出現「Unable to locate a Java Runtime」;而且在中文環境下,`keytool -list -v` 會因為中文訊息的格式錯誤直接當掉(`IllegalFormatConversionException`),所以每次都加 `-J-Duser.language=en` 改用英文訊息。先在終端機設一個變數(只在這個終端機視窗有效,關掉要重設):

  ```bash
  export KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
  "$KEYTOOL" -J-Duser.language=en -help | head -1
  ```

  第二行印出 `Key and Certificate Management Tool` 就對了。

#### Part A. 建立 repo 與推送

四個 repo 都設 **Public**:GitHub 免費方案的 Pages 只能從公開 repo 發布。秘密(資料庫連線字串、session 金鑰、簽署金鑰)都走環境變數或 `.gitignore`,不會因為公開而外洩。規格與設計文件只留在本機,不進 git(這個 repo 的 `.gitignore` 是白名單)。

★ GitHub 的 SSH 別名是 **`github.com-beethoreven`**,不是純 `github.com`。用錯會推不上去。

1. 到 <https://github.com/new> 建立四個**空的**公開 repo:`mama-fufood`、`mama-fufood-app`、`mama-fufood-backend`、`mama-fufood-web`。Owner 選 `beethoreven`、Visibility 選 **Public**;**不要**勾選加入 README、.gitignore 或授權條款,否則第一次推送會衝突。
2. **順序不能反**:先推三個子模組,再推 meta。meta 記錄的是子模組的 commit,反過來會指向 GitHub 上還不存在的 commit。每個子模組:

   ```bash
   cd mama-fufood-backend          # 再換 mama-fufood-web、mama-fufood-app 各做一次
   git status                      # 確認只有要提交的檔案
   git add -A
   git commit -m "Initial commit"
   git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-backend.git
   git push -u origin main
   cd ..
   ```

3. 回到 meta repo,把三個子模組掛上去再推:

   ```bash
   git submodule add https://github.com/beethoreven/mama-fufood-app.git mama-fufood-app
   git submodule add https://github.com/beethoreven/mama-fufood-backend.git mama-fufood-backend
   git submodule add https://github.com/beethoreven/mama-fufood-web.git mama-fufood-web
   git status                      # 只應該有 .gitignore、.gitmodules、README.md 與三個子模組
   git add -A
   git commit -m "Initial commit"
   git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood.git
   git push -u origin main
   ```

   子模組的資料夾已經是 git repo,`git submodule add` 會直接掛上現有的 repo,不會重新下載。`.gitmodules` 用 https 網址,別人才 clone 得到;推送時用的是各子模組自己的 `origin`。

**確認**:GitHub 上四個 repo 都看得到檔案;`mama-fufood` repo 裡三個子模組顯示成「資料夾名稱 @ 一串 commit 編號」,點得進去。

#### Part B. Google Cloud(登入與雲端硬碟)

**B1. 建立專案。** 打開 <https://console.cloud.google.com/> → 上方的專案選單 → **New Project** → Project name 填 `mama-fufood`(Location 維持「No organization」)→ **Create**。建好後確認上方的專案選單顯示 `mama-fufood`,之後的步驟都在這個專案裡做。不需要設定付款帳戶。

**B2. 啟用 Google Drive API。** 左上選單 → **APIs & Services → Library** → 搜尋「Google Drive API」→ 點進去 → **Enable**。

**B3. 同意畫面的基本資料(Branding)。** 左上選單 → **Google Auth Platform**。第一次進去會有 **Get started** 的精靈:
1. App Information:App name 填 `媽媽記副食品`;User support email 選你的 Email。
2. Audience:選 **External**。
3. Contact Information:填你的 Email。
4. 勾選同意 Google API Services User Data Policy → **Create**。

建立後到 **Branding** 頁補上(下面沒提到的欄位都留空):

| 欄位 | 值 |
|---|---|
| App logo | **不要上傳**:外部、正式版的 App 一上傳 Logo 就要走 Google 的驗證流程 |
| Application home page | `https://beethoreven.github.io/mama-fufood-web/` |
| Application privacy policy link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy` |
| Application terms of service link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms` |
| Authorized domains | `beethoreven.github.io`(**不是** `github.io`;所有用到的網域都要先登記在這裡) |

按 **Save**。

**B4. 權限範圍(Data Access)。** **Data Access → Add or remove scopes**:勾選 `openid`、`.../auth/userinfo.email`、`.../auth/userinfo.profile`;再在下方「Manually add scopes」貼上這兩行 → **Add to table**:

```
https://www.googleapis.com/auth/drive.appdata
https://www.googleapis.com/auth/drive.file
```

勾選它們 → **Update** → **Save**。這五個都是 Google 分類的「非敏感」範圍,所以不需要送審,使用者登入時也不會看到「這個應用程式未經驗證」的警告。

**B5. 發布成正式版(Audience)。** **Audience → Publishing status: Testing → Publish app → Confirm**,狀態變成 **In production**。原因:Testing 狀態下,名稱、Email、頭像以外的權限(這裡是雲端硬碟)授權 7 天就失效,等於每週都要重新授權一次,而且只有列在 Test users 的帳號登得進來。

- 沒有品牌驗證時,同意畫面不會顯示 App 名稱(只顯示網域),不影響功能,不需要申請。
- 如果按下發布時 Google 要求驗證網域或送審而卡住:先不要送審,改成維持 Testing,在 **Audience → Test users** 加入會用到這個 App 的 Google 帳號。代價是雲端硬碟的授權每 7 天要在 App 裡重新同意一次(同步失敗時 App 會請你授權)。

**B6. 建立用戶端(Clients)。** **Clients → Create client**,現在先建兩個,第三個等 Part G:

1. **Web application**
   - Name:`mama-fufood-web`(只是控制台裡的標籤)。
   - Authorized JavaScript origins:`https://beethoreven.github.io`、`http://localhost:8003`(本機預覽用)。**只有網域**,不含 `/mama-fufood-web/` 路徑,**結尾不要斜線**。
   - Authorized redirect URIs:不用填。
   - **Create** 之後畫面會顯示 Client ID,**記下來**(之後在 Clients 列表也查得到)。
2. **Android**(開發時用,除錯金鑰簽的版本)
   - Package name:`tw.com.tutorarthur.mamafufood`
   - SHA-1 certificate fingerprint:用下面的指令取得,複製 `SHA1:` 後面那一串:

     ```bash
     "$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA1
     ```

     (`~/.android/debug.keystore` 是第一次 `flutter run` 時自動產生的;找不到這個檔就先跑一次 `flutter run`。)
   - **Create**。Android 用戶端的 Client ID 不用填到任何地方,Google 用套件名稱加 SHA-1 對應。

**確認**:Clients 列表有一個 Web application、一個 Android;Branding 頁的 Authorized domains 有 `beethoreven.github.io`;Audience 顯示 In production(或你選了替代做法的 Testing 加 Test users)。

#### Part C. Neon 資料庫

**C1.** 打開 <https://neon.com/>,用 GitHub 帳號登入,方案用 Free。

**C2. 建立專案:**

| 欄位 | 值 |
|---|---|
| Project name | `mama-fufood` |
| Postgres version | **18**(和本機測試用的同一版) |
| Cloud provider | AWS |
| Region | **AWS Asia Pacific 1 (Singapore)**(建好不能改,要和 Render 同區) |

其他維持預設 → **Create project**。

**C3. 取得連線字串。** 專案首頁 → **Connect** → 打開 **Connection pooling** → 複製連線字串。確認它長得像 `postgresql://…@ep-…-pooler.ap-southeast-1.aws.neon.tech/neondb?sslmode=require`:主機名稱有 **`-pooler`**、區域是 **ap-southeast-1**、最後有 **`sslmode=require`**。

**C4.** 這串含有資料庫密碼:只貼到 Render 的環境變數,不要貼到聊天、不要進 git。

**確認**:Neon 的專案列表看得到 `mama-fufood`,區域顯示 Singapore。資料表不用自己建,後端第一次啟動時會自動建。

#### Part D. Render 後端

**D1.** 打開 <https://dashboard.render.com/>,用 GitHub 帳號登入。

**D2. 建立新的 workspace。** 左上的 workspace 選單 → **New workspace** → 名稱例如 `mama-fufood`,方案選免費的。免費方案每月 750 小時是**整個 workspace 共用**的,這個專案獨立一個,才不會和其他專案搶(一個服務 24 小時開著,一個月約 744 小時)。

**D3. 建立 Web Service。** 確認左上是新的 workspace → **New → Web Service** → Git Provider 選 **GitHub**。第一次會請你授權 Render 讀 GitHub:建議選 **Only select repositories**,勾 `mama-fufood-backend` → 回到 Render,選 `mama-fufood-backend`。

**D4. 設定:**

| 欄位 | 值 |
|---|---|
| Name | `mama-fufood-backend`(網址會是 `https://mama-fufood-backend.onrender.com`;被用走的話換一個,後面所有用到後端網址的地方跟著改) |
| Language | **Python 3**(版本會讀 repo 裡的 `.python-version`,3.15.1) |
| Branch | `main` |
| Region | **Singapore** |
| Root Directory | 留空 |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `gunicorn app:app --workers 1 --threads 8 --timeout 120`(**一定要單一 worker**,原因見後端 README) |
| Instance Type | **Free** |

**D5. 環境變數**(同一頁的 **Environment Variables → Add Environment Variable**;各變數的完整說明見後端 README 的 Part D):

| 名稱 | 值 |
|---|---|
| `DATABASE_URL` | C3 的 Neon 連線字串 |
| `GOOGLE_CLIENT_ID` | B6 的 Web Client ID |
| `ALLOWED_ORIGINS` | `https://beethoreven.github.io`(**結尾不要斜線**,不含路徑) |
| `FIRST_ADMIN_EMAIL` | 你要當第一位管理員的 Google 帳號 Email |
| `SESSION_SECRET` | 一串亂數,在終端機執行 `python3 -c "import secrets; print(secrets.token_hex(32))"` 產生 |

**D6. Advanced:** Health Check Path 填 `/health`(不碰資料庫,部署時 Render 用它確認服務起來了);Auto-Deploy 維持預設(推上 `main` 就自動部署)。

**D7.** 按 **Deploy Web Service**,看畫面上的部署記錄,大約 2~5 分鐘。部署成功後,服務頁面上方會顯示網址,**記下來**。

**確認:**
- 瀏覽器打開 `https://mama-fufood-backend.onrender.com/health`(換成你的網址),看到 `"ok": true`。
- 打開 `/ready`,看到 `"ok": true`,`"cache"` 裡有食材清單與單位的版本(`20261023000000`)。第一次可能要等半分鐘左右(資料庫在喚醒、後端在建表),回 503 就等一下再重新整理。第一次啟動時,後端會自動建表、放入第一版的食材清單(11 組、105 種)與單位。

**常見問題:**
- 部署記錄說找不到 Python 3.15.1:Render 還沒有這一版。把 `.python-version` 改成 Render 有的最新版、在旁邊寫下原因,推上去重新部署。
- `/ready` 一直是 503、`database_last_ok` 一直是 `null`:`DATABASE_URL` 貼錯(多了空白、少了 `sslmode=require`),或 Neon 專案被暫停。

#### Part E. cron-job.org(保活)

Render 免費方案閒置 15 分鐘會休眠;每 10 分鐘打一次 `/health`,就不會睡著。`/health` 不碰資料庫,Neon 照樣可以休眠,免費方案的時數不會被保活用掉。

**E1.** 打開 <https://cron-job.org/>,註冊並登入。

**E2. Create cronjob:**

| 欄位 | 值 |
|---|---|
| Title | `mama-fufood 保活` |
| URL | `https://mama-fufood-backend.onrender.com/health`(你的後端網址加 `/health`) |
| Execution schedule | 每 10 分鐘(Every 10 minutes) |
| Notifications | 打開「執行失敗時通知」,寄到你的信箱 |

**Create** 存檔。

**E3.** 在工作的詳情頁手動執行一次(Test run),展開執行記錄,狀態要是 **200**。

★ 姊妹專案 boo-king-king 實測:Render 冷啟動要 **32.4 秒**,而 cron-job.org 免費方案**等 30 秒就斷線**,並且**連續失敗超過 25 次會自動停用那個工作**。所以服務剛好睡著時那一次可能記成失敗(但已經把它叫醒,下一次就成功);只要通知信沒有一直來就沒問題。如果連續收到失敗通知,先打開 `/health` 確認後端活著,再到 cron-job.org 確認工作沒有被停用。

**確認**:執行記錄連續幾次都是 200。

#### Part F. 網站(GitHub Pages 與管理後台)

**F1.** 改 `mama-fufood-web/js/config.js`:

```js
apiBase: "https://mama-fufood-backend.onrender.com",   // D7 的後端網址,結尾不要斜線
googleClientId: "B6 的 Web Client ID",
```

然後 commit、push。

**F2.** `mama-fufood-web` repo → **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main`、資料夾 `/ (root)` → Save**。一兩分鐘後,同一頁上方會顯示網站的網址。

**F3. 確認:**
1. 打開 `https://beethoreven.github.io/mama-fufood-web/` 看得到首頁;條款、政策、判斷邏輯說明三個按鈕都打得開。
2. 右上角用 `FIRST_ADMIN_EMAIL` 的帳號登入(後端剛喚醒時最多等一分鐘)→ 出現「管理後台」。
3. 進後台:看得到 11 個分組、105 種食材;「系統」分頁的快取版本與資料庫版本一致。

**F4.** 要讓其他人也當管理員:後台「管理員」分頁新增他們的 Email。

**常見問題:**
- 首頁沒有登入按鈕:`googleClientId` 是空的,或瀏覽器的擋廣告外掛擋掉了 Google 的登入元件(頁面會顯示原因)。
- Google 的登入視窗顯示來源不符(origin mismatch):B6 的 JavaScript 來源少了 `https://beethoreven.github.io`,或多了路徑、結尾斜線。
- 登入後顯示「連不上後端」:後端在喚醒,等一分鐘再試;一直不行就檢查 `apiBase`。
- 瀏覽器的開發者工具出現 CORS 錯誤:Render 的 `ALLOWED_ORIGINS` 不是剛好 `https://beethoreven.github.io`。

#### Part G. App 的簽署金鑰(學一次就好)

**這把金鑰非常重要:裝到手機之後,之後的每一版都要用同一把簽,否則只能先解除安裝(手機上的資料會不見)。** 金鑰檔和密碼都要備份(例如放進密碼管理器);不能進 git。

**G1. 建立金鑰:**

```bash
mkdir -p ~/Keys/mama-fufood
"$KEYTOOL" -J-Duser.language=en -genkeypair -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

指令會一題一題問:

| 問題 | 怎麼填 |
|---|---|
| Enter keystore password / Re-enter new password | 自己訂一組密碼(至少 6 個字),輸入時畫面不會顯示 |
| What is your first and last name? 以及 organizational unit、organization、City、State、country code | ★ 這些會寫進 App 安裝檔的憑證裡,**任何人拿到 APK 都讀得到**。不想公開就直接按 Enter 留空 |
| Is CN=…, correct? | 輸入 `yes` |

**G2.** 在 `mama-fufood-app/android/` 建立 `key.properties`(已經在 `.gitignore` 裡)。`storeFile` 要寫完整路徑,`~` 不會被展開:

```properties
storePassword=你剛剛訂的密碼
keyPassword=你剛剛訂的密碼
keyAlias=upload
storeFile=/Users/<你的帳號>/Keys/mama-fufood/upload-keystore.jks
```

(用這個指令建立的金鑰,金鑰密碼和金鑰庫密碼是同一組。)

**G3.** 備份 `upload-keystore.jks` 與密碼到電腦以外的地方。

**G4.** 取得這把金鑰的 SHA-1,回到 Google Cloud 的 **Clients → Create client → Android**,建立第三個用戶端(Package name 一樣是 `tw.com.tutorarthur.mamafufood`):

```bash
"$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -alias upload | grep SHA1
```

**確認**:Google Cloud 的 Clients 列表有兩個 Android 用戶端,SHA-1 不一樣。

#### Part H. 打包與安裝到手機

**H1.** 在 `mama-fufood-app/env/` 建立 `prod.json`(已經在 `.gitignore` 裡),照 `prod.example.json` 填:

```json
{
  "MFF_API_BASE": "https://mama-fufood-backend.onrender.com",
  "MFF_WEB_CLIENT_ID": "B6 的 Web Client ID"
}
```

**H2. 打包正式版:**

```bash
cd mama-fufood-app
flutter build apk --release --dart-define-from-file=env/prod.json
```

產出 `build/app/outputs/flutter-apk/app-release.apk`(約 76 MB)。沒有 `key.properties` 時會直接停下來並說明原因(避免不小心用除錯金鑰簽)。

**H3. 手機打開 USB 偵錯**(Samsung 的路徑;其他廠牌名稱略有不同):**設定 → 關於手機 → 軟體資訊 → 連點「版本號碼」7 次**(會要求輸入手機的解鎖密碼)→ 回到設定,最下面多了「**開發人員選項**」→ 打開「**USB 偵錯**」。

**H4. 安裝:** 用 USB 線接上電腦,手機跳出「允許 USB 偵錯嗎?」按允許(可以勾「一律允許這台電腦」),然後:

```bash
adb devices
adb install -r build/app/outputs/flutter-apk/app-release.apk
```

`adb devices` 要列出手機,狀態是 `device`;如果是 `unauthorized`,看手機上有沒有跳出允許的視窗。安裝成功會顯示 `Success`。

**H5. 之後更新:** 改版號(x.y.z 的規則見 App README)→ 重新打包 → 同一行 `adb install -r`,手機上的資料會保留。

**常見問題:**
- 安裝時出現 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`:這支手機上已經有一個用**別的金鑰**簽的同名 App(例如之前裝過除錯版)。只能先解除安裝舊的(那支 App 的資料會不見)再裝。
- App 裡用 Google 登入時,出現「Developer console is not set up correctly」之類的錯誤:Google Cloud 裡沒有和這個安裝檔相符的 Android 用戶端,檢查套件名稱與 SHA-1(正式版要用 G4 的 SHA-1)。

#### Part I. 把各邊接起來(檢查清單)

**少任何一項,登入或連線都會失敗:**

1. Google 同意畫面:Authorized domains 有 `beethoreven.github.io`;權限範圍有 `drive.appdata` 與 `drive.file`;狀態是 In production(或 Testing 加 Test users)。
2. Google「網頁應用程式」用戶端的「已授權的 JavaScript 來源」有 `https://beethoreven.github.io`。
3. Google 有兩個 Android 用戶端:除錯金鑰與正式金鑰各一個 SHA-1,套件名稱都是 `tw.com.tutorarthur.mamafufood`。
4. Render 的 `ALLOWED_ORIGINS` = `https://beethoreven.github.io`,`GOOGLE_CLIENT_ID` = Web Client ID。
5. 網站 `js/config.js`:`apiBase` = 後端網址、`googleClientId` = Web Client ID。
6. App `env/prod.json`:`MFF_API_BASE` = 後端網址、`MFF_WEB_CLIENT_ID` = Web Client ID;改了要重新打包。
7. cron-job.org 每 10 分鐘打 `/health`,失敗通知是開著的。

#### Part J. 驗收

| 步驟 | 預期 |
|---|---|
| 第一次打開 | 深色的啟動畫面(只有圖示)→ 資料運用原則,捲到最下面才能按「我了解了」→ 輸入第一個寶寶名稱(關不掉) |
| 月曆點一天 → 新增紀錄 → 儲存 | 月曆那一格出現「emoji + 食材名稱」 |
| 右上角 Google 登入 | 系統的帳號選擇畫面,登入後換成頭像 |
| App 設定 → 檢查更新 | 「公用資料已是最新」,今天還有 9 次 |
| App 設定 → 打開「紀錄與雲端同步」 | 跳出 Google 的雲端硬碟授權;之後「最後同步」有時間 |
| 打開「照片存到雲端」,新增一筆有照片的紀錄 | 雲端硬碟出現「媽媽記副食品/照片」資料夾與照片 |
| 匯入/匯出 → 匯出 | 「匯出完成」,雲端硬碟「媽媽記副食品」資料夾有 `.mff` 檔 |
| 意見回饋 → 送出 | 管理後台「意見回饋」看得到,Email 是你的帳號 |
| 第二支手機登入同一個帳號、打開同步 | 兩支手機的紀錄會合併(同一筆兩邊都改才會問你保留哪一邊) |

有任何一步不對,在 App 的「意見回饋」選「遇到問題」送出。

### 已知限制

- **這份上線 SOP 還沒有照著完整跑過一次**;實際做的時候遇到和這裡不一樣的畫面或錯誤,要回來修正這份文件。Google 同意畫面在發布時會不會要求驗證網域,事先無法確認,所以 B5 寫了替代做法。
- 還沒有上架到 Google Play:上架需要的商店素材、封閉測試、Data safety 等都還沒做。
- Android 的開發者驗證:Google 從 2026-09-30 起在部分國家要求 App 的開發者向 Google 登記才能安裝,2027 年起擴大到全球;用 `adb` 安裝仍然可以。台灣的實施時間要再確認。
- Neon 免費方案的額度:方案頁有「每月 100 CU-hours」與「50」兩種寫法,建立前再確認一次。

---

## English

The project-overview repo for *媽媽記副食品* ("Mama's Baby Food Log"), an app for logging a baby's complementary food, wiring its three subsystems together via git submodules so the whole codebase can be pulled with a single `clone`. This repo itself **is never deployed** — each subsystem connects to its own repo for deployment or builds (see the table below). For technical details and architecture decisions, see each subsystem's own README.

Records live only on the phone and in the user's own Google Drive; the backend only handles public data (the food list), the daily check and feedback, and never touches any of the baby's data. This app is for record-keeping and reference only, not a judgement based on medical knowledge.

### Subsystems

| Subsystem | Repo | Deployment / output |
|---|---|---|
| Android app | [`mama-fufood-app`](https://github.com/beethoreven/mama-fufood-app) | Built locally into an APK and installed with `adb` (not on Google Play yet) |
| Backend API | [`mama-fufood-backend`](https://github.com/beethoreven/mama-fufood-backend) | Render (planned `https://mama-fufood-backend.onrender.com`) |
| Website and admin console | [`mama-fufood-web`](https://github.com/beethoreven/mama-fufood-web) | GitHub Pages (`https://beethoreven.github.io/mama-fufood-web/`) |

The database is Neon Postgres 18 in **ap-southeast-1 (Singapore)**, the same region as Render. This is a prerequisite, not a performance tweak: in the sister project zh-cn-to-tw, a single query took 1.76 s across regions and 0.03 s within the same region — 59 times slower. A Neon project's region cannot be changed after creation.

### Usage

**First time cloning everything:**

```bash
git clone --recurse-submodules git@github.com-beethoreven:beethoreven/mama-fufood.git
```

(`github.com-beethoreven` is an SSH alias on the developer's own machine; everyone else uses `https://github.com/beethoreven/mama-fufood.git`.)

**Already cloned, just need to fetch the submodules:**

```bash
git submodule update --init --recursive
```

**Updating a submodule to its own repo's latest version:**

```bash
cd mama-fufood-backend   # or mama-fufood-app, mama-fufood-web
git pull origin main
cd ..
git add mama-fufood-backend
git commit -m "Update mama-fufood-backend submodule pointer"
```

That final `git add` + `commit` cannot be skipped — without it, the submodule's local files are up to date, but this wrapper repo's recorded "which commit to point at" is still stale, so anyone else who `clone`s this repo will still get the old version.

---

### Setup Guide

★ The go-live steps deliberately live here rather than in each subsystem's README, because they **span all three subsystems plus Google Cloud**, with cross-references that nobody could follow end to end once split apart:

1. The Google Cloud "Web application" client ID → the backend's `GOOGLE_CLIENT_ID`, the website's `googleClientId` in `js/config.js`, and the app's `MFF_WEB_CLIENT_ID` in `env/prod.json` (the same value in all three places).
2. The website's URL → the backend's `ALLOWED_ORIGINS`, the Google client's "Authorized JavaScript origins", and the Google consent screen's "Authorized domains" and links.
3. The backend's URL → the website's `apiBase` in `js/config.js`, the app's `MFF_API_BASE` in `env/prod.json`, and the keep-alive URL on cron-job.org.
4. The SHA-1 of the app's signing key → the Android client in Google Cloud.

Follow Part 0 and Parts A–J in order. You sign in to and configure every external service yourself; menu names on screen may differ slightly from what's written here (Google, Render and Neon redesign occasionally) — as long as the meaning matches, you're fine. Every Part ends with a "Check"; don't move on until it passes.

#### Part 0. Before You Start

**1. Accounts**

| Service | What it's for | Suggestion |
|---|---|---|
| GitHub | Hosting the code; the website uses GitHub Pages | The account already set up with the SSH alias `github.com-beethoreven` |
| Google | The Google Cloud project (sign-in and Drive); also the first admin of the admin console | Your own everyday Google account; no billing details needed |
| Neon | Database | Sign in with GitHub |
| Render | Backend | Sign in with GitHub (makes connecting repos easier later) |
| cron-job.org | Waking the backend every 10 minutes | Sign up with email |

**2. Prepare a safe place to record these values** (e.g. a password manager). Those marked "secret" must never be pasted into a chat or committed to git.

| Value | Created in step | Goes into | Secret |
|---|---|---|---|
| Website URL `https://beethoreven.github.io/mama-fufood-web/` | Known in advance (set by the repo name) | Google consent screen links; origins only take `https://beethoreven.github.io` | |
| Web Client ID (`….apps.googleusercontent.com`) | B6 | Render's `GOOGLE_CLIENT_ID`, the website's `js/config.js`, the app's `env/prod.json` | |
| SHA-1 of the debug key | B6 | The Google Cloud Android client (development) | |
| Neon connection string | C3 | Render's `DATABASE_URL` | **Secret** |
| `SESSION_SECRET` | D5 | Render | **Secret** |
| Backend URL (`https://….onrender.com`) | D7 | The website's `js/config.js`, the app's `env/prod.json`, cron-job.org | |
| The signing keystore file and its password | G1 | `android/key.properties`; **must be backed up** | **Secret** |
| SHA-1 of the release key | G4 | The Google Cloud Android client (release) | |

**3. Local tools**

- Flutter and Android Studio: install them per Part A of the `mama-fufood-app` README.
- **Use the `keytool` bundled with Android Studio.** macOS's built-in `keytool` has no Java to run on and fails with "Unable to locate a Java Runtime"; and in a Chinese-language environment, `keytool -list -v` crashes on a formatting bug in its Chinese messages (`IllegalFormatConversionException`), so always add `-J-Duser.language=en` to switch it to English messages. Set a variable in the terminal first (it only lasts for this terminal window; set it again after closing it):

  ```bash
  export KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
  "$KEYTOOL" -J-Duser.language=en -help | head -1
  ```

  If the second line prints `Key and Certificate Management Tool`, you're set.

#### Part A. Create the Repos and Push

All four repos are **Public**: on GitHub's free plan, Pages can only publish from a public repo. Secrets (the database connection string, the session key, the signing key) all go through environment variables or `.gitignore`, so nothing leaks just because the repos are public. Spec and design documents stay on the local machine only and never enter git (this repo's `.gitignore` is an allowlist).

★ The GitHub SSH alias is **`github.com-beethoreven`**, not plain `github.com`. Using the wrong one means the push fails.

1. Go to <https://github.com/new> and create four **empty** public repos: `mama-fufood`, `mama-fufood-app`, `mama-fufood-backend`, `mama-fufood-web`. Choose owner `beethoreven` and visibility **Public**; do **not** tick adding a README, .gitignore or license, or the first push will conflict.
2. **The order can't be reversed**: push the three submodules first, then the meta repo. The meta repo records the submodules' commits; the other way round it would point at commits that don't exist on GitHub yet. For each submodule:

   ```bash
   cd mama-fufood-backend          # then repeat for mama-fufood-web and mama-fufood-app
   git status                      # confirm only the files you mean to commit are there
   git add -A
   git commit -m "Initial commit"
   git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-backend.git
   git push -u origin main
   cd ..
   ```

3. Back in the meta repo, attach the three submodules and push:

   ```bash
   git submodule add https://github.com/beethoreven/mama-fufood-app.git mama-fufood-app
   git submodule add https://github.com/beethoreven/mama-fufood-backend.git mama-fufood-backend
   git submodule add https://github.com/beethoreven/mama-fufood-web.git mama-fufood-web
   git status                      # should only show .gitignore, .gitmodules, README.md and the three submodules
   git add -A
   git commit -m "Initial commit"
   git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood.git
   git push -u origin main
   ```

   The submodule folders are already git repos, so `git submodule add` attaches the existing repos instead of downloading them again. `.gitmodules` uses https URLs so others can clone; pushing uses each submodule's own `origin`.

**Check**: all four repos show files on GitHub; in the `mama-fufood` repo the three submodules appear as "folder name @ a commit hash" and can be clicked through.

#### Part B. Google Cloud (Sign-In and Drive)

**B1. Create the project.** Open <https://console.cloud.google.com/> → the project picker at the top → **New Project** → Project name `mama-fufood` (leave Location as "No organization") → **Create**. Once created, make sure the project picker at the top shows `mama-fufood`; every following step happens inside this project. No billing account is needed.

**B2. Enable the Google Drive API.** Top-left menu → **APIs & Services → Library** → search "Google Drive API" → open it → **Enable**.

**B3. Consent screen basics (Branding).** Top-left menu → **Google Auth Platform**. The first visit shows a **Get started** wizard:
1. App Information: App name `媽媽記副食品`; User support email: your email.
2. Audience: **External**.
3. Contact Information: your email.
4. Tick agreement to the Google API Services User Data Policy → **Create**.

Afterwards, fill in the following on the **Branding** page (leave every field not mentioned here empty):

| Field | Value |
|---|---|
| App logo | **Don't upload one**: for an external app in production, uploading a logo triggers Google's verification process |
| Application home page | `https://beethoreven.github.io/mama-fufood-web/` |
| Application privacy policy link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy` |
| Application terms of service link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms` |
| Authorized domains | `beethoreven.github.io` (**not** `github.io`; every domain in use must be registered here first) |

Press **Save**.

**B4. Scopes (Data Access).** **Data Access → Add or remove scopes**: tick `openid`, `.../auth/userinfo.email` and `.../auth/userinfo.profile`; then paste these two lines into "Manually add scopes" at the bottom → **Add to table**:

```
https://www.googleapis.com/auth/drive.appdata
https://www.googleapis.com/auth/drive.file
```

Tick them → **Update** → **Save**. All five are classified by Google as "non-sensitive", so no review is needed and users won't see the "this app isn't verified" warning when signing in.

**B5. Publish to production (Audience).** **Audience → Publishing status: Testing → Publish app → Confirm**; the status becomes **In production**. Why: in Testing status, grants for anything beyond name, email and profile picture (here: Drive) expire after 7 days — re-authorizing every week — and only accounts listed under Test users can sign in.

- Without brand verification the consent screen doesn't show the app name (only the domain); this doesn't affect functionality and doesn't need to be applied for.
- If publishing gets stuck because Google asks for domain verification or a review: don't submit anything; stay in Testing instead and add the Google accounts that will use the app under **Audience → Test users**. The cost is that the Drive grant has to be re-approved in the app every 7 days (the app asks for it when sync fails).

**B6. Create the clients (Clients).** **Clients → Create client**; create two now and the third in Part G:

1. **Web application**
   - Name: `mama-fufood-web` (just a label in the console).
   - Authorized JavaScript origins: `https://beethoreven.github.io`, `http://localhost:8003` (for local preview). **Domain only**, without the `/mama-fufood-web/` path, **no trailing slash**.
   - Authorized redirect URIs: leave empty.
   - After **Create**, the Client ID is shown — **write it down** (it can also be found in the Clients list later).
2. **Android** (for development, builds signed with the debug key)
   - Package name: `tw.com.tutorarthur.mamafufood`
   - SHA-1 certificate fingerprint: get it with the command below and copy the string after `SHA1:`:

     ```bash
     "$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA1
     ```

     (`~/.android/debug.keystore` is created automatically by the first `flutter run`; if the file doesn't exist, run `flutter run` once first.)
   - **Create**. The Android client's ID isn't entered anywhere — Google matches it by package name plus SHA-1.

**Check**: the Clients list has one Web application and one Android client; the Branding page's Authorized domains include `beethoreven.github.io`; Audience shows In production (or Testing plus Test users if you took the fallback).

#### Part C. Neon Database

**C1.** Open <https://neon.com/> and sign in with GitHub; use the Free plan.

**C2. Create the project:**

| Field | Value |
|---|---|
| Project name | `mama-fufood` |
| Postgres version | **18** (the same version used for local tests) |
| Cloud provider | AWS |
| Region | **AWS Asia Pacific 1 (Singapore)** (can't be changed later; must match Render) |

Keep everything else at the defaults → **Create project**.

**C3. Get the connection string.** Project home → **Connect** → turn on **Connection pooling** → copy the connection string. Make sure it looks like `postgresql://…@ep-…-pooler.ap-southeast-1.aws.neon.tech/neondb?sslmode=require`: the host contains **`-pooler`**, the region is **ap-southeast-1**, and it ends with **`sslmode=require`**.

**C4.** This string contains the database password: paste it only into Render's environment variables — never into a chat, never into git.

**Check**: the `mama-fufood` project appears in Neon's project list with the Singapore region. No need to create tables yourself — the backend creates them on its first start.

#### Part D. Render Backend

**D1.** Open <https://dashboard.render.com/> and sign in with GitHub.

**D2. Create a new workspace.** The workspace menu at the top left → **New workspace** → name it e.g. `mama-fufood`, on the free plan. The free 750 hours per month are **shared across the whole workspace**, so this project gets its own to avoid competing with other projects (one service running around the clock uses about 744 hours a month).

**D3. Create the Web Service.** Make sure the new workspace is selected at the top left → **New → Web Service** → Git Provider **GitHub**. The first time, Render asks for access to GitHub: preferably choose **Only select repositories** and tick `mama-fufood-backend` → back in Render, pick `mama-fufood-backend`.

**D4. Settings:**

| Field | Value |
|---|---|
| Name | `mama-fufood-backend` (the URL becomes `https://mama-fufood-backend.onrender.com`; if the name is taken, pick another and update every place that uses the backend URL) |
| Language | **Python 3** (the version is read from the repo's `.python-version`, 3.15.1) |
| Branch | `main` |
| Region | **Singapore** |
| Root Directory | Leave empty |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `gunicorn app:app --workers 1 --threads 8 --timeout 120` (**must be a single worker** — see the backend README for why) |
| Instance Type | **Free** |

**D5. Environment variables** (on the same page, **Environment Variables → Add Environment Variable**; full descriptions of each variable are in Part D of the backend README):

| Name | Value |
|---|---|
| `DATABASE_URL` | The Neon connection string from C3 |
| `GOOGLE_CLIENT_ID` | The Web Client ID from B6 |
| `ALLOWED_ORIGINS` | `https://beethoreven.github.io` (**no trailing slash**, no path) |
| `FIRST_ADMIN_EMAIL` | The Google account email of the first admin |
| `SESSION_SECRET` | A random string: generate it in the terminal with `python3 -c "import secrets; print(secrets.token_hex(32))"` |

**D6. Advanced:** set Health Check Path to `/health` (it doesn't touch the database; Render uses it to confirm the service came up during deploys); leave Auto-Deploy at its default (pushing to `main` deploys automatically).

**D7.** Press **Deploy Web Service** and watch the deploy log on screen — about 2–5 minutes. Once the deploy succeeds, the URL is shown at the top of the service page — **write it down**.

**Check:**
- Open `https://mama-fufood-backend.onrender.com/health` (with your URL) in a browser and see `"ok": true`.
- Open `/ready` and see `"ok": true`, with the food list and unit versions (`20261023000000`) under `"cache"`. The first time may take around half a minute (the database is waking up and the backend is creating tables); if it returns 503, wait a moment and refresh. On first start the backend automatically creates the tables and seeds the first food list (11 groups, 105 foods) and units.

**Common problems:**
- The deploy log says Python 3.15.1 can't be found: Render doesn't offer that version yet. Change `.python-version` to the newest version Render has, note the reason next to it, and push to redeploy.
- `/ready` stays at 503 and `database_last_ok` stays `null`: `DATABASE_URL` was pasted wrong (extra spaces, missing `sslmode=require`), or the Neon project is suspended.

#### Part E. cron-job.org (Keep-Alive)

Render's free tier sleeps after 15 minutes idle; hitting `/health` every 10 minutes keeps it awake. `/health` doesn't touch the database, so Neon can still sleep and the free plan's hours aren't used up by the keep-alive.

**E1.** Open <https://cron-job.org/>, sign up and sign in.

**E2. Create cronjob:**

| Field | Value |
|---|---|
| Title | `mama-fufood keep-alive` |
| URL | `https://mama-fufood-backend.onrender.com/health` (your backend URL plus `/health`) |
| Execution schedule | Every 10 minutes |
| Notifications | Turn on "notify on failed execution", sent to your email |

**Create** to save.

**E3.** Run it once manually from the job's detail page (Test run) and expand the run history: the status must be **200**.

★ Measured in the sister project boo-king-king: a Render cold start takes **32.4 seconds**, while cron-job.org's free plan **closes the connection after 30 seconds**, and **auto-disables a job after more than 25 consecutive failures**. So the one run that hits a sleeping service may be recorded as a failure (but it has already woken the service, so the next run succeeds); as long as failure emails don't keep coming, all is well. If you do get repeated failure emails, first open `/health` to confirm the backend is alive, then check on cron-job.org that the job hasn't been disabled.

**Check**: several consecutive runs in the history show 200.

#### Part F. Website (GitHub Pages and the Admin Console)

**F1.** Edit `mama-fufood-web/js/config.js`:

```js
apiBase: "https://mama-fufood-backend.onrender.com",   // the backend URL from D7, no trailing slash
googleClientId: "the Web Client ID from B6",
```

Then commit and push.

**F2.** In the `mama-fufood-web` repo: **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main`, folder `/ (root)` → Save**. A minute or two later, the site URL appears at the top of the same page.

**F3. Check:**
1. `https://beethoreven.github.io/mama-fufood-web/` shows the home page; the terms, privacy policy and judgement-logic guide buttons all open.
2. Sign in at the top right with the `FIRST_ADMIN_EMAIL` account (allow up to a minute if the backend was asleep) → "管理後台" (Admin console) appears.
3. In the console: 11 groups and 105 foods are listed; the "系統" (System) tab shows matching cache and database versions.

**F4.** To make someone else an admin too: add their email on the console's "管理員" (Admins) tab.

**Common problems:**
- No sign-in button on the home page: `googleClientId` is empty, or an ad-blocking extension blocked Google's sign-in script (the page says so).
- Google's sign-in window reports an origin mismatch: B6's JavaScript origins are missing `https://beethoreven.github.io`, or have a path or trailing slash added.
- "連不上後端" (can't reach the backend) after signing in: the backend is waking up — wait a minute and retry; if it never works, check `apiBase`.
- A CORS error in the browser's developer tools: Render's `ALLOWED_ORIGINS` isn't exactly `https://beethoreven.github.io`.

#### Part G. The App's Signing Key (Learn It Once)

**This key is critical: once installed on a phone, every later version must be signed with the same key, or the only way forward is uninstalling (which deletes the phone's data).** Back up both the key file and the password (e.g. in a password manager); they must never enter git.

**G1. Create the key:**

```bash
mkdir -p ~/Keys/mama-fufood
"$KEYTOOL" -J-Duser.language=en -genkeypair -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

The command asks one question at a time:

| Question | What to enter |
|---|---|
| Enter keystore password / Re-enter new password | A password of your choosing (at least 6 characters); nothing shows on screen while typing |
| What is your first and last name? plus organizational unit, organization, City, State, country code | ★ These are written into the certificate inside the app's APK, so **anyone holding the APK can read them**. To keep them private, just press Enter to leave them empty |
| Is CN=…, correct? | Type `yes` |

**G2.** Create `key.properties` in `mama-fufood-app/android/` (already in `.gitignore`). `storeFile` must be a full path — `~` is not expanded:

```properties
storePassword=the password you just chose
keyPassword=the password you just chose
keyAlias=upload
storeFile=/Users/<your account>/Keys/mama-fufood/upload-keystore.jks
```

(With a key created by this command, the key password and the keystore password are the same.)

**G3.** Back up `upload-keystore.jks` and the password somewhere other than this computer.

**G4.** Get this key's SHA-1 and go back to Google Cloud: **Clients → Create client → Android** to create the third client (the package name is again `tw.com.tutorarthur.mamafufood`):

```bash
"$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -alias upload | grep SHA1
```

**Check**: Google Cloud's Clients list has two Android clients with different SHA-1s.

#### Part H. Build and Install on a Phone

**H1.** Create `prod.json` in `mama-fufood-app/env/` (already in `.gitignore`), following `prod.example.json`:

```json
{
  "MFF_API_BASE": "https://mama-fufood-backend.onrender.com",
  "MFF_WEB_CLIENT_ID": "the Web Client ID from B6"
}
```

**H2. Build the release:**

```bash
cd mama-fufood-app
flutter build apk --release --dart-define-from-file=env/prod.json
```

The output is `build/app/outputs/flutter-apk/app-release.apk` (about 76 MB). Without `key.properties` the build stops and explains why (so it can't accidentally be signed with the debug key).

**H3. Turn on USB debugging on the phone** (Samsung's path; other brands name things slightly differently): **Settings → About phone → Software information → tap "Build number" 7 times** (it asks for the phone's unlock code) → back in Settings, "**Developer options**" now appears at the bottom → turn on "**USB debugging**".

**H4. Install:** connect the phone with a USB cable, tap Allow when the phone asks "Allow USB debugging?" (you can tick "Always allow from this computer"), then:

```bash
adb devices
adb install -r build/app/outputs/flutter-apk/app-release.apk
```

`adb devices` must list the phone with the status `device`; if it says `unauthorized`, check the phone for the allow prompt. A successful install prints `Success`.

**H5. Updating later:** bump the version (the x.y.z rules are in the app README) → rebuild → the same `adb install -r`; the data on the phone is kept.

**Common problems:**
- `INSTALL_FAILED_UPDATE_INCOMPATIBLE` during install: the phone already has an app with the same package name signed with **a different key** (e.g. a debug build installed earlier). The only way is to uninstall the old one first (that app's data is lost) and then install.
- Signing in with Google inside the app shows something like "Developer console is not set up correctly": Google Cloud has no Android client matching this APK — check the package name and SHA-1 (the release build needs the SHA-1 from G4).

#### Part I. Wire Everything Together (Checklist)

**Miss any one of these and sign-in or connections will fail:**

1. Google consent screen: Authorized domains include `beethoreven.github.io`; the scopes include `drive.appdata` and `drive.file`; the status is In production (or Testing plus Test users).
2. The Google "Web application" client's "Authorized JavaScript origins" include `https://beethoreven.github.io`.
3. Google has two Android clients: one SHA-1 each for the debug key and the release key, both with package name `tw.com.tutorarthur.mamafufood`.
4. Render's `ALLOWED_ORIGINS` = `https://beethoreven.github.io`, `GOOGLE_CLIENT_ID` = the Web Client ID.
5. The website's `js/config.js`: `apiBase` = the backend URL, `googleClientId` = the Web Client ID.
6. The app's `env/prod.json`: `MFF_API_BASE` = the backend URL, `MFF_WEB_CLIENT_ID` = the Web Client ID; rebuild after any change.
7. cron-job.org hits `/health` every 10 minutes, with failure notifications on.

#### Part J. Acceptance

| Step | Expected |
|---|---|
| First launch | Dark splash screen (icon only) → data-use principles, where "我了解了" (I understand) only becomes tappable after scrolling to the bottom → enter the first baby's name (can't be dismissed) |
| Tap a day on the calendar → add a record → save | That calendar cell shows "emoji + food name" |
| Google sign-in at the top right | The system's account picker; after signing in it turns into your avatar |
| App settings → check for updates | "公用資料已是最新" (public data is up to date), 9 checks left today |
| App settings → turn on "紀錄與雲端同步" (record cloud sync) | Google's Drive authorization pops up; afterwards "最後同步" (last sync) shows a time |
| Turn on "照片存到雲端" (save photos to the cloud) and add a record with a photo | The "媽媽記副食品/照片" folder and the photo appear in Drive |
| Import/Export → Export | "匯出完成" (export complete), and a `.mff` file appears in the Drive "媽媽記副食品" folder |
| Feedback → send | It shows up under "意見回饋" (Feedback) in the admin console, with your account's email |
| A second phone signs in to the same account and turns on sync | The two phones' records merge (you're only asked which side to keep when both changed the same record) |

If any step doesn't match, send feedback from the app with "遇到問題" (ran into a problem).

### Known Limitations

- **This Setup Guide has not been run end to end yet**; if the real screens or errors differ from what's written here, come back and fix this document. Whether Google's consent screen asks for domain verification when publishing can't be confirmed in advance, which is why B5 includes a fallback.
- Not listed on Google Play yet: the store assets, closed testing, Data safety form and so on needed for listing haven't been done.
- Android developer verification: from 2026-09-30 Google requires app developers to register with Google before their apps can be installed in some countries, expanding worldwide from 2027; installing with `adb` still works. When it takes effect in Taiwan still needs checking.
- Neon free-plan quota: the plans page says both "100 CU-hours per month" and "50" in different places — double-check before creating the project.
