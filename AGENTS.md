# WildfireChat Android Chat

Multi-module Gradle (Groovy) Android IM app + SDK. Pure Java, ~20 modules.

## Build

- JDK 17, Gradle 8.9 (wrapper), compileSdk/targetSdk 34, minSdk 24 (app) / 21 (libs)
- `./gradlew clean aDebug` — debug APK (CI also uses this)
- `./gradlew clean aR` — release APK
- `./gradlew clean aR && release_sdk.sh` — publish SDK AARs to `Output/`
- Debug APK + minifyEnabled=false breaks audio/video calls (D8 bug with minSdk<24). Fix: `minifyEnabled true` in debug, or use release build

## Modules

| Module | Purpose | Path |
|---|---|---|
| `chat` | Main app (applicationId `cn.wildfirechat.chat.open`) | `chat/` |
| `client` | IM SDK library | `client/` |
| `uikit` | UI kit library | `uikit/` |
| `push` | Direct vendor push (Huawei/Honor/Xiaomi/OPPO/Vivo/Meizu/FCM) | `push/` |
| `mars-core-release` | Protocol stack native AAR (prebuilt, binary-only) | `mars-core-release/` |

Toggleable modules in `settings.gradle`: `push-getui`, `push-jpush`, `momentclient`, `uvccamera`.

## Key Architecture

- Entry: `SplashActivity` → `MyApp.onCreate()` → `WfcUIKit.init()` → `ChatManager.init(application, Config.IM_SERVER_HOST)`
- Config: `uikit/.../Config.java` (IM_SERVER_HOST, ICE_SERVERS, etc.) and `chat/.../AppService.java` (APP_SERVER_ADDRESS)
- Server config passes through: Config → ChatManager → ClientService → ProtoLogic.connect(mHost) (native mars library)

## AAR Patching (Protocol Stack Restriction)

`mars-core-release/mars-core-release.aar` contains the native IM protocol stack. The default AAR's `libmarsstn.so` has `wildfirechat.net` hardcoded and ignores the Java-side `Config.IM_SERVER_HOST`.

**If you need to change the IM server address:**
1. Edit `Config.IM_SERVER_HOST` and `AppService.APP_SERVER_ADDRESS`
2. Run the patching: extract `.so` from AAR, replace `wildfirechat.net` (16 bytes) with null bytes in all 4 architectures (`jni/arm64-v8a/`, `armeabi-v7a/`, `x86/`, `x86_64/`), repack AAR (preserving original ZIP structure with forward-slash paths)
3. `./gradlew clean aDebug`

Alternatively, get the unrestricted AAR from Wildfire Chat (微信: wfchat / wildfirechat) and replace `mars-core-release.aar` directly.

**Important**: When repacking the AAR, use a tool that preserves forward-slash ZIP paths (e.g., Java's ZipOutputStream, not .NET's ZipFile.CreateFromDirectory on Windows).

## Push

- Default: direct vendor push (`:push` module). Requires `google-services.json` and `mcs-services.json` in `chat/` root.
- Alternatives: Getui (`push-getui`), JPush (`push-jpush`) — toggle in `settings.gradle` and `chat/build.gradle`
- Placeholder vendor keys in `chat/build.gradle` `manifestPlaceholders`

## CI

- Manual trigger only (`workflow_dispatch` in `.github/workflows/android.yml`)
- Builds debug APK with `./gradlew clean aDebug`

## Testing

No tests. Only 2 placeholder unit tests in `cameraview/`. No test runner configured.
