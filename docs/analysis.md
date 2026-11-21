# DV38 APK — Full Static Analysis

**File:** `dv38usdf1_2.4.69_20250926222217.apk`
**Package:** `prod.app_dv38usdf1.com`
**Label:** `DV38`
**Version:** 2.4.69 (versionCode 1, targetSdk 34, minSdk 23)
**SHA-256:** `63bf80f8e92c61fada10f4bd7acf0cc294a87a9c6af015e445a2c0a477bfc668`
**Signing cert SHA-256:** `6D:CD:C1:B1:7C:E0:83:ED:A1:75:47:49:76:FD:60:1F:AB:16:56:65:F8:76:2D:12:46:F6:47:F8:C9:F8:CB:AB`
**Signer:** `C=TW`, serial `23750b4e`, valid 2019–2044

---

## Verdict

**Not a classic banking trojan/spyware**, but **high-risk grayware / PUA**: a white-label gambling-style WebView shell with aggressive tracking, domain rotation, disabled TLS validation, a powerful JavaScript bridge, and an APK download/install path.

No evidence of SMS/contact theft, accessibility abuse, overlays, device admin, root exploits, dynamic DEX loading, or silent install.

---

## 1. Architecture

The APK is a thin shell (~4.8 MB):

- one `classes.dex`
- no native `lib/` payload
- ~27 app-specific classes (`prod.apptest.com.*`); the rest is SDK code
- real content is remote (WebView site)

Bundled SDKs: Firebase (Auth, Messaging, Analytics), Google Play Services, AppsFlyer, Adjust, Facebook SDK, OpenInstall, Tencent X5/Sonic components, OkHttp, Glide.

App-specific classes:

```text
prod.apptest.com.AppTestApp
prod.apptest.com.activity.OpenWebActivity
prod.apptest.com.activity.X5_Activity
prod.apptest.com.fcm.MyFirebaseMessagingService
prod.apptest.com.js.a
prod.apptest.com.webview.*
```

Key obfuscated classes:

```text
F4.H                 WebView wrapper + JS bridges + DownloadListener
F4.r                 WebViewClient (SSL errors ignored)
prod.apptest.com.js.a  @JavascriptInterface bridge
F4.RunnableC0057h    download handler
E1.j                 APK install helper
L4.a / L4.c / L4.f   advertising ID + domain selection
M4.d                 device UUID generation
p120x2.a             telemetry logger
```

---

## 2. Manifest and metadata

Launcher activity: `prod.apptest.com.activity.X5_Activity` (exported).

Permissions include:

- `INTERNET`
- `READ_PHONE_STATE`
- storage (`READ/WRITE_EXTERNAL_STORAGE`, scoped variants)
- `CAMERA`
- location
- notifications

Other manifest facts:

- `usesCleartextTraffic="true"`
- **no** `REQUEST_INSTALL_PACKAGES`
- `FileProvider` authority: `prod.app_dv38usdf1.com.fileProvider`

Embedded metadata:

```text
merchant      = dv38usdf1
merchant_url  = https://www.12249046.com,https://www.dv38.info,
                https://www.dv38.dev,https://www.dv38.pro,https://www.dv38.blog
backupDomain  = https://www2.854819.com:42666
affiliateCode = ...
fbPixelId     = ...
tid           = ...
Firebase      = dv38usdf1-250727
```

`assets/app.json` is unused by the code but contains another merchant's data (`dingtsaiyl`, `sit-dayu88.com`, `xyfenfa.vip`, `datanetwork.cn`), confirming a reused white-label template.

---

## 3. Startup

`prod.apptest.com.AppTestApp.onCreate()`:

1. Reads manifest metadata (merchant, URLs, tracking IDs).
2. Initializes Firebase, Analytics, AppsFlyer, Adjust, Facebook, OpenInstall.
3. Creates a device UUID from `Settings.Secure.ANDROID_ID` / `Build` fields and stores it in `device_stats.xml`.
4. Constructs an update-check URL: `https://<merchant>.www.gfbwff.com/api/app/latest?os=android` (assigned/logged; no clear xref in this build).

`L4.a.b()` reads the Google Advertising ID via `AdvertisingIdClient` and caches it as `gaid_<merchant>`.

---

## 4. Domain selection and URL building

`prod.apptest.com.activity.X5_Activity` → `L4.c` / `L4.f`:

1. Split `merchant_url` on commas.
2. Reuse a cached "domain route" from `SharedPreferences` if valid.
3. Otherwise speed-test the last known URL or up to 20 domains.
4. Pick the fastest reachable domain.
5. Load:

```text
https://<fast-domain>/m/index.html
  ?affiliateCode=...
  &gaid=<Google Advertising ID>
  &fbPixelId=...
  &tid=...
```

Domain rotation/failover is typical of gray-market apps that expect domains to be blocked.

---

## 5. WebView configuration (`F4.H`)

