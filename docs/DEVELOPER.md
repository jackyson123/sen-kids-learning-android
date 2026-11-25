# 開發者說明（DEVELOPER.md）

呢份文件記錄 SEN 兒童學習 Android App（TWA 包裝）嘅技術資料，包括點 build、自動 build 同正式上架流程。

- App 名稱：SEN 兒童學習
- Package：`com.jackyson.senkidslearning`
- Start URL：https://sen-kids-learning.vercel.app/
- 版本：versionName 跟住網頁版（而家係 2.3.0），versionCode 每次上架加 1
- App icon：重用網頁版 icon（三個公仔），放喺 `icon-512.png` 同 `app/src/main/res/mipmap-*/`

## 檔案結構

```
sen-android/
├── app/                        Android app 原始碼（TWA 包裝，唔含網頁內容）
│   ├── build.gradle            applicationId、版本、androidbrowserhelper 依賴
│   └── src/main/
│       ├── AndroidManifest.xml LauncherActivity、DEFAULT_URL、顏色、deep link
│       └── res/                icon（mipmap）、顏色、文字
├── .github/workflows/
│   └── build-android.yml       每次 push 自動 build debug APK
├── icon-512.png / icon-192.png  TWA 用嘅大 icon
├── assetlinks.json             Digital Asset Links 範本（未填 fingerprint，見下）
└── gradlew                     Gradle wrapper（Gradle 8.7 + AGP 8.5.2）
```

## 喺自己電腦 build（debug APK）

需要：JDK 17、Android SDK（Command Line Tools 就可以）。

```bash
# 第一次：話俾 Gradle 知 Android SDK 喺邊
echo "sdk.dir=/path/to/Android/sdk" > local.properties

./gradlew assembleDebug
# APK 喺 app/build/outputs/apk/debug/app-debug.apk
```

## GitHub Actions 自動 build

每次 push 到 `main`，workflow 會自動 build debug APK，喺 Actions 頁面下載。
Debug APK 可以直接裝上電話試，但唔可以用嚟上架 Play Store。

## 發佈 Release（比人下載 APK）

最新 debug APK 會放喺 GitHub Release，畀人用手機直接下載安裝：
<https://github.com/jackyson123/sen-kids-learning-android/releases/latest>

出 release 步驟：

1. 喺 Actions 搵最新成功嘅 `Build Android APK` run，確認 APK 冇問題
2. 決定 tag（跟 `versionName`，例如 `v2.3.0`）
3. 將 APK 改名做 `sen-kids-learning-<version>-debug.apk`（例如 `sen-kids-learning-v2.3.0-debug.apk`），再出 release：
   ```bash
   cp app/build/outputs/apk/debug/app-debug.apk sen-kids-learning-v2.3.0-debug.apk
   gh release create v2.3.0 sen-kids-learning-v2.3.0-debug.apk -R jackyson123/sen-kids-learning-android \
       --title "SEN 兒童學習 Android App v2.3.0（Debug APK）" --notes-file notes.md
   ```
4. 開 release 頁面確認 asset 存在、可以下載

## 正式上架步驟（要用人手做，未做）

1. **開 Google Play Console 帳戶**：一次性 $25 美金。用 Jack 嘅 Google 帳戶登記，
   帳戶類型揀個人帳戶。注意：新個人帳戶上架前要做封閉測試（12 個測試員、14 日）。
2. **生成 upload key（上架簽名鑰匙）**：喺自己電腦跑（需要 JDK）：
   ```bash
   keytool -genkeypair -v -keystore upload-key.keystore -alias upload \
     -keyalg RSA -keysize 2048 -validity 10000
   ```
   個 keystore 檔同密碼**絕對唔可以**放入 repo。密碼要另外安全保存。
3. **攞 SHA-256 fingerprint**：
   ```bash
   keytool -list -v -keystore upload-key.keystore -alias upload
   ```
   將 `SHA256:` 後面嗰串值填入 `assetlinks.json` 嘅
   `REPLACE_WITH_UPLOAD_KEY_SHA256_FINGERPRINT` 位置。或者上架後喺 Play Console
   「App integrity」頁面直接 copy upload key 嘅 SHA-256。
4. **放 assetlinks.json 上網站**：將填好咗嘅 `assetlinks.json` 放喺
   `https://sen-kids-learning.vercel.app/.well-known/assetlinks.json`
   （即係要加去 sen-kids-learning 網頁 repo 再 deploy）。未做呢步之前，
   TWA 開出嚟會有瀏覽器網址列，唔係真正全屏。
5. **做 release build**：喺 `app/build.gradle` 嘅 `release` 段加 signingConfig，
   用 GitHub Secrets 存 keystore（base64）同密碼，workflow 先可以出 signed AAB。
   呢個 repo 而家只有 debug build，未有 release 簽名設定。
6. **上架 Play Store**：用 Android Studio 或 `./gradlew bundleRelease` 出 `.aab` 檔，
   上傳去 Play Console，填好商店資料、內容分級、目標受眾（兒童）、私隱政策連結。
   SEN app 本身免費、冇廣告，符合 Google Play Families 政策要求。

## 注意事項

- 呢個包裝本身唔儲存任何用戶資料；app 內嘅星星、進度、錄音全部繼續存喺用戶部手機嘅瀏覽器儲存入面。
- TWA 需要部電話有 Chrome（或支援 Custom Tabs 嘅瀏覽器）先開到；絕大部分 Android 機本身已經有。
- 唔好將 keystore、密碼、簽名檔 commit 入 repo，`.gitignore` 已經擋咗佢哋。
