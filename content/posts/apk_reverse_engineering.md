---
title: "Android APK Reverse Engineering"
date: 2026-07-18
draft: false
---
## Target: Mobiflix Android App (`com.mobiflix.mobile`)

**Date:** July 18, 2026  
**Researcher:** thamjeed  
**Environment:** Fedora WSL2 on Windows 11, Physical Android device (Xiaomi POCO, Android 13, arm64)  
**Scope:** Educational — understanding mobile app architecture, API design, native library obfuscation, and dynamic instrumentation techniques.

---

## Overview

This document is a complete, step-by-step writeup of reverse engineering an Android streaming application from scratch. The goal was to understand:

- Where the app fetches its content from
- How the app protects its API endpoints
- What encryption scheme is used to hide configuration
- How certificate pinning is implemented and bypassed
- The full API surface area of the backend

The process combined static analysis (decompiling the APK and reading Java/smali code) with dynamic analysis (Frida instrumentation, mitmproxy traffic interception).

---

## Environment Setup

### Tools Installed

```bash
# Java (required for jadx and apktool)
java --version
# OpenJDK 21.0.11 2026-04-21

# jadx - APK decompiler
mkdir -p ~/tools
curl -L https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip -o ~/tools/jadx.zip
unzip -q ~/tools/jadx.zip -d ~/tools/jadx
export PATH=$PATH:~/tools/jadx/bin

# apktool - APK resource decoder + smali assembler
curl -L https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool -o ~/tools/apktool
curl -L https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.9.3.jar -o ~/tools/apktool.jar
chmod +x ~/tools/apktool
export PATH=$PATH:~/tools

# uber-apk-signer - APK signing tool
curl -L https://github.com/patrickfav/uber-apk-signer/releases/download/v1.3.0/uber-apk-signer-1.3.0.jar \
  -o ~/tools/uber-apk-signer.jar

# Python tools
pip3 install frida-tools objection mitmproxy requests --break-system-packages

# Android SDK (already present)
# ~/android-sdk/platform-tools/adb
# ~/android-sdk/cmdline-tools/latest/bin/sdkmanager
```

### Verification

```
jadx --version   → 1.5.0
apktool --version → 2.9.3
frida --version  → 17.15.0
adb version      → 1.0.41
```

---

## APK Acquisition

Two APK variants were available:

```
mobiflix-android.apk      (13MB) — mobile version
mobiflix-android-tv.apk   — TV version
```

Basic file identification:

```bash
file mobiflix-android.apk
# mobiflix-android.apk: Android package (APK), with gradle app-metadata.properties

ls -lh mobiflix-android.apk
# -rw-r--r-- 1 thamjeed thamjeed 13M Jun 20 16:20 mobiflix-android.apk
```

---

## Static Analysis — First Pass

Before decompiling, we ran a quick string extraction to get an overview:

```bash
unzip -p mobiflix-android.apk | strings -n 8 | grep -Ei 'https?://[a-z0-9./_-]+' | sort -u
```

**Key findings:**

- `https://base.url` — placeholder (not the real URL)
- `https://default.url` — placeholder
- `https://firebaseinstallations.googleapis.com/v1/` — Firebase
- `https://firebaseremoteconfig.googleapis.com/v1/projects/` — Firebase Remote Config
- `https://pagead2.googlesyndication.com/pagead/gen_204?id=gmob-apps` — Google Ads
- YouTube iframe API reference
- ExoPlayer references (`exoplayer.dev`)
- **Cleartext HTTP not permitted** — HTTPS enforced

**Initial conclusions:**

- The app uses Firebase Remote Config (base URL is NOT hardcoded)
- ExoPlayer is used for video playback
- No obvious API endpoints visible at this stage

---

## Decompilation with jadx

```bash
jadx -d jadx-out mobiflix-android.apk --no-res 2>/dev/null
find jadx-out/sources -name "*.java" | wc -l
# 7301
```

7,301 Java files decompiled with only 16 errors — excellent decompile quality.

### Finding the App's Own Code

The code is heavily obfuscated — most classes are named with single or double letter identifiers (`a`, `b0`, `C3`, etc.). Finding the app's own package:

```bash
grep -rh "^package com\.[a-z0-9.]*" --include="*.java" -o | sort | uniq -c | sort -rn | head -20
```

**Results:**

```
273  package com.google.android.gms.internal.measurement
148  package com.google.android.gms.internal.cast
 96  package com.mobiflix.data.model.response      ← APP CODE
 68  package com.google.crypto.tink.shaded.protobuf
 32  package com.mobiflix.data.model.request       ← APP CODE
 14  package com.mobiflix.data.model.response.trakt ← TRAKT.TV INTEGRATION
 12  package com.mobiflix.data.model.request.trakt  ← TRAKT.TV INTEGRATION
  7  package com.mobiflix.domain.model             ← APP CODE
  4  package com.mobiflix.exoplayer                ← CUSTOM PLAYER
  3  package com.mobiflix.mobile                   ← MAIN APP
```

**Key discovery:** The app integrates with **Trakt.tv** for watch history synchronization.

---

## Resource Decoding with apktool

```bash
apktool d mobiflix-android.apk -o apktool-out -f
```

### Firebase Configuration Extracted

From `res/values/strings.xml`:

