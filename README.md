# HGhub TV Patch — 讓 HGhub 在 Android TV / 電視盒上用遙控器操作

把 [HGhub](https://github.com/zxsos/HGhub)（黃果短劇第三方客戶端，Tauri 2 + Rust/React）改成
**遙控器可操作**的電視版：方向鍵移動焦點、OK 進入、返回鍵分層、開機帶聲。

> ⚠️ **這個 repo 只放補丁，不含上游完整原始碼。**
> 上游是 **PolyForm Noncommercial 1.0.0** 授權，本補丁同樣受其約束：**禁止任何商業用途**。
> 與上游作者無關，非官方改造。若上游或權利方要求，會立即下架。

| 首頁（分類） | 播放中 |
| --- | --- |
| ![分類](docs/tv-category.jpg) | ![播放](docs/tv-player.jpg) |

## 內容

```
hghub-tv-tv1.diff      # 13 個檔案的源碼差異（可 git apply，不含 android-tv/）
android-tv/            # 原生覆蓋層：gen/android 被 gitignore，需在 tauri init 後套用
  ├── MainActivity.kt  #   DPAD 橋接 + 允許帶聲自動播放
  ├── apply.mjs        #   一鍵套用（含 manifest banner、applicationId）
  ├── tv_banner.png    #   320×180 Android TV banner
  └── README.md
TV-REMOTE.md           # 改動說明、遙控器操作對照、真機踩到的坑
LICENSE                # PolyForm Noncommercial 1.0.0（含 Required Notice）
```

## 套用

基準 commit：`2c55dfe`（feat(feed): 桌面端切集开关生效…）。上游持續更新，若 `git apply` 有衝突，
用 `git apply -3` 走三方合併，或把 `src/tv/` 與 5 個既有檔的改動手動搬過去（改動很集中）。

```bash
git clone https://github.com/zxsos/HGhub.git hghub-tv
cd hghub-tv
git checkout 2c55dfe
git apply /path/to/hghub-tv-tv1.diff

pnpm install
pnpm tauri android init --ci
node android-tv/apply.mjs        # 套原生改動（DPAD 橋接、TV banner、applicationId）

# 交付給電視盒：arm64 + armv7 通用包
pnpm tauri android build --target aarch64 --target armv7 --apk

# 模擬器除錯：x86_64
pnpm tauri android build --target x86_64 --debug --apk
```

`release` APK 是 **unsigned**，要自己簽：

```bash
zipalign -f 4 app-universal-release-unsigned.apk aligned.apk
apksigner sign --ks ~/.android/debug.keystore --ks-key-alias androiddebugkey \
  --ks-pass pass:android --key-pass pass:android --out HGhub-TV.apk aligned.apk
```

## 做了什麼

1. **焦點引擎** `src/tv/focus.ts`：WebView 不會用方向鍵移動 DOM 焦點，用
   `getBoundingClientRect()` 做幾何最近鄰搜尋；以 capture 掛 `keydown`，處理完
   `preventDefault + stopPropagation`，不跟播放器的按鍵處理打架。
2. **原生 DPAD 橋接** `android-tv/MainActivity.kt`：**這是最關鍵的一項**。部分電視盒
   （實測 Google TV Streamer / Android 14）的 WebView 會把方向鍵當成自己的焦點導航吃掉，
   `keydown` 根本送不到 JS（數字鍵卻正常）。改由 `dispatchKeyEvent` 攔下來呼叫
   `window.__tvKey(code)`，前端再合成一次 keydown，焦點引擎與播放器就都走同一條路。
3. **返回鍵階層**：面板 → 彈窗 → 頁面堆疊 → 回瀏覽頁 → 連按兩次才退出（第一次只提示），
   避免手滑直接掉回桌面。
4. **電視 UI**：字級與卡片放大、底欄 64px、單一黃色焦點框（`::after` + inset box-shadow，
   用 `outline` 會被 `overflow:hidden` 的橫向捲動列裁掉）、320×180 TV banner、
   獨立 `applicationId` 以便與官方版並存。
5. **開機帶聲**：上游 `muteOnStartup` 預設 `true`，這裡改預設不靜音，並用
   `hghub_unmute_v1` 做一次性遷移（舊安裝也會改過來），之後尊重使用者設定；
   另設 `mediaPlaybackRequiresUserGesture = false`，避免 Chromium 把自動播放的影片消音。

## 遙控器操作

| 按鍵 | 作用 |
| --- | --- |
| ↑ ↓ ← → | 移動焦點（幾何最近鄰，含捲動） |
| OK | 進入 / 播放暫停 |
| 播放中 ↑ ↓ | 切上一集／下一集 |
| 播放中 ← → | −5 秒／+5 秒 |
| 返回 | 上一層；根頁連按兩下退出 |
| 1 / 2 / 3 / 4 | 首頁 / 搜索 / 分類 / 我的 |

## 驗證環境

| 裝置 | 系統 | 結果 |
| --- | --- | --- |
| Google TV Streamer | Android 14（32-bit userspace, armeabi-v7a） | 導覽／播放／切集／返回／帶聲全部實測通過 |
| Fire TV Stick 4K Max (`AFTKA`) | Fire OS 9（armeabi-v7a） | 已安裝，遙控器走同一條原生橋接 |
| Android TV 36 模擬器 | x86_64, 1920×1080 | 開發期逐項驗證（焦點日誌 `[tv] focus x= y= w= h=`） |

## 已知限制

- 搜索頁打字仍需輸入法；遙控器可用「熱門題材」或搜尋歷史代替。
- 播放器的詳情／評論／選集面板尚無遙控器入口（上下鍵已能切集）。

## 授權

本補丁是 HGhub 的衍生作品，依 **PolyForm Noncommercial License 1.0.0** 釋出（見 [LICENSE](LICENSE)）。

Required Notice: Copyright (c) 2026 HGhub contributors (https://github.com/Minireel/HGhub)

- 你可以：非商業地使用、修改、散布（需保留本授權與 Required Notice、標明修改）。
- 你不能：任何商業用途（販售、廣告／會員變現、作為收費服務的一部分）。
- 請勿把接口位址、播放直鏈、解密方式公開張貼或打包成「資源站」——上游一旦封禁，所有人都沒得用。
