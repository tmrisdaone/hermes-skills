# Building Android APKs in Proot-Distro on ARM64

Building Android apps (Gradle + AGP) inside proot-distro on ARM64 Termux requires workarounds for architecture-specific SDK binaries.

## ⚠️ Surgical Fix Principle (Critical)

**When fixing a working Android app, only change the specific crash paths. Do NOT rewrite entire files.**

This is the #1 rule. The user's app opens fine but crashes on specific actions (capture, gallery). The fix is:
1. Identify the exact crash path (use parallel subagent analysis)
2. Apply targeted patches to ONLY those lines
3. Keep everything else — imports, structure, patterns — identical to the working original

**What NOT to do:**
- ❌ Don't replace `ctx as LifecycleOwner` with `LocalLifecycleOwner.current` if the original works — it's not the crash cause and can introduce launch crashes
- ❌ Don't replace `runBlocking` with `lifecycleScope.launch` — not the crash cause and can crash on launch
- ❌ Don't restructure the composable tree, rename variables, or "clean up" code
- ❌ Don't add new features or refactor while fixing crashes

**What TO do:**
- ✅ Only touch the lines that cause the crash
- ✅ Use `patch` for targeted edits, not `write_file` for full rewrites
- ✅ Verify the app still opens after each fix batch
- ✅ If you must rewrite a file (e.g., CameraHelper), keep the same class structure, method signatures, and import style

## Stability Ordering (Critical)

When fixing crashes in a working Android app, you MUST test at each step. The stability chain is:

1. **App opens** → if broken, revert immediately
2. **Feature works without crash** → the fix is done

Working patterns (tested, DO NOT change unless they cause the specific crash):
- `AndroidView(factory = { ctx -> ctx as LifecycleOwner })` — works at runtime
- `runBlocking` in `onCreate` for DataStore init — not great style but works
- `preview.setSurfaceProvider(previewView.surfaceProvider)` — method syntax (the property setter may not compile in some CameraX versions)
- Direct Compose state mutation from inside callbacks that already run on main thread

**If a "modern" refactor (`LocalLifecycleOwner.current`, `lifecycleScope.launch`, etc.) breaks launch, revert immediately.** The original pattern was working — the crash is elsewhere.

## Environment Setup

```bash
# Inside proot-distro (Ubuntu/Debian)
apt update && apt install -y openjdk-17-jdk wget unzip

# Install Android SDK command-line tools
cd /opt
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-*.zip
mkdir android-sdk
mv cmdline-tools android-sdk/
export ANDROID_HOME=/opt/android-sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME

# Accept licenses & install SDK platform + build-tools
yes | $ANDROID_HOME/cmdline-tools/bin/sdkmanager --sdk_root=$ANDROID_HOME \
  "platforms;android-34" "build-tools;34.0.0"
```

## Critical: AAPT2 Architecture Fix

The Android SDK's `sdkmanager` downloads **x86_64** aapt2 even on ARM64 devices. This binary won't execute inside proot-distro on ARM64.

### Fix: Replace with system ARM64 aapt2

```bash
# Install system ARM64-native build tools
apt install -y aapt android-sdk-build-tools

# Verify it's ARM64
file /usr/bin/aapt2
# → ELF 64-bit LSB pie executable, ARM aarch64

# Replace the SDK's x86_64 aapt2
cp /opt/android-sdk/build-tools/34.0.0/aapt2 /opt/android-sdk/build-tools/34.0.0/aapt2.x86_64.bak
cp /usr/bin/aapt2 /opt/android-sdk/build-tools/34.0.0/aapt2
chmod +x /opt/android-sdk/build-tools/34.0.0/aapt2
```

### Force Gradle to use override

Add to `gradle.properties`:
```properties
android.aapt2FromMavenOverride=/usr/bin/aapt2
```

This tells AGP to skip its own Maven-downloaded aapt2 (also x86_64) and use the system one.

### Dead End: qemu-user-static

Do NOT attempt to run the x86_64 aapt2 via `qemu-user-static` + `binfmt-support`. Inside proot-distro, proot intercepts `execve` syscalls, so binfmt_misc kernel-level handlers don't fire. The `qemu-x86_64` binary IS available but the dynamic linker (`/lib64/ld-linux-x86-64.so.2`) is not present on ARM64 Ubuntu, and installing x86_64 libraries via multiarch (`dpkg --add-architecture amd64`) fails because they don't exist in the ARM64 repository.