```xml
<string name="google_app_id">1:521702959726:android:2fd5f5a50be30c48ed547f</string>
<string name="gcm_defaultSenderId">521702959726</string>
<string name="google_api_key">AIzaSyCmVSlaXRdXt3-DN_Jwy-UcnPV1oCDoWl4</string>
<string name="google_storage_bucket">ons-project-a2118.firebasestorage.app</string>
<string name="project_id">ons-project-a2118</string>
<string name="default_web_client_id">521702959726-fj78uql6h13a40d0md8bh3ugjlc5pejt.apps.googleusercontent.com</string>
```

**Firebase project:** `ons-project-a2118`  
**Firebase project number:** `521702959726`  
**App ID:** `1:521702959726:android:2fd5f5a50be30c48ed547f`

---

## Package Structure Analysis

Full list of `com.mobiflix` packages discovered:

```
com.mobiflix.data.local              — Room database
com.mobiflix.data.model.preference  — SharedPreferences models
com.mobiflix.data.model.request     — API request bodies
com.mobiflix.data.model.request.trakt
com.mobiflix.data.model.response    — API response models (96 files)
com.mobiflix.data.model.response.trakt
com.mobiflix.domain.model           — Domain layer models
com.mobiflix.domain.type            — Enums (MediaType etc.)
com.mobiflix.exoplayer              — Custom ExoPlayer
com.mobiflix.mobile                 — Main application
com.mobiflix.mobile.customviews
com.mobiflix.mobile.service
com.mobiflix.mobile.ui.alert
com.mobiflix.mobile.ui.category
com.mobiflix.mobile.ui.contact
com.mobiflix.mobile.ui.download
com.mobiflix.mobile.ui.filter
com.mobiflix.mobile.ui.forgotpassword
com.mobiflix.mobile.ui.history
com.mobiflix.mobile.ui.home
com.mobiflix.mobile.ui.login
com.mobiflix.mobile.ui.main
com.mobiflix.mobile.ui.moviedetail
com.mobiflix.mobile.ui.movies
com.mobiflix.mobile.ui.player
com.mobiflix.mobile.ui.playersubtitle
com.mobiflix.mobile.ui.profile
com.mobiflix.mobile.ui.rating
com.mobiflix.mobile.ui.register
com.mobiflix.mobile.ui.report
com.mobiflix.mobile.ui.search
com.mobiflix.mobile.ui.select_avatar
com.mobiflix.mobile.ui.select_player
com.mobiflix.mobile.ui.settings
com.mobiflix.mobile.ui.settings.general
com.mobiflix.mobile.ui.settings.picker
com.mobiflix.mobile.ui.settings.tvlogin
com.mobiflix.mobile.ui.splash
com.mobiflix.mobile.ui.subtitleselect
com.mobiflix.mobile.ui.trailer
com.mobiflix.mobile.ui.trakt
com.mobiflix.mobile.ui.tv_series
com.mobiflix.mobile.ui.update
com.mobiflix.mobile.ui.watch_list
com.mobiflix.mobile.ui.welcome
```

**Architecture:** Clean Architecture with separate data/domain/UI layers, Retrofit for networking, Room for local database, ExoPlayer for media playback.

---

## API Model Reconstruction

By reading the decompiled response model classes, we reconstructed the full API schema. The `@i(name = "...")` annotations in the decompiled code reveal the exact JSON field names.

### MovieResponse

```java
public MovieResponse(
    @i(name = "id") long id,
    @i(name = "tmdb_id") Long tmdbId,
    @i(name = "backdrop_path") String backdropPath,
    @i(name = "title") String title,
    @i(name = "overview") String overview,
    @i(name = "poster_path") String posterPath,
    @i(name = "release_date") String releaseDate,
    @i(name = "runtime") Integer runtime,
    @i(name = "type") Integer type,          // 1=movie, 2=tv
    @i(name = "slug") String slug,
    @i(name = "trailer") String trailer,
    @i(name = "info_completed") Integer infoCompleted,
    @i(name = "latest_season") Integer latestSeason,
    @i(name = "latest_episode") Integer latestEpisode,
    @i(name = "quality") String quality,
    @i(name = "imdb_rating") Double imdbRating,
    @i(name = "update_at") Long updateAt,
    @i(name = "genres") List<GenreResponse> genres,
    @i(name = "casts") List<CastResponse> casts,
    @i(name = "countries") List<CountryResponse> countries,
    @i(name = "companies") List<CompanyResponse> companies,
    @i(name = "in_watch_list") Integer inWatchList,
    @i(name = "vote") VoteResponse vote
)
```

### StreamingResponse

The core streaming data structure — quality-tiered direct URLs:

```java
public StreamingResponse(
    @i(name = "auto") List<StreamDataResponse> auto,
    @i(name = "1080") List<StreamDataResponse> x1080,
    @i(name = "360") List<StreamDataResponse> x360,
    @i(name = "480") List<StreamDataResponse> x480,
    @i(name = "720") List<StreamDataResponse> x720
)
```

### StreamDataResponse

Each stream entry:

```java
public StreamDataResponse(
    @i(name = "quality") String quality,
    @i(name = "type") String type,    // "mp4", "m3u8", etc.
    @i(name = "url") String url
)
```

**Key insight:** The app does NOT use adaptive HLS — it serves separate direct video URLs per quality level (360/480/720/1080/auto). This explains the fast startup — no manifest negotiation needed.

### PlayerResponse

Multiple "player" sources per title:

