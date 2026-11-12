# mama-fufood

## 中文

「媽媽記副食品」寶寶副食品紀錄 App 的專案總覽 repo,透過 git submodule 把三個子系統掛在一起,方便一次 `clone` 齊全部程式碼。這個 repo 本身**不會被部署**——各子系統各自連到自己的 repo 做部署或打包(見下表)。各子系統的技術細節與架構決策,請見各自 repo 的 README。

紀錄只存在手機與使用者自己的 Google 雲端硬碟;後端只管公用資料(食材清單)、每日檢查與意見回饋,不碰任何寶寶的資料。本 App 僅供記錄與參考,非基於醫學知識做的判斷。

### 子系統

| 子系統 | Repo | 部署/產出 |
|---|---|---|
| Android App | [`mama-fufood-app`](https://github.com/beethoreven/mama-fufood-app) | 本機打包成 APK,用 `adb` 安裝到手機(還沒有上架 Google Play) |
| 後端 API | [`router-backend`](https://github.com/beethoreven/router-backend) 的 `apps/mama_fufood/`(好幾個專案共用的後端) | Render(`https://router-backend-3hl2.onrender.com/mama-fufood`) |
| 網站與管理後台 | [`mama-fufood-web`](https://github.com/beethoreven/mama-fufood-web) | GitHub Pages(`https://beethoreven.github.io/mama-fufood-web/`) |

資料庫是 Neon Postgres 18,開在 **ap-southeast-1(Singapore)**,與 Render 同區。這不是效能微調而是前提:實測兩端跨區時單次查詢 1.76 秒,同區 0.03 秒,差 59 倍。Neon 的區域建好之後不能改。

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
cd mama-fufood-app   # 或 mama-fufood-web、router-backend
git pull origin main
cd ..
git add mama-fufood-app
git commit -m "Update mama-fufood-app submodule pointer"
```

最後這個 `git add` + `commit` 不能省略——沒做的話,子模組本機檔案雖然是新的,但這個 wrapper repo 記錄的「該指向哪個 commit」還是舊的,別人 `clone` 這個 repo 還是會抓到舊版本。

---

### 上線 SOP

★ 上線步驟刻意放在這裡而不是各子系統的 README,因為它**跨越三個子系統與 Google Cloud**,有幾條交叉引用,拆開之後就沒有人能照著做完:

1. Google Cloud「網頁應用程式」用戶端 ID → router-backend 的 `MAMA_FUFOOD__GOOGLE_CLIENT_ID`、網站 `js/config.js` 的 `googleClientId`、App `env/prod.json` 的 `MFF_WEB_CLIENT_ID`(三個地方是同一個值)。
2. 網站的網址 → router-backend 的 `MAMA_FUFOOD__ALLOWED_ORIGINS`、Google 用戶端的「已授權的 JavaScript 來源」、Google 同意畫面的「已授權網域」與連結。
3. 後端的網址(`https://router-backend-3hl2.onrender.com/mama-fufood`)→ 網站 `js/config.js` 的 `apiBase`、App `env/prod.json` 的 `MFF_API_BASE`。
4. App 簽署金鑰的 SHA-1 → Google Cloud 的 Android 用戶端。

從頭架設照 Part 0、A~J 的順序做。後端是好幾個專案共用的 [router-backend](https://github.com/beethoreven/router-backend),Render 服務本身與它的保活照 router-backend 的 README 建立,這裡只寫 mama-fufood 要加的設定(Part D)。Part K 是從舊的獨立後端 `mama-fufood-backend` 搬過來的步驟,目前只剩 K4。外部服務的帳號由你自己登入、自己設定;畫面上的選單名稱可能和這裡寫的略有不同(Google、Render、Neon 偶爾會改版),意思對得上就可以。每個 Part 最後都有「確認」,沒有通過就先不要往下做。

#### Part 0. 開始前準備

**1. 帳號**

| 服務 | 用來做什麼 | 建議 |
|---|---|---|
| GitHub | 放程式;網站用 GitHub Pages | 用已經設好 SSH 別名 `github.com-beethoreven` 的帳號 |
| Google | 建 Google Cloud 專案(登入與雲端硬碟);也是管理後台的第一位管理員 | 用你自己常用的 Google 帳號;不需要付款資料 |
| Neon | 資料庫 | 用 GitHub 帳號登入 |
| Render | 後端(router-backend 服務) | 用 GitHub 帳號登入 |

**2. 準備一個安全的地方記下這些值**(例如密碼管理器)。標「秘密」的不能貼到聊天、不能進 git。

| 值 | 在哪一步產生 | 要填到哪裡 | 秘密 |
|---|---|---|---|
| 網站網址 `https://beethoreven.github.io/mama-fufood-web/` | 已知(repo 名稱決定) | Google 同意畫面的連結;來源只填 `https://beethoreven.github.io` | |
| Web Client ID(`….apps.googleusercontent.com`) | B7 | Render 的 `MAMA_FUFOOD__GOOGLE_CLIENT_ID`、網站 `js/config.js`、App `env/prod.json` | |
| 除錯金鑰的 SHA-1 | B7 | Google Cloud 的 Android 用戶端(開發用) | |
| Neon 連線字串 | C3 | Render 的 `MAMA_FUFOOD__DATABASE_URL` | **秘密** |
| Session 金鑰 | D2 | Render 的 `MAMA_FUFOOD__SESSION_SECRET` | **秘密** |
| 後端網址 `https://router-backend-3hl2.onrender.com/mama-fufood` | 已知(router-backend 的網址加上 `/mama-fufood`) | 網站 `js/config.js`、App `env/prod.json` | |
| 簽署金鑰檔與它的密碼 | G1 | `android/key.properties`;**一定要備份** | **秘密** |
| 正式金鑰的 SHA-1 | G4 | Google Cloud 的 Android 用戶端(正式版) | |

**3. 本機工具**

- Flutter 與 Android Studio:照 `mama-fufood-app` README 的 Part A 裝好。
- **`keytool` 要用 Android Studio 內附的那一支。** macOS 內建的 `keytool` 沒有 Java 可以跑,會出現「Unable to locate a Java Runtime」;而且在中文環境下,`keytool -list -v` 會因為中文訊息的格式錯誤直接當掉(`IllegalFormatConversionException`),所以每次都加 `-J-Duser.language=en` 改用英文訊息。先在終端機設一個變數,確認這支 keytool 能用(變數只在這個終端機視窗有效;後面用到 keytool 的指令區塊,第一行都會再設一次,換了終端機視窗也能直接整段貼上):

  ```bash
  export KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
  "$KEYTOOL" -J-Duser.language=en -help | head -1
  ```

  第二行印出 `Key and Certificate Management Tool` 就對了。

#### Part A. 建立 repo 與推送

三個 repo(`mama-fufood`、`mama-fufood-app`、`mama-fufood-web`)都設 **Public**:GitHub 免費方案的 Pages 只能從公開 repo 發布。第四個子模組 `router-backend` 是共用的 repo,已經存在,不在這裡建立,它的修改照它自己的 README 推送。秘密(資料庫連線字串、session 金鑰、簽署金鑰)都走環境變數或 `.gitignore`,不會因為公開而外洩。規格與設計文件只留在本機,不進 git(這個 repo 的 `.gitignore` 是白名單)。

★ GitHub 的 SSH 別名是 **`github.com-beethoreven`**,不是純 `github.com`。用錯會推不上去。

★ 下面的指令都用 `git -C <資料夾>` 指定是哪一個 repo,**一律在 meta repo 的資料夾(`mama-fufood/`)裡執行**,不用 `cd` 進出,也就不會在錯的資料夾裡設定或推送。每一步開頭都有「確認是否做過」,做過就跳到下一步。

**A1. 在 GitHub 建立三個空的公開 repo**

- 確認是否做過:瀏覽器打開 `https://github.com/beethoreven/mama-fufood`、`…/mama-fufood-app`、`…/mama-fufood-web`,三個都存在就跳過。
- 做法:到 <https://github.com/new> 建立這三個 repo。Owner 選 `beethoreven`、Visibility 選 **Public**;**不要**勾選加入 README、.gitignore 或授權條款,否則第一次推送會衝突。

**A2. 在本機提交**

- 確認是否做過:

  ```bash
  git log --oneline -1
  git -C mama-fufood-app log --oneline -1
  git -C mama-fufood-web log --oneline -1
  ```

  三個都印出一筆提交,而且 `cat .gitmodules` 列出 `mama-fufood-app`、`mama-fufood-web`、`router-backend` 三個子模組,就跳過。
- 做法:兩個子專案先各自提交,再在 meta 掛上子模組後提交(子模組的資料夾已經是 git repo,`git submodule add` 會直接掛上現有的 repo,不會重新下載;`.gitmodules` 用 https 網址,別人才 clone 得到):

  ```bash
  git -C mama-fufood-app add -A
  git -C mama-fufood-app commit -m "Initial commit"
  git -C mama-fufood-web add -A
  git -C mama-fufood-web commit -m "Initial commit"
  git submodule add https://github.com/beethoreven/mama-fufood-app.git mama-fufood-app
  git submodule add https://github.com/beethoreven/mama-fufood-web.git mama-fufood-web
  git submodule add https://github.com/beethoreven/router-backend.git router-backend
  git add -A
  git commit -m "Add the app, web and router-backend repos as submodules"
  ```

**A3. 設定每個 repo 要推到哪裡(remote)**

- 確認是否做過:

  ```bash
  git remote -v
  git -C mama-fufood-app remote -v
  git -C mama-fufood-web remote -v
  ```

  每個 repo 的 `origin` 網址結尾都要是**它自己的名稱**:meta 是 `…/mama-fufood.git`,`mama-fufood-app` 是 `…/mama-fufood-app.git`,以此類推。三個都對就跳過。
- 沒有 `origin` 的,加上去:

  ```bash
  git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood.git
  git -C mama-fufood-app remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-app.git
  git -C mama-fufood-web remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-web.git
  ```

- 已經有 `origin` 但網址不對的:把那一行的 `remote add` 換成 `remote set-url`。

**A4. 推送(順序不能反:子專案先,meta 最後)**

meta 記錄的是子模組的 commit,先推 meta 會指向 GitHub 上還不存在的 commit。

- 確認是否做過:

  ```bash
  git ls-remote https://github.com/beethoreven/mama-fufood-app.git
  git -C mama-fufood-app log -1 --format=%H
  ```

  第一行印出的 `refs/heads/main` 前面那串編號,要和第二行一樣;`mama-fufood-web` 和 meta 照樣比對(meta 用 `git log -1 --format=%H`)。都一樣就跳過。
- 做法:

  ```bash
  git -C mama-fufood-app push -u origin main
  git -C mama-fufood-web push -u origin main
  git push -u origin main
  ```

**確認**:GitHub 上三個 repo 都看得到各自的檔案;`mama-fufood` repo 裡三個子模組顯示成「資料夾名稱 @ 一串 commit 編號」,點得進去。

**推錯了怎麼辦**:某個 GitHub repo 收到的是別的 repo 的內容(例如 A3 的網址設錯):先用 `remote set-url` 改正網址,再對那個 repo 用 `git -C <資料夾> push --force -u origin main` 覆蓋。強制推送會蓋掉 GitHub 上那個分支原本的內容,只在確定那份內容是推錯的時候用。

#### Part B. Google Cloud(登入與雲端硬碟)

這個 Part 照**繁體中文介面**寫,括號裡是英文介面的名稱。

**B1. 建立專案。** 打開 <https://console.cloud.google.com/> → 上方的專案選單 → **新增專案**(New Project)→ 專案名稱填 `mama-fufood`(位置維持「無機構」)→ **建立**。建好後確認上方的專案選單顯示 `mama-fufood`,之後的步驟都在這個專案裡做。不需要設定付款帳戶。

**B2. 啟用 Google Drive API。** 左上選單 → **API 和服務 → 程式庫**(APIs & Services → Library)→ 搜尋「Google Drive API」→ 點進去 → **啟用**。

**B3. 先讓網站上線(GitHub Pages)。** B4 的「品牌」要填網站的首頁、隱私權政策、服務條款三個網址,Google 會實際打開它們;網站沒上線(打開是 404)就填不完品牌,B6 的「發布應用程式」按鈕也會一直是灰的。網站的首頁和條款頁是純靜態檔,不需要後端就能打開,所以現在就可以上線。

1. 打開 `mama-fufood-web` 的 GitHub 頁面 → **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main`、資料夾 `/ (root)` → Save**。
2. 等一兩分鐘,同一頁上方會顯示「Your site is live at …」和網址。沒出現就是沒存到,回第 1 步確認有按 **Save**。

**確認**:下面三個網址都打得開(不是 404),條款頁顯示的是對應的文件:
- <https://beethoreven.github.io/mama-fufood-web/>
- <https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy>
- <https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms>

**B4. 品牌(Branding)。** 左上選單 → **Google Auth Platform**。第一次進去會先跑「開始使用」(Get started)精靈:應用程式名稱填 `媽媽記副食品`、使用者支援電子郵件選你的 Email、目標對象選**外部**(External)、聯絡資訊填你的 Email、勾選同意 → **建立**。**已經跑過精靈就跳過這段**(左側有「品牌」「目標對象」「用戶端」「資料存取權」等選單就是跑過了)。

接著點左側的 **品牌**,填下面這些欄位(沒提到的欄位維持原樣或留空):

| 區塊 | 欄位 | 值 |
|---|---|---|
| 應用程式資訊 | 應用程式名稱 | `媽媽記副食品`(精靈填過就不用改) |
| 應用程式資訊 | 使用者支援電子郵件 | 你的 Email(精靈選過就不用改) |
| 應用程式標誌 | 標誌 | **不要上傳**:外部、正式版的 App 一上傳標誌就要走 Google 的品牌驗證 |
| 應用程式網域 | 應用程式首頁 | `https://beethoreven.github.io/mama-fufood-web/` |
| 應用程式網域 | 應用程式隱私權政策連結 | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy` |
| 應用程式網域 | 應用程式服務條款連結 | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms` |
| 授權網域 | 新增網域 | `beethoreven.github.io`(**不是** `github.io`;上面三個網址的網域都要先登記在這裡,否則儲存時會報錯) |
| 開發人員聯絡資訊 | 電子郵件地址 | 你的 Email(精靈填過就不用改) |

拉到最下面按 **儲存**。

- 畫面上的欄位名稱和這張表略有不同時,意思對得上就可以。
- 儲存後品牌頁上方如果出現「需要驗證網域擁有權」之類的提示:先不用處理,繼續做 B5、B6;B6 會說發布不了時怎麼辦。

**B5. 資料存取權(Data Access)。** 左側 **資料存取權 → 新增或移除範圍**(Add or remove scopes):
1. 在清單裡勾選 `openid`、`.../auth/userinfo.email`、`.../auth/userinfo.profile`。
2. 在下方「手動新增範圍」(Manually add scopes)貼上這兩行 → **新增至表格**(Add to table):

   ```
   https://www.googleapis.com/auth/drive.appdata
   https://www.googleapis.com/auth/drive.file
   ```

3. 確認這兩個也被勾選 → **更新**(Update)→ 回到資料存取權頁按 **儲存**。

這五個都是 Google 分類的「非敏感」範圍,所以不需要送審,使用者登入時也不會看到「這個應用程式未經驗證」的警告。

**B6. 目標對象:發布成正式版(Audience)。** 左側 **目標對象** → 「發布狀態」目前是**測試中**(Testing)→ **發布應用程式**(Publish app)→ **確認**,狀態變成**正式版**(In production)。

為什麼要發布:測試中的狀態下,名稱、Email、頭像以外的權限(這裡是雲端硬碟)授權 7 天就失效,等於每週都要重新授權一次,而且只有列在「測試使用者」的帳號登得進來。

- **「發布應用程式」按鈕是灰的**:品牌還沒填完。回 B4 檢查三個網址與授權網域都填了、而且 B3 的網站打得開。
- **品牌都填完、網站也打得開,按鈕還是灰的**:2026 年 Google 開發者論壇上有好幾則一樣的回報,Google 沒有給官方解法([論壇文章 1](https://discuss.google.dev/t/cannot-publish-app-publish-app-button-disabled-branding-page-complete-pii-removed-by-staff/401276)、[論壇文章 2](https://discuss.google.dev/t/google-auth-platform-publish-app-disabled-for-external-testing-personal-use-oauth-app/402925))。這時改用替代做法:**維持測試中**,在 **目標對象 → 測試使用者 → 新增使用者**,加入會用到這個 App 的 Google 帳號。代價是雲端硬碟的授權每 7 天要在 App 裡重新同意一次(同步失敗時 App 會請你授權)。之後可以不定期回來試發布。
- 按下發布時 Google 要求驗證網域或送審:**先不要送審**,一樣改用上面的替代做法。驗證 `beethoreven.github.io` 要在 Google Search Console 證明你擁有這個網域,GitHub Pages 的網域只能用「上傳 HTML 檔到網站根目錄」證明,而根目錄要另開一個名叫 `beethoreven.github.io` 的 repo,不在這份 SOP 的範圍。
- 沒有品牌驗證時,同意畫面不會顯示 App 名稱(只顯示網域),不影響功能,不需要申請。

**B7. 建立用戶端(Clients)。** 左側 **用戶端 → 建立用戶端**(Create client),現在先建兩個,第三個等 Part G:

1. 應用程式類型選 **網頁應用程式**(Web application)
   - 名稱:`mama-fufood-web`(只是控制台裡的標籤)。
   - 已授權的 JavaScript 來源(Authorized JavaScript origins)→ **新增 URI**,加兩個:`https://beethoreven.github.io`、`http://localhost:8003`(本機預覽用)。**只有網域**,不含 `/mama-fufood-web/` 路徑,**結尾不要斜線**。
   - 已授權的重新導向 URI:不用填。
   - **建立**之後畫面會顯示用戶端 ID(Client ID),**記下來**(之後在用戶端列表也查得到)。
2. 應用程式類型選 **Android**(開發時用,除錯金鑰簽的版本)
   - 套件名稱(Package name):`tw.com.tutorarthur.mamafufood`
   - SHA-1 憑證指紋(SHA-1 certificate fingerprint):用下面的指令取得,複製 `SHA1:` 後面那一串:

     ```bash
     KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
     "$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA1
     ```

     (`~/.android/debug.keystore` 是第一次 `flutter run` 時自動產生的;找不到這個檔就先跑一次 `flutter run`。)
   - **建立**。Android 用戶端的 ID 不用填到任何地方,Google 用套件名稱加 SHA-1 對應。

**確認**:用戶端列表有一個網頁應用程式、一個 Android;品牌頁的授權網域有 `beethoreven.github.io`;目標對象顯示正式版(或你選了替代做法的測試中加測試使用者)。

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

**C4.** 這串含有資料庫密碼:只貼到 Render 的環境變數(D2 的 `MAMA_FUFOOD__DATABASE_URL`),不要貼到聊天、不要進 git。每個專案要用自己的資料庫:router-backend 發現兩個專案的資料庫連線字串一樣時會拒絕啟動。

**確認**:Neon 的專案列表看得到 `mama-fufood`,區域顯示 Singapore。資料表不用自己建,後端第一次連上時會自動建。

#### Part D. Render 後端(router-backend 上的 mama-fufood)

後端程式在 router-backend 的 `apps/mama_fufood/`,跟著 router-backend 一起部署在 Render 的 `router-backend` 服務上(網址 `https://router-backend-3hl2.onrender.com`,mama-fufood 掛在 `/mama-fufood` 底下)。Render 服務本身(workspace、建立服務、免費額度、保活)由 router-backend 的 README 負責;這裡只加 mama-fufood 的環境變數。

**D1.** 先確認是否做過:打開 `https://router-backend-3hl2.onrender.com/mama-fufood/ready`,`"ok": true` 而且 `"database_configured": true`,就跳到 Part E。

**D2. 環境變數。** 打開 <https://dashboard.render.com/>(用 GitHub 帳號登入)→ `router-backend` 服務 → **Environment → Add Environment Variable**,加上這五個(名稱前面是 `MAMA_FUFOOD__`,**兩個底線**;值**不要加引號**):

| 名稱 | 值 |
|---|---|
| `MAMA_FUFOOD__DATABASE_URL` | C3 的 Neon 連線字串 |
| `MAMA_FUFOOD__GOOGLE_CLIENT_ID` | B7 的 Web Client ID |
| `MAMA_FUFOOD__SESSION_SECRET` | 一串亂數,在終端機執行 `python3 -c "import secrets; print(secrets.token_hex(32))"` 產生。**已經有人在用時不能換**,換了 App 的通行證和後台的登入全部失效,所有人都要重新登入 |
| `MAMA_FUFOOD__ALLOWED_ORIGINS` | `https://beethoreven.github.io`(**結尾不要斜線**,不含路徑) |
| `MAMA_FUFOOD__FIRST_ADMIN_EMAIL` | 你要當第一位管理員的 Google 帳號 Email |

**Save** 之後 Render 會自動重新部署,約 2~5 分鐘。

**確認:**
- `https://router-backend-3hl2.onrender.com/ready`:`"apps"` 裡的 `"mama-fufood"` 是 `"loaded": true`。
- `https://router-backend-3hl2.onrender.com/mama-fufood/ready`:`"ok": true`,`"cache"` 裡有食材清單與單位的版本。第一次可能是 503(第一個請求才開始載入、資料庫在喚醒、第一次連上時會建表並放入第一版的食材清單與單位),等半分鐘再重新整理。

**常見問題:**
- 部署記錄出現「… have the same value」:兩個專案的資料庫連線字串或 session 金鑰填成一樣的,服務不會換成新版;「… wrapped in quotes」:值多了引號。
- `/mama-fufood/ready` 一直是 503、`database_last_ok` 一直是 `null`:`database_configured` 是 `false` 就是 Render 沒讀到 `MAMA_FUFOOD__DATABASE_URL`(名稱拼錯、少一個底線);是 `true` 就到 Render 的 **Logs** 找「公用資料快取」:每 30 秒一行「載入失敗」代表連線字串貼錯(多了空白、少了 `sslmode=require`)或 Neon 專案被暫停。

#### Part E. 保活(不用另外設)

Render 免費方案閒置 15 分鐘會休眠。router-backend 整台共用一個保活(每 10 分鐘打 router 的 `/health`,不碰資料庫),設定在 router-backend 的 README;mama-fufood 不用另外設排程。

#### Part F. 網站(GitHub Pages 與管理後台)

**F1.** 先確認是否做過:打開 `mama-fufood-web/js/config.js`,`googleClientId` 已經有值(`….apps.googleusercontent.com`)就跳到 F3。沒有的話,改 `mama-fufood-web/js/config.js`:

```js
apiBase: "https://router-backend-3hl2.onrender.com/mama-fufood",   // 後端網址,結尾不要斜線
googleClientId: "B7 的 Web Client ID",
```

然後 commit、push。只填「用戶端 ID」;B7 畫面上另外顯示的「用戶端密鑰」(`GOCSPX-` 開頭)這個專案用不到,不要填進任何檔案。

**F2.** GitHub Pages 已在 B3 開啟,這裡不用再做;F1 推送後一兩分鐘,網站就會換成新的設定。

**F3. 確認:**
1. 打開 `https://beethoreven.github.io/mama-fufood-web/` 看得到首頁;條款、政策、判斷邏輯說明三個按鈕都打得開。
2. 右上角用 `FIRST_ADMIN_EMAIL` 的帳號登入(後端剛喚醒時最多等一分鐘)→ 出現「管理後台」。
3. 進後台:看得到 11 個分組、105 種食材;「系統」分頁的快取版本與資料庫版本一致。

**F4.** 要讓其他人也當管理員:後台「管理員」分頁新增他們的 Email。

**常見問題:**
- 首頁沒有登入按鈕:`googleClientId` 是空的,或瀏覽器的擋廣告外掛擋掉了 Google 的登入元件(頁面會顯示原因)。
- Google 的登入視窗顯示來源不符(origin mismatch):B7 的 JavaScript 來源少了 `https://beethoreven.github.io`,或多了路徑、結尾斜線。
- 登入後顯示「連不上後端」:後端在喚醒,等一分鐘再試;一直不行就檢查 `apiBase`。
- 瀏覽器的開發者工具出現 CORS 錯誤:Render 的 `MAMA_FUFOOD__ALLOWED_ORIGINS` 不是剛好 `https://beethoreven.github.io`。

#### Part G. App 的簽署金鑰(學一次就好)

**這把金鑰非常重要:裝到手機之後,之後的每一版都要用同一把簽,否則只能先解除安裝(手機上的資料會不見)。** 金鑰檔和密碼都要備份(例如放進密碼管理器);不能進 git。

**G1. 建立金鑰:**

```bash
mkdir -p ~/Keys/mama-fufood
KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
"$KEYTOOL" -J-Duser.language=en -genkeypair -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

指令會一題一題問:

| 問題 | 怎麼填 |
|---|---|
| Enter keystore password / Re-enter new password | 自己訂一組密碼(至少 6 個字),輸入時畫面不會顯示 |
| What is your first and last name? 以及 organizational unit、organization、City、State、country code | ★ 這些會寫進 App 安裝檔的憑證裡,**任何人拿到 APK 都讀得到**。不想公開就直接按 Enter 留空 |
| Is CN=…, correct? | 輸入 `yes` |

**G2.** 建立 `mama-fufood-app/android/key.properties`(注意是 `android` 資料夾裡,不是 `mama-fufood-app/` 底下;已經在 `.gitignore` 裡)。`storeFile` 要寫完整路徑,`~` 不會被展開:

```properties
storePassword=你剛剛訂的密碼
keyPassword=你剛剛訂的密碼
keyAlias=upload
storeFile=/Users/<你的帳號>/Keys/mama-fufood/upload-keystore.jks
```

`<你的帳號>` 是這台 Mac 的使用者名稱(家目錄的資料夾名稱),不是 Google 或 GitHub 帳號。不確定的話,在終端機跑下面這行,印出來的整串就是 `storeFile` 要填的值:

```bash
echo ~/Keys/mama-fufood/upload-keystore.jks
```

(用這個指令建立的金鑰,金鑰密碼和金鑰庫密碼是同一組。)

**G3. 備份。** 金鑰建好之後只存在這台電腦的 `~/Keys/mama-fufood/upload-keystore.jks`;密碼另外以明文寫在 G2 的 `key.properties`(只在這台電腦、不進 git)。電腦壞掉或遺失,金鑰就沒了,所以要備份到電腦以外的地方:

- **金鑰檔**:複製一份到雲端硬碟(Google 雲端硬碟、iCloud、OneDrive 都可以),或隨身碟。
- **密碼**:放進密碼管理器,**不要和金鑰檔放在同一個地方**。只拿到金鑰檔、沒有密碼,是沒辦法拿來簽 App 的;兩樣分開放,其中一邊外洩也還安全。
- 備份好之後,從雲端下載一次回來,用 G4 的指令加上 `-keystore <下載的檔案>` 跑一次,能印出同一個 SHA-1 就代表備份是好的。

為什麼要備份:App 還沒上架時,手機上的 App 只認這一把金鑰。金鑰不見了,新版就裝不上去,只能先解除安裝(手機上的資料會不見,除非先匯出或開了雲端同步)。上架 Google Play 之後,Google 會另外保管正式的簽署金鑰,這一把只是「上傳金鑰」,遺失時可以向 Google 申請重設。

**G4.** 取得這把金鑰的 SHA-1,回到 Google Auth Platform 的 **用戶端 → 建立用戶端 → 應用程式類型選 Android**(Clients → Create client → Android),建立第三個用戶端(套件名稱一樣是 `tw.com.tutorarthur.mamafufood`):

```bash
KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
"$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -alias upload | grep SHA1
```

**確認**:Google Cloud 的 Clients 列表有兩個 Android 用戶端,SHA-1 不一樣。

#### Part H. 打包與安裝到手機

★ **只是要裝到手機:不用自己打包。** 每一版的安裝檔都放在 [`mama-fufood-app` 的 GitHub Release](https://github.com/beethoreven/mama-fufood-app/releases/latest),網站首頁的「下載最新版媽媽記副食品」也連到這裡。用手機的瀏覽器下載 `mama-fufood-<版本>.apk` 直接安裝(第一次會問要不要允許瀏覽器安裝 App,選允許);更新也一樣,下載新版直接安裝,資料會保留。下面是自己打包、用 USB 線安裝的做法。

**H1.** 在 `mama-fufood-app/env/` 建立 `prod.json`(已經在 `.gitignore` 裡),照 `prod.example.json` 填:

```json
{
  "MFF_API_BASE": "https://router-backend-3hl2.onrender.com/mama-fufood",
  "MFF_WEB_CLIENT_ID": "B7 的 Web Client ID"
}
```

**H2. 打包正式版:**

```bash
cd mama-fufood-app
flutter build apk --release --dart-define-from-file=env/prod.json
```

產出 `build/app/outputs/flutter-apk/app-release.apk`(約 76 MB),同一個資料夾還會有一份檔名帶版本號的 `mama-fufood-<版本>.apk`(例如 `mama-fufood-2.0.0.apk`),內容一樣,要留存或傳給別人時用這份。下面的安裝指令用 `app-release.apk`,它永遠是最新打包的那一版。沒有 `key.properties` 時會直接停下來並說明原因(避免不小心用除錯金鑰簽)。

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
4. Render 的 `router-backend` 服務:`MAMA_FUFOOD__ALLOWED_ORIGINS` = `https://beethoreven.github.io`,`MAMA_FUFOOD__GOOGLE_CLIENT_ID` = Web Client ID。
5. 網站 `js/config.js`:`apiBase` = `https://router-backend-3hl2.onrender.com/mama-fufood`、`googleClientId` = Web Client ID。
6. App `env/prod.json`:`MFF_API_BASE` = `https://router-backend-3hl2.onrender.com/mama-fufood`、`MFF_WEB_CLIENT_ID` = Web Client ID;改了要重新打包。

#### Part J. 驗收

| 步驟 | 預期 |
|---|---|
| 第一次打開 | 深色的啟動畫面(只有圖示)→ 資料運用原則,捲到最下面才能按「我了解了」→ 輸入第一個寶寶名稱(關不掉) |
| 月曆點一天 → 新增紀錄 → 儲存 | 月曆那一格出現「emoji + 食材名稱」 |
| 右上角 Google 登入 | 系統的帳號選擇畫面,登入後換成頭像 |
| 從「最近使用的 App」把 App 滑掉,再打開 | 直接是登入狀態(還是頭像),**不會**跳出選帳號畫面 |
| App 設定 → 檢查更新 | 「公用資料已是最新」,今天還有 9 次 |
| App 設定 → 打開「紀錄與雲端同步」 | 跳出 Google 的雲端硬碟授權;之後「最後同步」有時間 |
| 打開「照片存到雲端」,新增一筆有照片的紀錄 | 雲端硬碟出現「媽媽記副食品/照片」資料夾與照片 |
| 匯入/匯出 → 匯出 | 「匯出完成」,雲端硬碟「媽媽記副食品」資料夾有 `.mff` 檔 |
| 意見回饋 → 送出 | 管理後台「意見回饋」看得到,Email 是你的帳號 |
| 第二支手機登入同一個帳號、打開同步 | 兩支手機的紀錄會合併(同一筆兩邊都改才會問你保留哪一邊) |

有任何一步不對,在 App 的「意見回饋」選「遇到問題」送出。

#### Part K. 搬到共用後端 router-backend

後端從獨立的 `mama-fufood-backend` 搬進好幾個專案共用的 [router-backend](https://github.com/beethoreven/router-backend),網址變成 `https://router-backend-3hl2.onrender.com/mama-fufood`。**K1~K3 已經做完**(網站已改新網址;App 2.0.0 已發布到 GitHub Release),**下一步是 K4**,手機都換成 2.0.0、確認沒問題才做。從頭架設的人不用做這個 Part,Part D 已經是新的做法。**資料庫不搬**:新服務連的就是 Part C 的同一個 Neon 資料庫(區域 ap-southeast-1,和 Render 同區)。程式已經在 router-backend 的 `apps/mama_fufood/`。

★ App 的通行證與後台的登入都不會失效,前提是 K1 的 `MAMA_FUFOOD__SESSION_SECRET` 填**原本的** `SESSION_SECRET`。填了新的值,所有人都要重新登入。

★ 新舊服務會同時連同一個資料庫一段時間,兩邊的每日上限各算各的。等 K3 的手機都換成新版、確認沒問題,才做 K4 關掉舊服務。

**K1. Render 的環境變數。** 先確認是否做過:打開 `https://router-backend-3hl2.onrender.com/ready`,`"apps"` 裡的 `"mama-fufood"` 是 `"loaded": true`,而且 `https://router-backend-3hl2.onrender.com/mama-fufood/ready` 的 `"database_configured"` 是 `true`,就跳到 K2。

到 Render 打開 `mama-fufood-backend` 服務 → **Environment**,把原本的值複製下來;再打開 `router-backend` 服務 → **Environment → Add Environment Variable**,加上這五個(名稱前面多了 `MAMA_FUFOOD__`,**兩個底線**;值不要加引號):

| 名稱 | 值 |
|---|---|
| `MAMA_FUFOOD__DATABASE_URL` | `mama-fufood-backend` 的 `DATABASE_URL`(同一個) |
| `MAMA_FUFOOD__GOOGLE_CLIENT_ID` | `mama-fufood-backend` 的 `GOOGLE_CLIENT_ID` |
| `MAMA_FUFOOD__SESSION_SECRET` | `mama-fufood-backend` 的 `SESSION_SECRET`(**一定要同一個**) |
| `MAMA_FUFOOD__ALLOWED_ORIGINS` | `https://beethoreven.github.io` |
| `MAMA_FUFOOD__FIRST_ADMIN_EMAIL` | `mama-fufood-backend` 的 `FIRST_ADMIN_EMAIL` |

**Save** 之後 Render 會自動重新部署,約 2~5 分鐘。

**確認:**
- `https://router-backend-3hl2.onrender.com/ready`:`"mama-fufood"` 是 `"loaded": true`。
- `https://router-backend-3hl2.onrender.com/mama-fufood/ready`:`"ok": true`,`"cache"` 裡有食材清單與單位的版本。第一次可能是 503(第一個請求才開始載入),等半分鐘再重新整理。
- 部署記錄出現「… have the same value」:兩個專案的 `DATABASE_URL` 或 `SESSION_SECRET` 填成一樣的,服務不會換成新版;「… wrapped in quotes」:值多了引號。

**K2. 網站改用新網址。** 先確認是否做過:`mama-fufood-web/js/config.js` 的 `apiBase` 已經是 `https://router-backend-3hl2.onrender.com/mama-fufood` 就跳到 K3。

把 `apiBase` 改成 `https://router-backend-3hl2.onrender.com/mama-fufood`,重跑示範版的產生工具,提交並推送:

```bash
python3 mama-fufood-web/demo/tools/build.py
git -C mama-fufood-web add -A
git -C mama-fufood-web commit -m "Point the site at the router-backend URL"
git -C mama-fufood-web push
```

**確認**:一兩分鐘後打開管理後台,登入後「總覽」看得到公用資料的版本。

**K3. App 改用新網址並發新版。** 先確認是否做過:`mama-fufood-app/env/prod.json` 的 `MFF_API_BASE` 已經是新網址,而且手機上 App 設定頁的版本是新版,就跳到 K4。

1. `mama-fufood-app/env/prod.json` 的 `MFF_API_BASE` 改成 `https://router-backend-3hl2.onrender.com/mama-fufood`(結尾不要斜線)。
2. 版號升主版本號(換後端是重大修改;規則見 App README),例如 1.2.0 → 2.0.0。
3. 照 Part H 的 H2 打包,發布到 GitHub Release,再把網站 `js/config.js` 的 `apkVersion` 改成新版本;每一支手機從網站下載安裝(或照 H4 用 USB 線安裝)。

**確認**:App 打開後**還是登入狀態**,不用重新登入;App 設定 → 檢查更新 顯示「公用資料已是最新」;送一則意見回饋,管理後台看得到。

**K4. 關掉舊服務(K3 的手機都確認過才做)。** 先確認是否做過:Render 上已經沒有 `mama-fufood-backend`,而且 GitHub 上的 repo 是私人的,就做完了。

1. cron-job.org:刪掉舊的「mama-fufood 保活」工作(打的是舊後端的 `/health`;router-backend 有自己的保活,見 Part E)。
2. Render:打開 `mama-fufood-backend` → **Settings** 最下面 → **Suspend Web Service**(暫停,隨時可以恢復)。觀察幾天沒問題再 **Delete Web Service**。
3. Neon:**不要動**,新服務還在用。
4. meta repo 的 `mama-fufood-backend` 子模組:**已經拿掉了**,跳過。(要確認:`cat .gitmodules` 沒有 `mama-fufood-backend`。第 5 步一定要在這之後做,不然別人 clone meta repo 會失敗。)
5. GitHub:打開 `mama-fufood-backend` repo → **Settings** → 最下面 **Change repository visibility** → **Make private**。

還沒更新的舊版 App 會連不上後端(檢查更新與意見回饋失敗),紀錄不受影響,更新 App 就好。

**確認**:舊網址打不開;新網址的 `/mama-fufood/ready` 還是 `"ok": true`;沒登入 GitHub 的瀏覽器打不開 `mama-fufood-backend` repo。

### 已知限制

- **這份上線 SOP 還沒有照著完整跑過一次**;實際做的時候遇到和這裡不一樣的畫面或錯誤,要回來修正這份文件。Google 同意畫面在發布時會不會要求驗證網域,事先無法確認,所以 B6 寫了替代做法。
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
| Backend API | [`router-backend`](https://github.com/beethoreven/router-backend), `apps/mama_fufood/` (a backend shared by several projects) | Render (`https://router-backend-3hl2.onrender.com/mama-fufood`) |
| Website and admin console | [`mama-fufood-web`](https://github.com/beethoreven/mama-fufood-web) | GitHub Pages (`https://beethoreven.github.io/mama-fufood-web/`) |

The database is Neon Postgres 18 in **ap-southeast-1 (Singapore)**, the same region as Render. This is a prerequisite, not a performance tweak: measured, a single query took 1.76 s across regions and 0.03 s within the same region — 59 times slower. A Neon project's region cannot be changed after creation.

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
cd mama-fufood-app   # or mama-fufood-web, router-backend
git pull origin main
cd ..
git add mama-fufood-app
git commit -m "Update mama-fufood-app submodule pointer"
```

That final `git add` + `commit` cannot be skipped — without it, the submodule's local files are up to date, but this wrapper repo's recorded "which commit to point at" is still stale, so anyone else who `clone`s this repo will still get the old version.

---

### Setup Guide

★ The go-live steps deliberately live here rather than in each subsystem's README, because they **span all three subsystems plus Google Cloud**, with cross-references that nobody could follow end to end once split apart:

1. The Google Cloud "Web application" client ID → router-backend's `MAMA_FUFOOD__GOOGLE_CLIENT_ID`, the website's `googleClientId` in `js/config.js`, and the app's `MFF_WEB_CLIENT_ID` in `env/prod.json` (the same value in all three places).
2. The website's URL → router-backend's `MAMA_FUFOOD__ALLOWED_ORIGINS`, the Google client's "Authorized JavaScript origins", and the Google consent screen's "Authorized domains" and links.
3. The backend's URL (`https://router-backend-3hl2.onrender.com/mama-fufood`) → the website's `apiBase` in `js/config.js` and the app's `MFF_API_BASE` in `env/prod.json`.
4. The SHA-1 of the app's signing key → the Android client in Google Cloud.

For a setup from scratch, follow Part 0 and Parts A–J in order. The backend is [router-backend](https://github.com/beethoreven/router-backend), shared by several projects; the Render service itself and its keep-alive are set up per the router-backend README, and this guide only covers the settings mama-fufood adds (Part D). Part K is the move from the old standalone backend `mama-fufood-backend`; only K4 is left. You sign in to and configure every external service yourself; menu names on screen may differ slightly from what's written here (Google, Render and Neon redesign occasionally) — as long as the meaning matches, you're fine. Every Part ends with a "Check"; don't move on until it passes.

#### Part 0. Before You Start

**1. Accounts**

| Service | What it's for | Suggestion |
|---|---|---|
| GitHub | Hosting the code; the website uses GitHub Pages | The account already set up with the SSH alias `github.com-beethoreven` |
| Google | The Google Cloud project (sign-in and Drive); also the first admin of the admin console | Your own everyday Google account; no billing details needed |
| Neon | Database | Sign in with GitHub |
| Render | Backend (the router-backend service) | Sign in with GitHub |

**2. Prepare a safe place to record these values** (e.g. a password manager). Those marked "secret" must never be pasted into a chat or committed to git.

| Value | Created in step | Goes into | Secret |
|---|---|---|---|
| Website URL `https://beethoreven.github.io/mama-fufood-web/` | Known in advance (set by the repo name) | Google consent screen links; origins only take `https://beethoreven.github.io` | |
| Web Client ID (`….apps.googleusercontent.com`) | B7 | Render's `MAMA_FUFOOD__GOOGLE_CLIENT_ID`, the website's `js/config.js`, the app's `env/prod.json` | |
| SHA-1 of the debug key | B7 | The Google Cloud Android client (development) | |
| Neon connection string | C3 | Render's `MAMA_FUFOOD__DATABASE_URL` | **Secret** |
| Session key | D2 | Render's `MAMA_FUFOOD__SESSION_SECRET` | **Secret** |
| Backend URL `https://router-backend-3hl2.onrender.com/mama-fufood` | Known in advance (the router-backend URL plus `/mama-fufood`) | The website's `js/config.js`, the app's `env/prod.json` | |
| The signing keystore file and its password | G1 | `android/key.properties`; **must be backed up** | **Secret** |
| SHA-1 of the release key | G4 | The Google Cloud Android client (release) | |

**3. Local tools**

- Flutter and Android Studio: install them per Part A of the `mama-fufood-app` README.
- **Use the `keytool` bundled with Android Studio.** macOS's built-in `keytool` has no Java to run on and fails with "Unable to locate a Java Runtime"; and in a Chinese-language environment, `keytool -list -v` crashes on a formatting bug in its Chinese messages (`IllegalFormatConversionException`), so always add `-J-Duser.language=en` to switch it to English messages. Set a variable in the terminal first and confirm this keytool works (the variable only lasts for this terminal window; every later command block that uses keytool sets it again on its first line, so it can be pasted as a whole even in a new terminal window):

  ```bash
  export KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
  "$KEYTOOL" -J-Duser.language=en -help | head -1
  ```

  If the second line prints `Key and Certificate Management Tool`, you're set.

#### Part A. Create the Repos and Push

The three repos (`mama-fufood`, `mama-fufood-app`, `mama-fufood-web`) are all **Public**: on GitHub's free plan, Pages can only publish from a public repo. The fourth submodule, `router-backend`, is a shared repo that already exists; it isn't created here, and its changes are pushed per its own README. Secrets (the database connection string, the session key, the signing key) all go through environment variables or `.gitignore`, so nothing leaks just because the repos are public. Spec and design documents stay on the local machine only and never enter git (this repo's `.gitignore` is an allowlist).

★ The GitHub SSH alias is **`github.com-beethoreven`**, not plain `github.com`. Using the wrong one means the push fails.

★ Every command below names its repo with `git -C <folder>`, and **all of them run from the meta repo's folder (`mama-fufood/`)** — no `cd` back and forth, so nothing gets configured or pushed from the wrong folder. Every step starts with "Check whether it's done"; if it is, move on to the next step.

**A1. Create three empty public repos on GitHub**

- Check whether it's done: open `https://github.com/beethoreven/mama-fufood`, `…/mama-fufood-app` and `…/mama-fufood-web` in a browser; if all three exist, skip this step.
- How: go to <https://github.com/new> and create these three repos. Choose owner `beethoreven` and visibility **Public**; do **not** tick adding a README, .gitignore or license, or the first push will conflict.

**A2. Commit locally**

- Check whether it's done:

  ```bash
  git log --oneline -1
  git -C mama-fufood-app log --oneline -1
  git -C mama-fufood-web log --oneline -1
  ```

  If all three print a commit and `cat .gitmodules` lists the three submodules `mama-fufood-app`, `mama-fufood-web` and `router-backend`, skip this step.
- How: commit each of the two subprojects first, then attach them as submodules in the meta repo and commit (the submodule folders are already git repos, so `git submodule add` attaches the existing repos instead of downloading them again; `.gitmodules` uses https URLs so others can clone):

  ```bash
  git -C mama-fufood-app add -A
  git -C mama-fufood-app commit -m "Initial commit"
  git -C mama-fufood-web add -A
  git -C mama-fufood-web commit -m "Initial commit"
  git submodule add https://github.com/beethoreven/mama-fufood-app.git mama-fufood-app
  git submodule add https://github.com/beethoreven/mama-fufood-web.git mama-fufood-web
  git submodule add https://github.com/beethoreven/router-backend.git router-backend
  git add -A
  git commit -m "Add the app, web and router-backend repos as submodules"
  ```

**A3. Set where each repo pushes to (remote)**

- Check whether it's done:

  ```bash
  git remote -v
  git -C mama-fufood-app remote -v
  git -C mama-fufood-web remote -v
  ```

  Each repo's `origin` URL must end with **its own name**: the meta repo's is `…/mama-fufood.git`, `mama-fufood-app`'s is `…/mama-fufood-app.git`, and so on. If all three are right, skip this step.
- For any repo without an `origin`, add it:

  ```bash
  git remote add origin git@github.com-beethoreven:beethoreven/mama-fufood.git
  git -C mama-fufood-app remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-app.git
  git -C mama-fufood-web remote add origin git@github.com-beethoreven:beethoreven/mama-fufood-web.git
  ```

- For a repo whose `origin` exists but points to the wrong URL: use `remote set-url` instead of `remote add` on that line.

**A4. Push (the order can't be reversed: the subprojects first, the meta repo last)**

The meta repo records the submodules' commits; pushing it first would point at commits that don't exist on GitHub yet.

- Check whether it's done:

  ```bash
  git ls-remote https://github.com/beethoreven/mama-fufood-app.git
  git -C mama-fufood-app log -1 --format=%H
  ```

  The hash in front of `refs/heads/main` on the first line must match the second line; compare `mama-fufood-web` and the meta repo the same way (for the meta repo, use `git log -1 --format=%H`). If they all match, skip this step.
- How:

  ```bash
  git -C mama-fufood-app push -u origin main
  git -C mama-fufood-web push -u origin main
  git push -u origin main
  ```

**Check**: all three repos show their own files on GitHub; in the `mama-fufood` repo the three submodules appear as "folder name @ a commit hash" and can be clicked through.

**If something was pushed to the wrong repo**: when a GitHub repo received another repo's content (e.g. a wrong URL in A3), first fix the URL with `remote set-url`, then overwrite that repo with `git -C <folder> push --force -u origin main`. A force push replaces what that branch on GitHub held before, so only use it when you're sure that content was pushed by mistake.

#### Part B. Google Cloud (Sign-In and Drive)

This Part follows the **Traditional Chinese console UI**; the English UI labels are given alongside.

**B1. Create the project.** Open <https://console.cloud.google.com/> → the project picker at the top → **New Project** (新增專案) → Project name `mama-fufood` (leave Location as "No organization") → **Create**. Once created, make sure the project picker at the top shows `mama-fufood`; every following step happens inside this project. No billing account is needed.

**B2. Enable the Google Drive API.** Top-left menu → **APIs & Services → Library** (API 和服務 → 程式庫) → search "Google Drive API" → open it → **Enable**.

**B3. Put the website online first (GitHub Pages).** Branding in B4 asks for the website's home page, privacy policy and terms of service URLs, and Google actually opens them; while the site is offline (404) Branding can't be completed and B6's "Publish app" button stays greyed out. The home page and legal pages are static files that open without the backend, so the site can go online now.

1. Open the `mama-fufood-web` GitHub page → **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main`, folder `/ (root)` → Save**.
2. Wait a minute or two; "Your site is live at …" and the URL appear at the top of the same page. If they don't, the setting wasn't saved — go back to step 1 and make sure you pressed **Save**.

**Check**: all three URLs open (not 404), and the legal page shows the matching document:
- <https://beethoreven.github.io/mama-fufood-web/>
- <https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy>
- <https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms>

**B4. Branding.** Top-left menu → **Google Auth Platform**. The first visit runs a **Get started** (開始使用) wizard: App name `媽媽記副食品`, User support email: your email, Audience: **External** (外部), Contact information: your email, tick the agreement → **Create**. **Skip this if you've already run the wizard** (if the left side shows Branding, Audience, Clients, Data Access and so on, you have).

Then click **Branding** (品牌) on the left and fill in these fields (leave fields not mentioned as they are or empty):

| Section | Field | Value |
|---|---|---|
| App information | App name | `媽媽記副食品` (no change if the wizard set it) |
| App information | User support email | Your email (no change if the wizard set it) |
| App logo | Logo | **Don't upload one**: for an external app in production, uploading a logo triggers Google's brand verification |
| App domain | Application home page | `https://beethoreven.github.io/mama-fufood-web/` |
| App domain | Application privacy policy link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=privacy` |
| App domain | Application terms of service link | `https://beethoreven.github.io/mama-fufood-web/legal.html?doc=terms` |
| Authorized domains | Add domain | `beethoreven.github.io` (**not** `github.io`; the domain of the three URLs above must be registered here first, or saving fails) |
| Developer contact information | Email addresses | Your email (no change if the wizard set it) |

Scroll to the bottom and press **Save** (儲存).

- If the field names on your screen differ slightly from this table, go by the meaning.
- If, after saving, the Branding page shows a notice like "domain ownership verification required": leave it for now and carry on with B5 and B6; B6 explains what to do if publishing doesn't work.

**B5. Data Access.** Left side **Data Access → Add or remove scopes** (資料存取權 → 新增或移除範圍):
1. Tick `openid`, `.../auth/userinfo.email` and `.../auth/userinfo.profile` in the list.
2. Paste these two lines into "Manually add scopes" (手動新增範圍) at the bottom → **Add to table** (新增至表格):

   ```
   https://www.googleapis.com/auth/drive.appdata
   https://www.googleapis.com/auth/drive.file
   ```

3. Make sure these two are ticked too → **Update** (更新) → back on the Data Access page, press **Save**.

All five are classified by Google as "non-sensitive", so no review is needed and users won't see the "this app isn't verified" warning when signing in.

**B6. Audience: publish to production.** Left side **Audience** (目標對象) → "Publishing status" currently reads **Testing** (測試中) → **Publish app** (發布應用程式) → **Confirm**; the status becomes **In production** (正式版).

Why publish: in Testing status, grants for anything beyond name, email and profile picture (here: Drive) expire after 7 days — re-authorizing every week — and only accounts listed under Test users can sign in.

- **The "Publish app" button is greyed out**: Branding isn't complete. Go back to B4 and check that the three URLs and the authorized domain are filled in, and that the B3 site opens.
- **Branding is complete and the site opens, but the button is still greyed out**: several identical reports were posted on Google's developer forum in 2026, with no official fix from Google ([thread 1](https://discuss.google.dev/t/cannot-publish-app-publish-app-button-disabled-branding-page-complete-pii-removed-by-staff/401276), [thread 2](https://discuss.google.dev/t/google-auth-platform-publish-app-disabled-for-external-testing-personal-use-oauth-app/402925)). Use the fallback: **stay in Testing** and under **Audience → Test users → Add users**, add the Google accounts that will use the app. The cost is that the Drive grant has to be re-approved in the app every 7 days (the app asks for it when sync fails). You can come back and retry publishing now and then.
- If publishing asks for domain verification or a review: **don't submit a review**; use the same fallback. Verifying `beethoreven.github.io` means proving ownership of that domain in Google Search Console, and for a GitHub Pages domain that can only be done by uploading an HTML file to the site root, which needs a separate repo named `beethoreven.github.io` — outside the scope of this guide.
- Without brand verification the consent screen doesn't show the app name (only the domain); this doesn't affect functionality and doesn't need to be applied for.

**B7. Create the clients.** Left side **Clients → Create client** (用戶端 → 建立用戶端); create two now and the third in Part G:

1. Application type **Web application** (網頁應用程式)
   - Name: `mama-fufood-web` (just a label in the console).
   - Authorized JavaScript origins (已授權的 JavaScript 來源) → **Add URI**, add two: `https://beethoreven.github.io`, `http://localhost:8003` (for local preview). **Domain only**, without the `/mama-fufood-web/` path, **no trailing slash**.
   - Authorized redirect URIs: leave empty.
   - After **Create**, the Client ID is shown — **write it down** (it can also be found in the Clients list later).
2. Application type **Android** (for development, builds signed with the debug key)
   - Package name (套件名稱): `tw.com.tutorarthur.mamafufood`
   - SHA-1 certificate fingerprint (SHA-1 憑證指紋): get it with the command below and copy the string after `SHA1:`:

     ```bash
     KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
     "$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA1
     ```

     (`~/.android/debug.keystore` is created automatically by the first `flutter run`; if the file doesn't exist, run `flutter run` once first.)
   - **Create**. The Android client's ID isn't entered anywhere — Google matches it by package name plus SHA-1.

**Check**: the Clients list has one Web application and one Android client; the Branding page's authorized domains include `beethoreven.github.io`; Audience shows In production (or Testing plus Test users if you took the fallback).

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

**C4.** This string contains the database password: paste it only into Render's environment variables (`MAMA_FUFOOD__DATABASE_URL` in D2) — never into a chat, never into git. Each project needs its own database: router-backend refuses to start when two projects share a database connection string.

**Check**: the `mama-fufood` project appears in Neon's project list with the Singapore region. No need to create tables yourself — the backend creates them the first time it connects.

#### Part D. Render Backend (mama-fufood on router-backend)

The backend code lives in router-backend's `apps/mama_fufood/` and deploys with router-backend on the Render service `router-backend` (URL `https://router-backend-3hl2.onrender.com`, with mama-fufood mounted under `/mama-fufood`). The Render service itself (workspace, creating the service, free hours, keep-alive) is covered by the router-backend README; this Part only adds mama-fufood's environment variables.

**D1.** First check whether it's done: if `https://router-backend-3hl2.onrender.com/mama-fufood/ready` shows `"ok": true` and `"database_configured": true`, skip to Part E.

**D2. Environment variables.** Open <https://dashboard.render.com/> (sign in with GitHub) → the `router-backend` service → **Environment → Add Environment Variable** and add these five (the names start with `MAMA_FUFOOD__`, **two underscores**; **no quotes** around values):

| Name | Value |
|---|---|
| `MAMA_FUFOOD__DATABASE_URL` | The Neon connection string from C3 |
| `MAMA_FUFOOD__GOOGLE_CLIENT_ID` | The Web Client ID from B7 |
| `MAMA_FUFOOD__SESSION_SECRET` | A random string: generate it in the terminal with `python3 -c "import secrets; print(secrets.token_hex(32))"`. **Don't change it once people use the app**: a new value invalidates every app pass and admin sign-in, and everyone has to sign in again |
| `MAMA_FUFOOD__ALLOWED_ORIGINS` | `https://beethoreven.github.io` (**no trailing slash**, no path) |
| `MAMA_FUFOOD__FIRST_ADMIN_EMAIL` | The Google account email of the first admin |

After **Save**, Render redeploys automatically (about 2–5 minutes).

**Check:**
- `https://router-backend-3hl2.onrender.com/ready`: `"mama-fufood"` under `"apps"` has `"loaded": true`.
- `https://router-backend-3hl2.onrender.com/mama-fufood/ready`: `"ok": true`, with the food-list and unit versions under `"cache"`. The first visit may be 503 (loading starts on the first request, the database is waking up, and the first connection creates the tables and seeds the first food list and units); wait half a minute and reload.

**Common problems:**
- The deploy log says "… have the same value": two projects share a database connection string or session key, and the new version won't go live; "… wrapped in quotes" means a value has quotes.
- `/mama-fufood/ready` stays at 503 and `database_last_ok` stays `null`: if `database_configured` is `false`, Render isn't reading `MAMA_FUFOOD__DATABASE_URL` (misspelled name, one underscore missing); if it's `true`, search Render's **Logs** for the public-data cache lines (公用資料快取): a load-failure line every 30 seconds means the connection string was pasted wrong (extra spaces, missing `sslmode=require`) or the Neon project is suspended.

#### Part E. Keep-Alive (Nothing to Set Up)

Render's free tier sleeps after 15 minutes idle. router-backend has one keep-alive for the whole service (hitting the router's `/health` every 10 minutes, without touching the database), set up per the router-backend README; mama-fufood needs no schedule of its own.

#### Part F. Website (GitHub Pages and the Admin Console)

**F1.** First check whether it's done: open `mama-fufood-web/js/config.js`; if `googleClientId` already has a value (`….apps.googleusercontent.com`), skip to F3. Otherwise edit `mama-fufood-web/js/config.js`:

```js
apiBase: "https://router-backend-3hl2.onrender.com/mama-fufood",   // the backend URL, no trailing slash
googleClientId: "the Web Client ID from B7",
```

Then commit and push. Fill in only the Client ID; the "Client secret" (starting with `GOCSPX-`) that B7 also shows isn't used by this project — don't put it in any file.

**F2.** GitHub Pages was already turned on in B3, so nothing to do here; a minute or two after F1 is pushed, the site picks up the new settings.

**F3. Check:**
1. `https://beethoreven.github.io/mama-fufood-web/` shows the home page; the terms, privacy policy and judgement-logic guide buttons all open.
2. Sign in at the top right with the `FIRST_ADMIN_EMAIL` account (allow up to a minute if the backend was asleep) → "管理後台" (Admin console) appears.
3. In the console: 11 groups and 105 foods are listed; the "系統" (System) tab shows matching cache and database versions.

**F4.** To make someone else an admin too: add their email on the console's "管理員" (Admins) tab.

**Common problems:**
- No sign-in button on the home page: `googleClientId` is empty, or an ad-blocking extension blocked Google's sign-in script (the page says so).
- Google's sign-in window reports an origin mismatch: B7's JavaScript origins are missing `https://beethoreven.github.io`, or have a path or trailing slash added.
- "連不上後端" (can't reach the backend) after signing in: the backend is waking up — wait a minute and retry; if it never works, check `apiBase`.
- A CORS error in the browser's developer tools: Render's `MAMA_FUFOOD__ALLOWED_ORIGINS` isn't exactly `https://beethoreven.github.io`.

#### Part G. The App's Signing Key (Learn It Once)

**This key is critical: once installed on a phone, every later version must be signed with the same key, or the only way forward is uninstalling (which deletes the phone's data).** Back up both the key file and the password (e.g. in a password manager); they must never enter git.

**G1. Create the key:**

```bash
mkdir -p ~/Keys/mama-fufood
KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
"$KEYTOOL" -J-Duser.language=en -genkeypair -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

The command asks one question at a time:

| Question | What to enter |
|---|---|
| Enter keystore password / Re-enter new password | A password of your choosing (at least 6 characters); nothing shows on screen while typing |
| What is your first and last name? plus organizational unit, organization, City, State, country code | ★ These are written into the certificate inside the app's APK, so **anyone holding the APK can read them**. To keep them private, just press Enter to leave them empty |
| Is CN=…, correct? | Type `yes` |

**G2.** Create `mama-fufood-app/android/key.properties` (inside the `android` folder, not directly under `mama-fufood-app/`; already in `.gitignore`). `storeFile` must be a full path — `~` is not expanded:

```properties
storePassword=the password you just chose
keyPassword=the password you just chose
keyAlias=upload
storeFile=/Users/<your account>/Keys/mama-fufood/upload-keystore.jks
```

`<your account>` is this Mac's user name (the name of your home folder), not a Google or GitHub account. If unsure, run the line below in Terminal; the whole string it prints is the value for `storeFile`:

```bash
echo ~/Keys/mama-fufood/upload-keystore.jks
```

(With a key created by this command, the key password and the keystore password are the same.)

**G3. Back up.** Once created, the key exists only on this computer at `~/Keys/mama-fufood/upload-keystore.jks`; the password is also written in plain text in G2's `key.properties` (on this computer only, never in git). If the computer breaks or is lost, the key is gone, so back it up somewhere off this computer:

- **The key file**: copy it to cloud storage (Google Drive, iCloud, OneDrive are all fine) or a USB drive.
- **The password**: keep it in a password manager, **not in the same place as the key file**. The key file without the password can't be used to sign the app; keeping them apart means one of them leaking is still safe.
- After backing up, download the copy once and run G4's command with `-keystore <the downloaded file>`; if it prints the same SHA-1, the backup is good.

Why back up: until the app is on the store, the installed app only accepts this key. Lose it and new versions can't be installed — only uninstalling works (which deletes the phone's data, unless it was exported or cloud sync was on). Once on Google Play, Google keeps the actual signing key and this one is only the "upload key", which can be reset by asking Google if lost.

**G4.** Get this key's SHA-1 and go back to Google Cloud: **Clients → Create client → Android** to create the third client (the package name is again `tw.com.tutorarthur.mamafufood`):

```bash
KEYTOOL="/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool"
"$KEYTOOL" -J-Duser.language=en -list -v -keystore ~/Keys/mama-fufood/upload-keystore.jks -alias upload | grep SHA1
```

**Check**: Google Cloud's Clients list has two Android clients with different SHA-1s.

#### Part H. Build and Install on a Phone

★ **Just installing on a phone: no need to build.** Every version's APK is on the [`mama-fufood-app` GitHub Releases](https://github.com/beethoreven/mama-fufood-app/releases/latest), and the website's "下載最新版媽媽記副食品" (download the latest version) link points there. Download `mama-fufood-<version>.apk` in the phone's browser and install it (the first time, the phone asks whether to allow the browser to install apps — allow it); updates work the same way, and the data is kept. Below is how to build it yourself and install over USB.

**H1.** Create `prod.json` in `mama-fufood-app/env/` (already in `.gitignore`), following `prod.example.json`:

```json
{
  "MFF_API_BASE": "https://router-backend-3hl2.onrender.com/mama-fufood",
  "MFF_WEB_CLIENT_ID": "the Web Client ID from B7"
}
```

**H2. Build the release:**

```bash
cd mama-fufood-app
flutter build apk --release --dart-define-from-file=env/prod.json
```

The output is `build/app/outputs/flutter-apk/app-release.apk` (about 76 MB). The same folder also gets an identical copy whose name carries the version, `mama-fufood-<version>.apk` (for example `mama-fufood-2.0.0.apk`); use that one when keeping or sending a build. The install command below uses `app-release.apk`, which is always the latest build. Without `key.properties` the build stops and explains why (so it can't accidentally be signed with the debug key).

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
4. The Render service `router-backend`: `MAMA_FUFOOD__ALLOWED_ORIGINS` = `https://beethoreven.github.io`, `MAMA_FUFOOD__GOOGLE_CLIENT_ID` = the Web Client ID.
5. The website's `js/config.js`: `apiBase` = `https://router-backend-3hl2.onrender.com/mama-fufood`, `googleClientId` = the Web Client ID.
6. The app's `env/prod.json`: `MFF_API_BASE` = `https://router-backend-3hl2.onrender.com/mama-fufood`, `MFF_WEB_CLIENT_ID` = the Web Client ID; rebuild after any change.

#### Part J. Acceptance

| Step | Expected |
|---|---|
| First launch | Dark splash screen (icon only) → data-use principles, where "我了解了" (I understand) only becomes tappable after scrolling to the bottom → enter the first baby's name (can't be dismissed) |
| Tap a day on the calendar → add a record → save | That calendar cell shows "emoji + food name" |
| Google sign-in at the top right | The system's account picker; after signing in it turns into your avatar |
| Swipe the app away from recent apps, then reopen it | Still signed in (still your avatar), and **no** account picker appears |
| App settings → check for updates | "公用資料已是最新" (public data is up to date), 9 checks left today |
| App settings → turn on "紀錄與雲端同步" (record cloud sync) | Google's Drive authorization pops up; afterwards "最後同步" (last sync) shows a time |
| Turn on "照片存到雲端" (save photos to the cloud) and add a record with a photo | The "媽媽記副食品/照片" folder and the photo appear in Drive |
| Import/Export → Export | "匯出完成" (export complete), and a `.mff` file appears in the Drive "媽媽記副食品" folder |
| Feedback → send | It shows up under "意見回饋" (Feedback) in the admin console, with your account's email |
| A second phone signs in to the same account and turns on sync | The two phones' records merge (you're only asked which side to keep when both changed the same record) |

If any step doesn't match, send feedback from the app with "遇到問題" (ran into a problem).

#### Part K. Move to the shared router-backend

The backend moves from the standalone `mama-fufood-backend` into [router-backend](https://github.com/beethoreven/router-backend), a service shared by several projects; the URL becomes `https://router-backend-3hl2.onrender.com/mama-fufood`. **K1–K3 are done** (the website uses the new URL; app 2.0.0 is published on GitHub Releases), and **the next step is K4**, once every phone runs 2.0.0 and works. A setup from scratch skips this Part — Part D already describes the new way. **The database doesn't move**: the new service connects to the same Neon database from Part C (region ap-southeast-1, the same as Render). The code is already in router-backend's `apps/mama_fufood/`.

★ App passes and admin sign-ins stay valid, as long as `MAMA_FUFOOD__SESSION_SECRET` in K1 is the **original** `SESSION_SECRET`. With a new value, everyone has to sign in again.

★ For a while both services connect to the same database, each with its own daily limits. Only after every phone runs the new version from K3 and works do you shut the old service down in K4.

**K1. Render environment variables.** First check whether it's done: if `https://router-backend-3hl2.onrender.com/ready` shows `"mama-fufood"` with `"loaded": true` under `"apps"`, and `"database_configured"` is `true` at `https://router-backend-3hl2.onrender.com/mama-fufood/ready`, skip to K2.

In Render, open the `mama-fufood-backend` service → **Environment** and copy its values; then open the `router-backend` service → **Environment → Add Environment Variable** and add these five (the names gain a `MAMA_FUFOOD__` prefix, **two underscores**; no quotes around values):

| Name | Value |
|---|---|
| `MAMA_FUFOOD__DATABASE_URL` | `DATABASE_URL` from `mama-fufood-backend` (the same one) |
| `MAMA_FUFOOD__GOOGLE_CLIENT_ID` | `GOOGLE_CLIENT_ID` from `mama-fufood-backend` |
| `MAMA_FUFOOD__SESSION_SECRET` | `SESSION_SECRET` from `mama-fufood-backend` (**must be the same**) |
| `MAMA_FUFOOD__ALLOWED_ORIGINS` | `https://beethoreven.github.io` |
| `MAMA_FUFOOD__FIRST_ADMIN_EMAIL` | `FIRST_ADMIN_EMAIL` from `mama-fufood-backend` |

After **Save**, Render redeploys automatically (about 2–5 minutes).

**Check:**
- `https://router-backend-3hl2.onrender.com/ready`: `"mama-fufood"` has `"loaded": true`.
- `https://router-backend-3hl2.onrender.com/mama-fufood/ready`: `"ok": true`, with the food-list and unit versions under `"cache"`. The first visit may be 503 (loading starts on the first request); wait half a minute and reload.
- If the deploy log says "… have the same value", two projects share a `DATABASE_URL` or `SESSION_SECRET` and the new version won't go live; "… wrapped in quotes" means a value has quotes.

**K2. Point the website at the new URL.** First check whether it's done: if `apiBase` in `mama-fufood-web/js/config.js` is already `https://router-backend-3hl2.onrender.com/mama-fufood`, skip to K3.

Change `apiBase` to `https://router-backend-3hl2.onrender.com/mama-fufood`, rerun the demo generator, then commit and push:

```bash
python3 mama-fufood-web/demo/tools/build.py
git -C mama-fufood-web add -A
git -C mama-fufood-web commit -m "Point the site at the router-backend URL"
git -C mama-fufood-web push
```

**Check**: a minute or two later, open the admin console; after signing in, the overview shows the public data versions.

**K3. Point the app at the new URL and release a new version.** First check whether it's done: if `MFF_API_BASE` in `mama-fufood-app/env/prod.json` is already the new URL and the app's settings page on the phone shows the new version, skip to K4.

1. Change `MFF_API_BASE` in `mama-fufood-app/env/prod.json` to `https://router-backend-3hl2.onrender.com/mama-fufood` (no trailing slash).
2. Bump the major version (switching backends is a major change; the rules are in the app README), e.g. 1.2.0 → 2.0.0.
3. Build with H2 from Part H, publish it on GitHub Releases, then set `apkVersion` in the website's `js/config.js` to the new version; every phone downloads and installs it from the website (or installs over USB with H4).

**Check**: the app opens **still signed in**, with no new sign-in; App settings → check for updates says the public data is up to date; a feedback submission shows up in the admin console.

**K4. Shut down the old service (only after every phone passed K3).** First check whether it's done: if `mama-fufood-backend` is gone from Render and the GitHub repo is private, you're done.

1. cron-job.org: delete the old "mama-fufood 保活" job (it hits the old backend's `/health`; router-backend has its own keep-alive — see Part E).
2. Render: open `mama-fufood-backend` → bottom of **Settings** → **Suspend Web Service** (can be resumed any time). After a few days without problems, **Delete Web Service**.
3. Neon: **leave it alone**; the new service uses it.
4. The meta repo's `mama-fufood-backend` submodule: **already removed** — skip. (To confirm: `cat .gitmodules` has no `mama-fufood-backend`. Step 5 must come after this, or cloning the meta repo fails for others.)
5. GitHub: open the `mama-fufood-backend` repo → **Settings** → bottom **Change repository visibility** → **Make private**.

Old app versions that weren't updated can no longer reach the backend (update checks and feedback fail); records aren't affected, and updating the app fixes it.

**Check**: the old URL no longer opens; `/mama-fufood/ready` at the new URL still shows `"ok": true`; the `mama-fufood-backend` repo doesn't open in a browser that isn't signed in to GitHub.

### Known Limitations

- **This Setup Guide has not been run end to end yet**; if the real screens or errors differ from what's written here, come back and fix this document. Whether Google's consent screen asks for domain verification when publishing can't be confirmed in advance, which is why B6 includes a fallback.
- Not listed on Google Play yet: the store assets, closed testing, Data safety form and so on needed for listing haven't been done.
- Android developer verification: from 2026-09-30 Google requires app developers to register with Google before their apps can be installed in some countries, expanding worldwide from 2027; installing with `adb` still works. When it takes effect in Taiwan still needs checking.
- Neon free-plan quota: the plans page says both "100 CU-hours per month" and "50" in different places — double-check before creating the project.