The only working approach is the system ARM64 aapt2 replacement + `android.aapt2FromMavenOverride` in gradle.properties.

## APK Verification

After building, verify the APK is well-formed:

```bash
# Check package name, permissions, SDK versions
aapt dump badging app/build/outputs/apk/debug/app-debug.apk

# Verify all resources compiled correctly
aapt dump resources app/build/outputs/apk/debug/app-debug.apk | grep -i "mipmap\|Theme\|style/Theme"

# Check mipmap/ic_launcher and Theme/CalAI resources are present
```

This is useful when using the old system aapt2 (v2.19) to compile for targetSdk 34 — it catches malformed resource tables before the user installs.

### Missing mipmap/ic_launcher
If the manifest references `@mipmap/ic_launcher` but no icon resources exist:
- Create `app/src/main/res/drawable/ic_launcher_background.xml` (solid color vector)
- Create `app/src/main/res/drawable/ic_launcher_foreground.xml` (icon shape vector)
- Create `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml` (adaptive icon)
- Create `app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml`

Adaptive icon structure for API 26+:
```xml
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background"/>
    <foreground android:drawable="@drawable/ic_launcher_foreground"/>
</adaptive-icon>
```

### Kotlin Compiler Issues
- **Double/Float type mismatch:** `fillMaxWidth(fraction:)` expects `Float`. Convert with `.toFloat()` when passing `Double` values.
- **CameraX Preview API:** ALWAYS use `preview.setSurfaceProvider(previewView.surfaceProvider)` (Java method syntax). The Kotlin property setter `preview.surfaceProvider = previewView.surfaceProvider` can fail to compile with `Unresolved reference: surfaceProvider` on CameraX 1.3.1 / AGP 8.2.2, even though both produce identical bytecode. The property getter `previewView.surfaceProvider` always resolves; it's the property SETTER on `Preview` that may not exist in the API's Kotlin declarations.
- **Unused parameter warning** on `context` in helper functions — harmless, ignore.

## Common Android App Crash Patterns (Jetpack Compose)

### 1. Unsafe LifecycleOwner Cast
```kotlin
// CRASH: ctx may be ContextThemeWrapper, not LifecycleOwner
lifecycleOwner = ctx as LifecycleOwner

// FIX: Use LocalLifecycleOwner.current (BUT verify app still opens)
val lifecycleOwner = LocalLifecycleOwner.current
```

`AndroidView(factory = { ctx -> ... })` passes a `Context` that may not implement `LifecycleOwner`. `LocalLifecycleOwner.current` is the recommended Compose approach.

**⚠️ STABILITY NOTE:** If the app was already using `ctx as LifecycleOwner` and working, this is probably NOT the crash cause. The `ctx` in `AndroidView` IS the Activity in practice. If replacing with `LocalLifecycleOwner.current` causes a launch crash (as it did on some Compose BOM versions), REVERT to the original pattern immediately. The crash was elsewhere. Prefer targeted patching over pattern modernization during debug sprints.

### 2. Background Thread State Mutation
```kotlin
// CRASH: CameraX callback fires on executor thread
capture.takePicture(..., executor, object : OnImageSavedCallback {
    override fun onError(e) {
        onError("failed")  // ← sets Compose mutableStateOf from background thread
    }
})

// FIX: Route callbacks to main thread
ContextCompat.getMainExecutor(context).execute {
    onError("failed")
}
```
CameraX `takePicture()` delivers callbacks on the provided executor. Compose state (`mutableStateOf`) must only be mutated from the main thread.

### 3. Main-Thread IO (Gallery/Content Resolver)
```kotlin
// CRASH: ANR on targetSdk 34
scope.launch {  // ← defaults to Dispatchers.Main
    context.contentResolver.openInputStream(uri)  // ← IO on main thread
}

// FIX: Wrap IO in withContext(Dispatchers.IO)
scope.launch {
    val file = withContext(Dispatchers.IO) {
        context.contentResolver.openInputStream(uri)?.use { ... }
    }
}
```

### 4. OOM from InputStream.copyTo()
```kotlin
// CRASH: OutOfMemoryError on large gallery photos
input.copyTo(output)  // reads entire stream into memory

// FIX: Buffered incremental copy
BufferedInputStream(input).use { buffered ->
    output.use { out ->
        val buffer = ByteArray(8192)
        var bytesRead: Int
        while (buffered.read(buffer).also { bytesRead = it } != -1) {
            out.write(buffer, 0, bytesRead)
        }
    }
}
```