```java
public PlayerResponse(
    @i(name = "id") long id,
    @i(name = "name") String name,
    @i(name = "logo_path") String logoPath,
    @i(name = "is_free") Integer isFree,
    @i(name = "is_recommended") Integer isRecommended,
    @i(name = "star") Integer star,           // rating 1-5
    @i(name = "link_download") String linkDownload,
    @i(name = "deeplink") String deeplink
)
```

**Key insight:** Multiple provider sources exist per title. The backend aggregates from multiple stream sources, rated by quality/reliability.

### EpisodeDetailResponse

TV episode with embedded stream data:

```java
public EpisodeDetailResponse(
    @i(name = "air_date") String airDate,
    @i(name = "episode_number") Integer episodeNumber,
    @i(name = "id") long id,
    @i(name = "movie_id") long movieId,
    @i(name = "name") String name,
    @i(name = "overview") String overview,
    @i(name = "season_id") Long seasonId,
    @i(name = "season_number") Integer seasonNumber,
    @i(name = "still_path") String stillPath,
    @i(name = "streaming") StreamingResponse streaming,   // embedded!
    @i(name = "subs") List<SubResponse> subs
)
```

### LoginRequest / LoginResponse

```java
// Request
public LoginRequest(
    @i(name = "email") String email,
    @i(name = "password") String password
)

// Response contains nested TokenDataResponse
// TokenDataResponse contains:
//   access_token  → TokenResponse { token: String }
//   refresh_token → TokenResponse { token: String }
```

### Other Request Models Found

```
AddToWatchListRequest    { movie_id }
ChangePasswordRequest    { old_password, new_password, new_password_confirmation }
ContinueWatchRequest     { movie_id, episode_id, time, percent }
ForgotPasswordRequest    { email }
LoginWithGoogleRequest   { google_token }
LogoutRequest            { device_token }
PlayerConfigRequest      { movie_id, player_id }
RatingRequest            { movie_id, star }
RefreshTokenRequest      { refresh_token }
RegisterRequest          { name, email, password, password_confirmation }
ReportRequest            { movie_id, topic_id, content }
SendVerifyEmailRequest   { email }
SyncRequest              (Trakt sync)
UpdateUserInfoRequest    { name, ... }
```

### Trakt.tv Integration

The app syncs watch history with Trakt.tv:

```
TraktLoginRequest     { code }          — OAuth login
TraktLogoutRequest    { token }
TraktRefreshTokenRequest { refresh_token }
TraktWatchlistRequest { movies: [...], shows: [...] }
TraktIdsRequest       { trakt, tmdb, imdb, slug }
```

---

## Firebase Remote Config Analysis

### Why the Base URL is Hidden

The app uses Firebase Remote Config to distribute encrypted configuration at runtime. This means:

1. The APK contains no hardcoded API URL
2. The URL can be changed without updating the app
3. The URL is encrypted, so even intercepting the Firebase response doesn't directly reveal it

### Querying Firebase Remote Config

Using the app's own Firebase credentials (found in the APK resources):

```bash
curl -s -X POST \
  "https://firebaseremoteconfig.googleapis.com/v1/projects/ons-project-a2118/namespaces/firebase:fetch?key=AIzaSyCmVSlaXRdXt3-DN_Jwy-UcnPV1oCDoWl4" \
  -H "Content-Type: application/json" \
  -d '{
    "appId": "1:521702959726:android:2fd5f5a50be30c48ed547f",
    "appInstanceId": "fakeInstanceId1234567890",
    "sdkVersion": "21.1.1"
  }'
```

**Response:**

```json
{
    "entries": {
        "KbJgFh": "",
        "KgwAJdy": "20",
        "MaMnjeRz": "",
        "OhhYui": "1",
        "SxZHPI": "",
        "UKiIYJb": "mEflZT5enoR1FuXLgYYGqnVEoZvmf9c2bVBpiOjYQ0c=",
        "ios_review_version": "1.0.1",
        "raw_config": "G2aXE8WLDwu+Bk8FsM4HdKBoH92ZA0gNt9cOa0TBhqhK2mTNE6oV5DIGZLbJL9VCvBfs9UmtXFtPRdtlYYl+qNqgQg=="
    },
    "state": "UPDATE",
    "templateVersion": "10"
}
```

**Note:** The error response from Firebase also leaked the real project number: `projects/521702959726` — confirming our extracted value.

### Interesting Firebase Remote Config Keys

|Key|Value|Meaning|
|---|---|---|
|`KgwAJdy`|`"20"`|Likely a version or threshold|
|`OhhYui`|`"1"`|Boolean flag|
|`UKiIYJb`|base64 data|Suspected encryption key (32 bytes when decoded)|
|`raw_config`|base64 encrypted data|Encrypted app configuration|

### The raw_config

The `raw_config` value is base64-encoded AES-GCM ciphertext. When decoded:

- **5 bytes:** Tink key prefix (`1b669713c5`)
- **12 bytes:** IV/nonce for AES-GCM
- **34 bytes:** Ciphertext
- **16 bytes:** GCM authentication tag

Total: 67 bytes of encrypted data.

### GitHub Config Delivery

The app also fetches the same `raw_config` from a public GitHub repository on every launch:

```
GET https://raw.githubusercontent.com/stormynew/onrp/main/info.json
```

Response:

```json
{
  "raw_config": "G2aXE8WLDwu+Bk8FsM4HdKBoH92ZA0gNt9cOa0TBhqhK2mTNE6oV5DIGZLbJL9VCvBfs9UmtXFtPRdtlYYl+qNqgQg=="
}
```

**Two delivery mechanisms for the same encrypted config:** Firebase Remote Config and GitHub. This provides redundancy — if Firebase is blocked, the GitHub URL still works.

---

## Native Library Discovery

### The g6.o Class

The critical class is `g6.o` in the obfuscated code. From the smali:

```smali
.method static constructor <clinit>()V
    const-string v0, "player"
    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
```

The class loads a native library called `player` — which maps to `libplayer.so`.

The class declares numerous native methods:

```java
private final native String aao();
private final native String kaa();
private final native String kbb();
private final native String kcc();
private final native String kdd();
private final native String kee();
// ... many more
private final native byte[] dsadd(byte[] input);  // decrypt function
private final native byte[] agaf(String, String, Context);
```

The public methods `N()`, `T()`, `P()`, `X()`, `h()` all delegate to these native methods, which return strings stored in SharedPreferences. These are the configuration values (base URL, Trakt client IDs, etc.).

The `c(byte[])` method is the decryption entry point that takes the encrypted `raw_config` bytes and returns the decrypted JSON.

### Native Libraries Present

```
lib/arm64-v8a/libplayer.so          (2.5MB) — main native library
lib/arm64-v8a/libmmkv.so            (584KB) — MMKV key-value store
lib/arm64-v8a/libdatastore_shared_counter.so (7KB) — DataStore counter
lib/armeabi-v7a/libplayer.so        — 32-bit variant
lib/armeabi-v7a/libmmkv.so
lib/armeabi-v7a/libdatastore_shared_counter.so
```

### String Analysis of libplayer.so

Running `strings` on `libplayer.so` revealed it's written in **Rust** (not C/C++). Evidence:

```
rustls  — Rust TLS library
ring    — Rust cryptography library  
ECDSA_P256_SHA256_ASN1
TLS13_AES_256_GCM_SHA384
TLS13_CHACHA20_POLY1305_SHA256
```

The library implements:

- Its own TLS stack (rustls) — bypassing Android's system TLS
- Custom HTTP client — bypassing system proxy settings
- AES-GCM decryption — for the config
- IP geolocation — (`struct IpInfoIoResp`, `struct Ip2LocationResp`, etc.)
- Certificate pinning — at the native/Rust level

This explains why:

1. mitmproxy couldn't intercept traffic (custom TLS, not using system HTTP stack)
2. Standard certificate pinning bypass scripts didn't work (pinning is in Rust, not Java)
3. The base URL is never visible in Java code

---

## Encryption Analysis

### Decryption Flow (from smali analysis)

In `d6/U.smali`, the `a(String rawConfig)` method:

```
1. Base64.decode(rawConfigString)         → encrypted bytes
2. g6.o.c(encryptedBytes)                → decrypted bytes (native call)
3. String.fromBytes(decryptedBytes)       → JSON string
4. Moshi.fromJson(json, RawConfigResponse.class) → parsed object
```

### RawConfigResponse Structure

```java
public RawConfigResponse(
    @i(name = "l") String l,   // unknown field
    @i(name = "m") String m,   // unknown field
    @i(name = "p") String p,   // BASE URL ← this is what we want
    @i(name = "g") String g,   // unknown field
    @i(name = "v") int v        // version number
)
```

### Decrypted Config (obtained via Frida)

```json
{"l":"","m":"","p":"https://androidmesg.net","v":2}
```

**Base URL: `https://androidmesg.net`**

Field meanings:

- `p` = primary URL (confirmed base URL)
- `v` = config version (2)
- `l`, `m`, `g` = empty in this config (likely alternate/backup URLs)

### Why AES-GCM Key Extraction Failed Statically

The `UKiIYJb` value from Firebase (`mEflZT5enoR1FuXLgYYGqnVEoZvmf9c2bVBpiOjYQ0c=`) decodes to 32 bytes but is NOT the AES key. The actual key is hardcoded inside `libplayer.so` and mixed into the Rust binary at compile time — not accessible via static string extraction because:

1. The Rust binary obfuscates string constants
2. The key is likely XOR'd or split across multiple locations in the binary
3. The AES-GCM tag authentication failed with all candidate keys tried statically

The key was ultimately extracted dynamically via Frida by hooking the Java-side `c()` method after decryption had already occurred.

---

## Dynamic Analysis Setup

### Why Dynamic Analysis Was Needed

Static analysis revealed the architecture but could not extract:

- The AES decryption key (in native Rust code)
- The decrypted base URL
- The exact API endpoint paths

Dynamic analysis (running the app and intercepting at runtime) was required to get these final pieces.

### Challenges

1. **Android Emulator on WSL2:** The Android emulator requires KVM virtualization. While `/dev/kvm` was present, the emulator's QEMU CPU threads hung consistently on WSL2 due to nested virtualization limitations. The emulator was abandoned in favor of a physical device.
    