- JavaScript enabled
- DOM storage enabled
- file and content access enabled
- multiple windows / JS popups enabled
- media autoplay enabled
- mixed content **always allowed** (`setMixedContentMode(0)`)
- cleartext HTTP allowed by manifest
- **all TLS certificate errors ignored** — `F4.r.onReceivedSslError()` calls `handler.proceed()`
- `shouldOverrideUrlLoading()` loads any `http*` URL inside the same privileged WebView
- non-http schemes are launched via `Intent.ACTION_VIEW`

Consequence: a third-party page reached through the WebView keeps access to the JavaScript bridge.

---

## 6. JavaScript bridges

`F4.H.g()` registers several `@JavascriptInterface` objects. The main one is `prod.apptest.com.js.a`, exposed as `window.Android`:

| Method | Behavior |
|---|---|
| `openWebView(url)` | Opens a URL in another privileged WebView |
| `openAndroid(intentUri)` | `Intent.parseUri(...)`, adds `BROWSABLE`, `startActivity()` |
| `nativeShare(title, text)` | Text-only `ACTION_SEND` chooser |
| `oauthLogin(provider, callback)` | Firebase sign-in; token returned via `javascript:window.<callback>('...')` |
| `facebookLogin(callback)` | Facebook login flow |
| `generateFcmToken(callback)` / `getFcmToken` | Returns FCM token |
| `storageGetItem` / `storageSetItem` / `storageRemoveItem` | SharedPreferences with `JS_` prefix |
| `receiveBase64FromBlob` | Saves base64 data as image or text file |
| `eventTracker` | Sends events to Firebase Analytics, AppsFlyer, OpenInstall |
| `getVersion` | Returns app version |

Additional bridges: `JSInterface` (`clearCache`), `LaunchOpen`, `LaunchClose`, `Color`, `LogOpen`, `LogClose`, `LogTest`.

The OAuth token callback is a real risk: any page loaded in the WebView can request sign-in and receive the resulting token.

---

## 7. Downloads and APK install path

`F4.H` sets a `DownloadListener`:

- images → saved to gallery (MediaStore)
- other files → `Downloads/GenericFiles`
- `.apk` URLs → progress dialog → `F4.RunnableC0057h` case 0 → `E1.j` case 3

`E1.j` downloads to `getExternalFilesDir(null)/update.apk`, then calls `E1.j.a(File)`:

```text
FileProvider authority: prod.app_dv38usdf1.com.fileProvider
Intent: android.intent.action.INSTALL_PACKAGE
```

The FileProvider config exposes the whole external storage:

```xml
<paths>
    <external-path name="external_storage_root" path="." />
</paths>
```

Because `REQUEST_INSTALL_PACKAGES` is not declared, the install intent normally fails with a `SecurityException` on Android 8+. On Android 6/7 it can still prompt the user.

---

## 8. Tracking and telemetry

- Device UUID from Android ID / Build fields (`M4.d`) → `device_stats.xml`
- Google Advertising ID appended to the loaded URL (`L4.a.b()`)
- AppsFlyer, Adjust, Facebook SDK, Firebase Analytics, OpenInstall
- `p120x2.a.B()` posts error/log messages to the first `backupDomain` entry: `https://www2.854819.com:42666/`
- FCM service (`MyFirebaseMessagingService`) displays notifications with remote images; click opens the app

---

## 9. What was not found

Based on static analysis, there is no evidence of:

- SMS or contact stealing
- accessibility-service abuse
- overlay / clickjacking attacks
- device-admin abuse
- root exploitation
- `DexClassLoader` / dynamic DEX loading
- silent APK installation (install permission missing)
- microphone / call recording
- ransomware behavior
- exfiltration of messages, photos, or contacts

The main harm is financial and privacy-related, plus the security weaknesses above.

---

## 10. Assessment

- **Malware?** Not a classic trojan/stealer, but **high-risk grayware / PUA**.
- **Safe to install?** No.
- **Why?** Sideloaded gambling-style WebView app with domain rotation, heavy tracking, TLS validation disabled, powerful JS bridges, and an APK download/install path.
- **If installed:** uninstall, revoke permissions, check for unknown apps or unexpected charges, and change any passwords entered through it.
- **Second opinion:** upload the SHA-256 to VirusTotal. Static analysis cannot prove the remote site is benign — content can change at any time.

---

## Indicators of compromise

```text
Package:  prod.app_dv38usdf1.com
SHA-256:  63bf80f8e92c61fada10f4bd7acf0cc294a87a9c6af015e445a2c0a477bfc668

Cert SHA-256:
  6D:CD:C1:B1:7C:E0:83:ED:A1:75:47:49:76:FD:60:1F:AB:16:56:65:F8:76:2D:12:46:F6:47:F8:C9:F8:CB:AB

Domains:
  12249046.com
  dv38.info
  dv38.dev
  dv38.pro
  dv38.blog
  www2.854819.com:42666
  www.gfbwff.com

Template leftovers:
  sit-dayu88.com
  xyfenfa.vip
  datanetwork.cn

Firebase project: dv38usdf1-250727

Paths:
  <external-files>/update.apk
  Downloads/GenericFiles/
  SharedPreferences: device_stats.xml, JS_*, gaid_<merchant>
```