### 5. Null Bitmap from BitmapFactory.decodeFile()
```kotlin
// CRASH: NPE when file is corrupt
val bitmap = BitmapFactory.decodeFile(path)  // can return null
bitmap.compress(...)  // NPE

// FIX: Check for null, add downsampling to prevent OOM
val opts = BitmapFactory.Options().apply { inSampleSize = 2 }
val bitmap = BitmapFactory.decodeFile(path, opts)
if (bitmap == null) { /* fallback */ }
```

### 6. Hardcoded MIME Type for Gallery Images
```kotlin
// BUG: Always sends as image/jpeg even for PNG/WEBP/HEIC
"data:image/jpeg;base64,$base64Image"

// FIX: Detect MIME from file extension
fun detectMimeType(fileName: String): String = when {
    fileName.endsWith(".png") -> "image/png"
    fileName.endsWith(".webp") -> "image/webp"
    fileName.endsWith(".heic") -> "image/heic"
    else -> "image/jpeg"
}
```

### 7. CameraX Executor Shutdown Race
```kotlin
// CRASH: RejectedExecutionException if takePicture callback fires after shutdown
fun release() { executor.shutdown() }

// FIX: Unbind camera first, null out references, then shutdown gracefully
fun release() {
    cameraProvider?.unbindAll()
    cameraProvider = null
    imageCapture = null
    executor.shutdown()
}
```

### 8. Shutter Button Before Camera Ready
```kotlin
// UX BUG: User taps capture before ProcessCameraProvider initializes
// imageCapture is null → silent "Camera not initialized" error

// FIX: Gate shutter on previewReady flag
var previewReady by remember { mutableStateOf(false) }
// In onReady callback: previewReady = true
// Shutter: .clickable(enabled = previewReady) { ... }
```

### 9. runBlocking in Activity.onCreate()
```kotlin
// BUG: Blocks main thread for DataStore read, delays first frame
runBlocking {
    val url = settings.apiBaseUrl.first()
}

// FIX: Use lifecycleScope.launch (BUT verify app still opens)
lifecycleScope.launch {
    val url = settings.apiBaseUrl.first()
}
```

**⚠️ STABILITY NOTE:** `lifecycleScope.launch` in `onCreate` can cause launch crashes on some devices/API levels. If the app was using `runBlocking` and opening fine, this is NOT the crash cause. Revert to `runBlocking` immediately if the app stops opening. The crash is elsewhere. Only modernize this pattern in a dedicated refactor pass, not during crash debugging.

### 10. Missing READ_MEDIA_IMAGES Permission (API 33+)
On Android 13+, `READ_EXTERNAL_STORAGE` is deprecated. For gallery access via `GetContent()`, the system grants temporary URI permissions, but some OEMs may still require `READ_MEDIA_IMAGES`. Prefer `ActivityResultContracts.PickVisualMedia()` (PhotoPicker) on API 33+ which needs no storage permission at all.

## Git Workflow for Android Projects

### .gitignore for Android

```gitignore
*.iml
.gradle
/local.properties
/.idea
.DS_Store
/build
/captures
.externalNativeBuild
.cxx
local.properties
/app/build
/app/release
*.apk
*.aab
*.jks
*.keystore
# Keep the wrapper scripts, not the jar
gradle/wrapper/gradle-wrapper.jar
# Logs
*.log
```

### Initial Commit Pattern

When committing a new Android project, write a structured message covering:
- **What** the app does (one-liner)
- **Core features** (bulleted list)
- **Architecture** (key classes, screens, data flow)
- **Tech stack** (libraries, SDK versions, AGP/Kotlin versions)
- **Platform notes** (ARM64 fixes, any workarounds)

Set git identity first:
```bash
git config user.email "your@email.com"
git config user.name "YourName"
git init && git branch -M main
git add -A && git commit -m "feat: description — subtitle"
```

## Build Command

```bash
cd ~/project-name
export ANDROID_HOME=/opt/android-sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
./gradlew assembleDebug --no-daemon
```

APK output: `app/build/outputs/apk/debug/app-debug.apk`

## Install & Test

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

## Verification

```bash
# Check aapt2 is ARM64
file /opt/android-sdk/build-tools/34.0.0/aapt2
# Should show: ARM aarch64

# Check all x86_64 binaries in build-tools (they may all need replacement)
file /opt/android-sdk/build-tools/34.0.0/* | grep x86-64
```
