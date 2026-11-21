# DV38 — Runtime Workflow

This document describes what happens from install to Telegram sharing and referral credit.

![DV38 runtime workflow](workflow.png)

---

## 1. End-to-end runtime

```mermaid
flowchart TD
    A[User sideloads DV38 APK] --> B["X5_Activity (launcher, exported)"]
    B --> C[AppTestApp.onCreate]
    C --> C1["Read manifest metadata<br/>merchant = dv38usdf1<br/>merchant_url = 12249046.com, dv38.info, ...<br/>backupDomain = www2.854819.com:42666"]
    C --> C2["Init Firebase / Analytics /<br/>AppsFlyer / Adjust / Facebook / OpenInstall"]
    C --> C3["Device UUID from Android ID or Build<br/>→ device_stats.xml"]
    C1 --> D["Domain selection (L4.c / L4.f)"]
    C2 --> D
    C3 --> D
    D --> D1{"Cached domainRoute?"}
    D1 -->|yes| E[Build URL]
    D1 -->|no| D2["Speed-test last URL / up to 20 domains"]
    D2 --> E
    E --> F["https://fast-domain/m/index.html<br/>?affiliateCode=...&gaid=...<br/>&fbPixelId=...&tid=..."]
    F --> G["F4.H WebView<br/>JS enabled, file access enabled<br/>mixed content allowed<br/>SSL errors ignored"]
    G --> H[Remote DV38 website]
    H --> I[JS bridge window.Android]
    H --> J[DownloadListener]
    I --> I1[openAndroid intent URI]
    I --> I2[oauthLogin / facebookLogin]
    I --> I3[nativeShare text]
    I --> I4[storageGet / storageSet]
    I --> I5[eventTracker]
    I3 --> T["Android share sheet → Telegram<br/>text link → preview card"]
    J --> J1[".apk → update.apk → INSTALL_PACKAGE"]
    J --> J2["other files → Downloads/GenericFiles"]
    J --> J3[images → gallery]
```

---

## 2. Step-by-step

### Step 1 — Install

The APK is sideloaded from a link or forwarded file. It is not distributed through Google Play. The manifest requests `INTERNET`, `READ_PHONE_STATE`, storage, camera, location, and notifications.

### Step 2 — Launcher

`X5_Activity` is exported and is the launcher entry point. It immediately initializes `AppTestApp`.

### Step 3 — Configuration

`AppTestApp` reads the merchant metadata from the manifest, starts the tracking SDKs, and generates a stable device identifier.

### Step 4 — Domain selection

`L4.c` / `L4.f` split the merchant URL list and either reuse a cached route or speed-test domains. The fastest reachable domain wins. This is domain rotation/failover.

### Step 5 — WebView

`F4.H` creates a permissive WebView:

- JavaScript on
- DOM storage on
- file access on
- mixed content always allowed
- SSL errors ignored
- any `http*` navigation kept inside the same WebView

### Step 6 — Remote content

The site at `/m/index.html` is the actual product (gambling/betting content). It receives the affiliate code, Google Advertising ID, Facebook pixel ID, and tracking ID.

### Step 7 — Native capabilities

The site can call `window.Android.*` to:

- open another URL in a privileged WebView
- launch arbitrary intents / deep links
- start Firebase/Facebook login and receive the token
- share text
- read/write app storage
- emit tracking events

### Step 8 — Downloads

The `DownloadListener` handles images, generic files, and `.apk` files. APK downloads go to `update.apk` and trigger an install intent via `FileProvider`.

---

## 3. Telegram share flow

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant W as Remote DV38 site
    participant A as Android bridge (window.Android)
    participant S as Android share sheet
    participant T as Telegram
    U->>W: taps Invite / Share
    W->>A: nativeShare(title, text)
    A->>S: ACTION_SEND, type=text/plain
    U->>S: selects Telegram
    S->>T: text containing APK / download URL
    T->>T: fetch link preview
    T-->>U: APK / download card
    U->>T: presses Send
    T-->>U: message delivered to chosen chat
```

### What the APK does

```java
@JavascriptInterface
public void nativeShare(String title, String text) {
    Intent intent = new Intent();
    intent.setType("text/plain");
    intent.setAction("android.intent.action.SEND");
    intent.putExtra("android.intent.extra.TITLE", title);
    intent.putExtra("android.intent.extra.TEXT", text);
    startActivity(Intent.createChooser(intent, title));
}
```

### What the APK does not do

- No `org.telegram` classes
- No `tg://` handler
- No `t.me` / `api.telegram.org`
- No bot token
- No `sendDocument`
- No `EXTRA_STREAM`
- No file attachment
- No silent send

The "file" appearance is Telegram's link preview for a URL that points to an APK or a download page.

`openAndroid(intentUri)` can launch `tg://` links, but the user still confirms.

---

## 4. Referral loop

```mermaid
flowchart LR
    A[User A installs] --> B[Website gives invite/share link]
    B --> C[A shares link/APK to Telegram contacts]
    C --> D[User B downloads and installs]
    D --> E[Site receives affiliateCode + GAID]
    E --> F[B registers]
    F --> G[eventTracker register]
    G --> H[A gets referral credit / bonus]
```

This is a referral/viral loop, not self-replication. It relies on the user tapping Share, choosing Telegram, selecting contacts, and pressing Send.

---

## 5. Background components

```mermaid
flowchart TD
    FCM[Firebase Cloud Messaging] --> MSG[MyFirebaseMessagingService]
    MSG --> N[Notification with remote image]
    N --> APP[Click opens app]

    ERR[Error / telemetry] --> POST["POST https://www2.854819.com:42666/"]

    UPD[Update check] --> URL["https://merchant.www.gfbwff.com/api/app/latest?os=android"]
```

---

## 6. Download and install flow

```mermaid
sequenceDiagram
    autonumber
    participant W as Website
    participant H as F4.H DownloadListener
    participant R as F4.RunnableC0057h
    participant J as E1.j
    participant F as FileProvider
    participant O as Android installer
    W->>H: download URL ending in .apk
    H->>H: show progress dialog
    H->>R: start download
    R->>J: save to getExternalFilesDir()/update.apk
    J->>F: getUriForFile(...)
    J->>O: ACTION_INSTALL_PACKAGE
    O-->>J: SecurityException on Android 8+<br/>(REQUEST_INSTALL_PACKAGES missing)
```

---

## 7. Timeline summary

| Phase | Action |
|---|---|
| T0 | User installs/sideloads APK |
| T1 | `X5_Activity` launches, `AppTestApp` reads config |
| T2 | Tracking SDKs initialize, device UUID generated |
| T3 | Domain speed-test / cached route selection |
| T4 | WebView loads remote site with tracking parameters |
| T5 | User interacts; site calls JS bridge as needed |
| T6 | User taps Invite → text shared to Telegram |
| T7 | Recipient installs and the loop repeats |
| T8 | Site may push APK update / install prompt |