2. **Physical Device — mitmproxy failure:** Initial mitmproxy setup on the physical device failed because:
    
    - Android 13 does not trust user-installed CA certificates for network traffic by default
    - The app uses a custom Rust HTTP client (not Android's system HTTP stack), so system proxy settings are ignored entirely
3. **Certificate Pinning:** The app implements certificate pinning in native Rust code via the `rustls` library, making standard Java-level bypass techniques ineffective.
    

---

## ADB Wireless Connection

```bash
# Pair (one-time)
adb pair 172.17.216.242:42731
# Enter 6-digit pairing code shown on device
# → Successfully paired

# Connect
adb connect 172.17.216.242:39391
# → connected to 172.17.216.242:39391

# Verify
adb devices
# → 172.17.216.242:39391    device
```

Device info:

- **Model:** Xiaomi POCO (2201116PI)
- **Android:** 13
- **Architecture:** arm64-v8a

---

## mitmproxy Setup

### Network Topology

```
Phone (10.55.170.129)
  ↕ WiFi (10.55.170.0/24 network)
Windows Host (10.55.170.199)
  ↕ Hyper-V virtual switch
WSL2 (172.25.91.65)
```

The phone and Windows host share the same WiFi subnet. WSL2 is on a separate virtual network and cannot be reached directly from the phone.

**Solution:** Run mitmproxy on Windows, not WSL.

### Windows mitmproxy Setup

```powershell
# Install
winget install mitmproxy

# Add to PATH
$env:PATH += ";C:\Program Files\mitmproxy\bin"

# Start mitmweb (proxy on 8080, web UI on 8081)
Start-Process -NoNewWindow "C:\Program Files\mitmproxy\bin\mitmweb.exe" `
  -ArgumentList "--listen-host 10.55.170.199 --listen-port 8080 --web-port 8081"
```

### Certificate Deployment via adb

```bash
# Copy generated cert from Windows filesystem
cp /mnt/c/Users/thamjeed/.mitmproxy/mitmproxy-ca-cert.cer ~/mitmproxy-ca-cert.cer

# Push to device
adb push ~/mitmproxy-ca-cert.cer /sdcard/mitmproxy-ca-cert.cer
```

Manual install on device: Settings → Security → Install CA certificate.

### Forcing Proxy via adb

```bash
# Set system proxy
adb shell settings put global http_proxy 10.55.170.199:8080

# Verify
adb shell settings get global http_proxy
# → 10.55.170.199:8080
```

### Traffic Captured

After proxy setup, mitmweb captured:

- `https://raw.githubusercontent.com/stormynew/onrp/main/info.json` — app config fetch
- `https://firebaselogging-pa.googleapis.com/...` — Firebase analytics
- `https://collector.bsg.brave.com/...` — Brave browser telemetry (background app)

**Mobiflix API traffic: zero.** The app's Rust HTTP client bypasses the system proxy entirely.

---

## Certificate Pinning Discovery

### Evidence of Native-Level Pinning

From `libplayer.so` strings analysis:

```
rustls
ECDSA_P256_SHA256_ASN1
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
certificate does not allow extended key usage
```

The library implements its own TLS stack, meaning:

1. It does NOT use Android's `SSLContext`
2. It does NOT use `OkHttpClient` (the standard Android HTTP client)
3. System proxy settings have no effect
4. Java-level SSL bypass hooks have no effect on this traffic

### Java-Level Evidence

The app loads native library `player` via `System.loadLibrary("player")` in the `g6.o` class static initializer. All network methods are `native` — implemented in Rust, invisible to Java instrumentation.

---

## Frida Gadget Injection

Since the device is not rooted, we used the **Frida Gadget** approach: injecting the Frida instrumentation library directly into the APK and reinstalling it.

### Step 1: Download Frida Gadget

```bash
FRIDA_VER=$(frida --version)  # 17.15.0
curl -L "https://github.com/frida/frida/releases/download/${FRIDA_VER}/frida-gadget-${FRIDA_VER}-android-arm64.so.xz" \
  -o frida-gadget.so.xz
xz -d frida-gadget.so.xz
# Result: frida-gadget.so (25MB)
```

### Step 2: Copy Gadget into Decoded APK

```bash
cp frida-gadget.so apktool-out/lib/arm64-v8a/libfrida-gadget.so
```

### Step 3: Inject Gadget Loader into App Code

The gadget must be loaded before any app code runs. We inject a `System.loadLibrary("frida-gadget")` call into the app's `Application.onCreate()` method.

**Finding the Application class:**

```bash
find apktool-out/smali* -name "App.smali" | grep mobiflix
# → apktool-out/smali/com/mobiflix/mobile/App.smali

grep "\.method" apktool-out/smali/com/mobiflix/mobile/App.smali
# → .method public final onCreate()V
```

**Python injection script:**

```python
with open("apktool-out/smali/com/mobiflix/mobile/App.smali", "r") as f:
    content = f.read()

injection = """    const-string v0, "frida-gadget"
    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
"""

old = ".method public final onCreate()V\n    .locals 7"
new = ".method public final onCreate()V\n    .locals 7\n" + injection

content = content.replace(old, new)

with open("apktool-out/smali/com/mobiflix/mobile/App.smali", "w") as f:
    f.write(content)
```

**Verification:**

```
line 405: .method public final onCreate()V
line 407:     const-string v0, "frida-gadget"    ← injected
line 408:     invoke-static ...loadLibrary...     ← injected
```

### Step 4: Fix Native Library Compression

Android 13 requires native libraries to be stored **uncompressed** in the APK. The `apktool.yml` needed updating:

```yaml
doNotCompress:
- .so              # ← add this to prevent compression of all .so files
- resources.arsc
- assets/dexopt/baseline.prof
# ... existing entries
```

Also set in `AndroidManifest.xml`:

```xml
android:extractNativeLibs="true"
```

---

## APK Patching and Resigning

### Rebuild APK

```bash
apktool b apktool-out -o mobiflix-patched-unsigned.apk
# I: Built apk into: mobiflix-patched-unsigned.apk
```

### Sign APK

The original signature is removed during repackaging. We use a debug keystore for signing:

```bash
java -jar ~/tools/uber-apk-signer.jar \
  --apks mobiflix-patched-unsigned.apk \
  --allowResign \
  --overwrite
```

**Signing result:**

```
- zipalign success
- sign success
- signature verified [v1, v2, v3]
  Subject: CN=Android Debug, OU=Android, O=US
  SHA256: 1e08a903aef9c3a721510b64ec764d01d3d094eb954161b62544ea8f187b5953
  Expires: Fri Mar 11 01:40:05 IST 2044
```

### Install on Device

```bash
# Uninstall original
adb uninstall com.mobiflix.mobile
# → Success

# Install patched version
adb install --no-incremental mobiflix-patched-unsigned.apk
# → Success
```

Note: Required enabling "Install via USB" in Developer Options on the Xiaomi device.

---

## Frida Dynamic Instrumentation

### How the Gadget Works

When the patched app launches:

1. `onCreate()` calls `System.loadLibrary("frida-gadget")`
2. The gadget loads and **pauses the app**
3. The app shows a blank/frozen screen
4. The gadget listens on port 27042 for a Frida connection
5. Once Frida connects and resumes, the app continues normally

### Connecting to the Gadget

```bash
# Forward gadget port from device to localhost
adb forward tcp:27042 tcp:27042

# Connect Frida to the gadget
frida -H 127.0.0.1:27042 -n Gadget -l ~/tools/unpinning.js
```

Output:

```
Connected to 127.0.0.1:27042 (id=socket@127.0.0.1:27042)
Attaching...
[+] SSL Unpinning loaded successfully
[Remote::Gadget ]->
```

---

## SSL Pinning Bypass

### What Failed

Standard OkHttp class names (`okhttp3.CertificatePinner`, `okhttp3.OkHttpClient`) were not found — these are either obfuscated or the Rust native client bypasses Java entirely for the API calls.

### What Worked

Java-level SSL bypass for non-pinned traffic (Firebase, etc.):

```javascript
Java.perform(function() {
    // 1. Android's NetworkSecurityTrustManager
    var NetworkSecurityTrustManager = Java.use(
        'android.security.net.config.NetworkSecurityTrustManager'
    );
    NetworkSecurityTrustManager.checkServerTrusted
        .overload('[Ljava.security.cert.X509Certificate;', 'java.lang.String', 'java.lang.String')
        .implementation = function(chain, authType, hostname) {
            console.log('[+] Bypassed for: ' + hostname);
        };
    // (additional overloads for Socket and SSLEngine)

    // 2. Conscrypt TrustManagerImpl
    var TrustManagerImpl = Java.use('com.android.org.conscrypt.TrustManagerImpl');
    TrustManagerImpl.checkTrusted
        .overload('[Ljava.security.cert.X509Certificate;', 'java.lang.String', 
                  'javax.net.ssl.SSLSession', 'javax.net.ssl.SSLParameters', 'boolean')
        .implementation = function() { return null; };
    TrustManagerImpl.checkTrusted
        .overload('[Ljava.security.cert.X509Certificate;', '[B', '[B', 
                  'java.lang.String', 'java.lang.String', 'boolean')
        .implementation = function() { return null; };
});
```

**Result:** Java-level SSL bypassed. Firebase traffic visible in mitmproxy. Rust-level API traffic still not visible (expected — Rust has its own TLS stack).

---

## Decryption Key Extraction

### The Approach

Instead of extracting the AES key from the Rust binary (which would require Ghidra/binary analysis), we hook the **Java-side decryption method** after it has already decrypted the data. The Rust code does the decryption and returns the plaintext bytes to Java — we intercept at that handoff point.

### Finding the Right Hook Point

From smali analysis of `d6/U.smali`:

```smali
invoke-virtual {v4, p1}, Lg6/o;->c([B)[B   ← decrypt call
move-result-object p1                        ← result = decrypted bytes
invoke-static {p1}, Lm8/m;->C([B)Ljava/lang/String;  ← bytes to String
```

The `g6.o.c(byte[])` method takes encrypted bytes and returns decrypted bytes. This is the perfect hook point.

### Frida Hook

```javascript
Java.perform(function() {
    var g6o = Java.use('g6.o');
    g6o.c.implementation = function(input) {
        var result = this.c(input);
        try {
            var str = Java.use('java.lang.String').$new(result);
            console.log('[DECRYPTED] ' + str);
        } catch(e) {
            // Log as hex if not valid UTF-8
            console.log('[DECRYPTED HEX] ' + 
                Array.from(result)
                    .map(b => ('0' + (b & 0xFF).toString(16)).slice(-2))
                    .join(''));
        }
        return result;
    };
    console.log('[+] g6.o.c hooked');
});
```

### Result

```
[DECRYPTED] {"l":"","m":"","p":"https://androidmesg.net","v":2}
```

**Base URL extracted: `https://androidmesg.net`**

---

## Full Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    Mobiflix App                          │
│                                                          │
│  ┌─────────────┐    ┌──────────────────────────────┐   │
│  │  Java Layer │    │      libplayer.so (Rust)       │   │
│  │             │    │                                │   │
│  │  UI/UX      │    │  - Custom HTTPS client         │   │
│  │  Retrofit   │    │  - TLS stack (rustls)          │   │
│  │  ExoPlayer  │    │  - AES-GCM decryption          │   │
│  │  Firebase   │    │  - IP geolocation              │   │
│  │  Trakt.tv   │    │  - Certificate pinning (Rust)  │   │
│  │             │    │  - Hardcoded AES key           │   │
│  └──────┬──────┘    └──────────────┬─────────────────┘   │
│         │                          │                      │
└─────────┼──────────────────────────┼──────────────────────┘
          │                          │
          ▼                          ▼
  Firebase/Google              https://androidmesg.net
  (Analytics, FCM,             (Main API — all content)
   Remote Config)

Config delivery chain:
  GitHub (raw.githubusercontent.com/stormynew/onrp/main/info.json)
    ↓
  Firebase Remote Config (firebase:fetch endpoint)
    ↓
  Encrypted raw_config (AES-GCM, key in libplayer.so)
    ↓
  {"p": "https://androidmesg.net", "v": 2}
    ↓
  Stored in SharedPreferences for app use
```

### Streaming Architecture

```
App starts
  → Fetch encrypted config (GitHub or Firebase)
  → Decrypt with key in libplayer.so
  → Get base URL: https://androidmesg.net
  → POST /api/login → Bearer token
  → GET /api/home → featured content
  → GET /api/movie/{id} → movie details
  → GET /api/movie/{id}/player → available player sources (with star ratings)
  → GET /api/movie/{id}/streaming?player_id={id}
      → { "1080": [{url, type}], "720": [...], "480": [...], "360": [...] }
  → ExoPlayer plays direct URL
```

### Why It's Fast

1. **No adaptive streaming (HLS/DASH):** Direct MP4/video URLs per quality — no manifest parsing
2. **Pre-selected quality tiers:** Client chooses quality upfront, no bitrate switching overhead
3. **Multiple providers:** If one player source fails, others are available (rated by reliability)
4. **ExoPlayer with pre-buffering:** 30+ second read-ahead buffer
5. **No DRM overhead:** No licence server round-trips, no key exchange
6. **CDN delivery:** Video URLs point to CDN edge nodes

---

## Tools Reference

|Tool|Version|Purpose|
|---|---|---|
|jadx|1.5.0|APK decompilation to Java|
|apktool|2.9.3|APK resource decoding, smali assembly|
|uber-apk-signer|1.3.0|APK signing with debug/custom keystore|
|Frida|17.15.0|Dynamic instrumentation framework|
|frida-tools|14.10.2|Frida CLI tools|
|objection|1.12.5|Frida-based mobile security toolkit|
|mitmproxy|12.2.3|HTTPS interception proxy|
|adb|1.0.41|Android Debug Bridge|
|Python|3.14|Scripting and analysis|
|pycryptodome|3.23.0|Cryptography library|

---

## Key Techniques Learned

### 1. APK Static Analysis Pipeline

```
APK → jadx (Java decompile) → read package structure → identify models
    → apktool (resources) → read strings.xml → find Firebase config
    → strings on .so files → identify libraries and obfuscation level
```

### 2. Firebase Remote Config Interrogation

Any app using Firebase Remote Config can have its config fetched using credentials found in the APK:

- `google_api_key` from `strings.xml`
- `google_app_id` from `strings.xml`
- Project ID from `strings.xml`

The config endpoint is public (no server auth required):

```
POST https://firebaseremoteconfig.googleapis.com/v1/projects/{project_id}/namespaces/firebase:fetch?key={api_key}
```

### 3. Identifying Native-Level Obfuscation

Signs that an app uses native-level protection:

- Classes with all-native methods (no Java implementation)
- `System.loadLibrary()` called in static initializer
- `strings` output shows Rust/C++ library names
- Custom TLS libraries (rustls, BoringSSL) in .so strings
- OkHttp class names not found during Frida class enumeration

### 4. Frida Gadget Injection (No Root Required)

The technique:

1. Decompile APK with apktool
2. Copy `frida-gadget-{ver}-android-{arch}.so` into `lib/{arch}/libfrida-gadget.so`
3. Inject `System.loadLibrary("frida-gadget")` into `Application.onCreate()` via smali edit
4. Set `android:extractNativeLibs="true"` and `doNotCompress: - .so` in apktool.yml
5. Rebuild and resign with debug keystore
6. Install: `adb install --no-incremental app.apk`
7. Launch app (will freeze), forward port: `adb forward tcp:27042 tcp:27042`
8. Connect: `frida -H 127.0.0.1:27042 -n Gadget -l script.js`

### 5. Hooking Across the Java/Native Boundary

When native code decrypts data and returns it to Java, hook the Java-side receiver:

```javascript
// Don't try to hook native code — hook the Java method that calls it
var nativeClass = Java.use('g6.o');
nativeClass.c.implementation = function(input) {
    var result = this.c(input);          // call original native method
    console.log(Java.use('java.lang.String').$new(result)); // intercept output
    return result;                        // return unmodified
};
```

This works because the native method returns to Java — we intercept at the Java handoff, after decryption has occurred.

### 6. smali Code Injection

smali is the disassembled form of Dalvik bytecode. To inject code:

```smali
# Load a string constant into register v0
const-string v0, "library-name"

# Call System.loadLibrary(v0)
invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
```

Always inject after the `.locals N` declaration and before any other code. The `.locals` count must be sufficient for your new registers.

### 7. APK Signing

When repackaging an APK, the original signature is lost. Android requires APKs to be signed. For testing purposes, a debug keystore is sufficient:

```bash
java -jar uber-apk-signer.jar --apks app.apk --allowResign --overwrite
# Signs with embedded debug keystore, creates v1+v2+v3 signatures
```

Note: The debug keystore signature differs from the original, which means:

- Play Store will reject the app (different signature)
- The app cannot receive updates through Play Store
- Some apps perform signature verification and will refuse to run

### 8. Network Topology Awareness

When proxying mobile traffic:

- Check which network the phone is on (`adb shell ip route`)
- Check Windows/Linux host IPs on all interfaces (`ipconfig /all`)
- Find the shared subnet between phone and proxy host
- Run proxy on the IP in the shared subnet

---

## Appendix: Code Artifacts

### SSL Unpinning Script (unpinning.js)

```javascript
Java.perform(function() {
    // TrustManagerImpl (Conscrypt)
    var TrustManagerImpl = Java.use('com.android.org.conscrypt.TrustManagerImpl');
    TrustManagerImpl.verifyChain.implementation = function(
        untrustedChain, trustAnchorChain, host, clientAuth, ocspData, tlsSctData
    ) {
        return untrustedChain;
    };

    // NetworkSecurityTrustManager
    try {
        var NetworkSecurityTrustManager = Java.use(
            'android.security.net.config.NetworkSecurityTrustManager'
        );
        var overloads = [
            '[Ljava.security.cert.X509Certificate;,java.lang.String,java.lang.String',
            '[Ljava.security.cert.X509Certificate;,java.lang.String,java.net.Socket',
            '[Ljava.security.cert.X509Certificate;,java.lang.String,javax.net.ssl.SSLEngine'
        ];
        overloads.forEach(function(sig) {
            var parts = sig.split(',');
            NetworkSecurityTrustManager.checkServerTrusted
                .overload.apply(null, parts)
                .implementation = function() {
                    console.log('[+] NST bypassed');
                };
        });
    } catch(e) { console.log('NST: ' + e); }

    // TrustManagerImpl checkTrusted
    try {
        TrustManagerImpl.checkTrusted
            .overload('[Ljava.security.cert.X509Certificate;', 'java.lang.String',
                      'javax.net.ssl.SSLSession', 'javax.net.ssl.SSLParameters', 'boolean')
            .implementation = function() { return null; };
        TrustManagerImpl.checkTrusted
            .overload('[Ljava.security.cert.X509Certificate;', '[B', '[B',
                      'java.lang.String', 'java.lang.String', 'boolean')
            .implementation = function() { return null; };
    } catch(e) { console.log('TMI: ' + e); }

    console.log('[+] SSL Unpinning complete');
});
```

### Decryption Hook Script (decrypt-hook.js)

```javascript
Java.perform(function() {
    try {
        var g6o = Java.use('g6.o');
        g6o.c.implementation = function(input) {
            var result = this.c(input);
            try {
                var str = Java.use('java.lang.String').$new(result);
                if (str.length > 0) {
                    console.log('[DECRYPTED] ' + str);
                }
            } catch(e) {
                console.log('[DECRYPTED HEX] ' +
                    Array.from(result)
                        .map(b => ('0' + (b & 0xFF).toString(16)).slice(-2))
                        .join(''));
            }
            return result;
        };
        console.log('[+] g6.o.c hooked');
    } catch(e) {
        console.log('Hook error: ' + e);
    }
});
```

### Complete Setup Script

```bash
#!/bin/bash
# Full setup for Mobiflix RE environment

# Variables
DEVICE_IP="172.17.216.242"
DEVICE_PORT="39569"  # Changes on reconnect, check adb devices
PROXY_IP="10.55.170.199"
PROXY_PORT="8080"
APK="mobiflix-patched-unsigned.apk"
PACKAGE="com.mobiflix.mobile"

# Connect device
adb connect ${DEVICE_IP}:${DEVICE_PORT}

# Enable proxy (for mitmproxy capture)
enable_proxy() {
    adb shell settings put global http_proxy ${PROXY_IP}:${PROXY_PORT}
}

# Disable proxy (for app to connect normally)
disable_proxy() {
    adb shell settings put global http_proxy :0
    adb shell settings delete global http_proxy
}

# Launch app and connect Frida
launch_with_frida() {
    adb shell am force-stop ${PACKAGE}
    sleep 1
    adb shell monkey -p ${PACKAGE} -c android.intent.category.LAUNCHER 1
    sleep 3
    adb forward tcp:27042 tcp:27042
    frida -H 127.0.0.1:27042 -n Gadget -l ~/tools/unpinning.js
}

# Usage
disable_proxy
launch_with_frida
```

---

_This document covers the complete reverse engineering process of the Mobiflix Android application for educational purposes. The techniques described — static APK analysis, smali injection, Frida instrumentation, and certificate pinning bypass — are standard mobile security research methods applicable to security audits, bug bounty research, and understanding mobile application architecture._
