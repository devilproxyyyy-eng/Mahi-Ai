# Mahi AI (v0.1.0) — Android AI companion + voice assistant

> **Status: source code written, NOT compiled or tested yet.** No build was run when this was generated.
> First build may need small fixes. Verify with the commands below and send me the errors.

## What is included
Kotlin + Jetpack Compose (Material 3) · MVVM · Room · DataStore · Keystore-encrypted secrets · Gemini through a secure backend (Firebase Cloud Function) or a DEV-only direct key.
Screens: Splash, Onboarding (+mic permission), Dashboard (animated orb), Chat, Live Voice, Coding Workspace, App Launcher, Automation Center, History, Memory Manager, Settings, Gemini Config, Privacy, About.

| Feature | State |
|---|---|
| Chat + local history (Room) | implemented |
| Gemini via backend / direct dev key | implemented |
| Voice (turn-based STT → Gemini → TTS) | implemented; **not** realtime/full-duplex |
| App launcher, favorites | implemented |
| YouTube search (confirm first, app → browser fallback) | implemented |
| Command parser + strict allowlist + confirmations | implemented, unit-tested |
| AccessibilityService (tap-by-label, scroll, back/home, Stop button) | implemented; swipe gestures & multi-step sequences **not** included |
| Coding workspace (generate/explain/debug/format/README, copy, save, export) | implemented; **no syntax highlighting** |
| Memory (manual add/edit/delete, 📌 from chat) | implemented |
| Firebase sign-in UI in the app | **not included** (backend accepts Firebase ID tokens or access codes) |
| Reminders | timer only (`timer 5 minutes`) |

## Architecture
```
Android app ──HTTPS + Bearer token──▶ Cloud Function `api` ──▶ Gemini API
 ui/ data/ ai/ voice/ automation/ launcher/ security/          (key in Secret Manager)
```
Model replies may end with `ACTION: {json}`; it is parsed to a fixed schema, validated against an allowlist, and confirmed before running. Raw AI output never executes.

## Gemini models (important)
`gemini-2.5-*` models were scheduled for shutdown on **16 Oct 2026**. Defaults here are `gemini-3.5-flash`, `gemini-3.1-flash-lite`, `gemini-3.1-pro-preview`. Names change — check https://ai.google.dev/gemini-api/docs/models and edit `ALLOWED_MODELS` in `backend/functions/src/index.ts`, or type any model in Settings → Gemini.

## Backend setup (needs internet, a Google account, Blaze billing plan for Cloud Functions + Secret Manager)
```bash
cd backend
npm i -g firebase-tools
firebase login && firebase use --add          # pick your project
cd functions && npm install && cd ..
firebase functions:secrets:set GEMINI_API_KEY   # paste Gemini key
firebase functions:secrets:set ACCESS_CODES     # e.g. a long random string (comma-separate for several)
firebase deploy --only functions,firestore:rules
```
Use the printed function URL (https://…run.app) as **Backend URL** in the app, and the access code as the **token**. Quotas: 10 req/min and 300/day per user (edit constants). Enable Firebase Authentication/App Check in the console if you add sign-in later.
Quota docs live in Firestore collection `quota` (rules deny all client access). Set a Firestore TTL policy on field `exp`.

## Direct DEV mode
Settings → Gemini → *Direct (DEV only)* stores a key encrypted with Android Keystore on the phone. An APK holding a key can be extracted — never share that APK.

## Android permissions
INTERNET, RECORD_AUDIO (voice), SET_ALARM (timer). App list uses a `<queries>` launcher filter (no QUERY_ALL_PACKAGES). Accessibility is enabled by you in Android Settings → Accessibility → Mahi AI. The service ignores all events, doesn't read password fields, uses labels (never coordinates), and acts only when you **arm** it and approve each action. Big red STOP disarms it. Google Play has strict Accessibility policy — this is fine for sideloading, not guaranteed for Play.

## Build on Termux (honest version)
Native Android builds on a phone are hard. Main blockers: Google's `aapt2` is x86 only, and you need an ARM Android SDK.
```bash
pkg update && pkg upgrade
pkg install openjdk-17 gradle aapt2 git unzip wget
java -version                      # must be 17
# 1) Make the Gradle wrapper (Termux's gradle may be too new for this project, so only use it to create the wrapper)
mkdir -p ~/w && cd ~/w && gradle wrapper --gradle-version 8.7
cp gradlew gradlew.bat ~/MahiAI/ && cp gradle/wrapper/gradle-wrapper.jar ~/MahiAI/gradle/wrapper/
cd ~/MahiAI && chmod +x gradlew
# 2) Android SDK: you need platforms;android-34 and build-tools;34.0.0 built for aarch64 Termux
#    (community builds exist, e.g. Lzhiyong/termux-ndk releases — I haven't verified current links).
echo "sdk.dir=$HOME/android-sdk" > local.properties
# 3) Use Termux's aapt2: uncomment the aapt2 line in gradle.properties (path /data/data/com.termux/files/usr/bin/aapt2)
./gradlew --version
./gradlew tasks
./gradlew testDebugUnitTest
./gradlew assembleDebug
```
APK: `app/build/outputs/apk/debug/app-debug.apk` → `cp` it to `~/storage/downloads/` (run `termux-setup-storage` first) and tap to install (allow "install unknown apps").
**Fallback (recommended if Termux fights you): AndroidIDE** — open this folder as a project; it ships ARM tooling. Or build on any PC with Android Studio (Hedgehog or newer) / JDK 17.
Common errors: `aapt2 ... Syntax error` → wrong/x86 aapt2, set the override; `SDK location not found` → local.properties; `Unsupported class file`/Gradle mismatch → JDK 17 + wrapper 8.7; out-of-memory → lower `org.gradle.jvmargs`.

## Using it
Settings → Gemini → set Backend URL + token → Save → Test connection. Then chat or tap Talk. Bangla voice needs the Bangla speech/TTS engine installed on the phone (Settings → TTS shortcut included).
Example commands: `open YouTube`, `play relaxing music`, `flashlight on`, `timer 5 minutes`, `ক্রোম খোলো`, `scroll down`.

## Known limitations
Unbuilt/untested code · turn-based voice · no Firebase sign-in UI · no swipe gestures/sequences · no syntax highlighting · single-instance Firestore quota is simple · chat history sent to Gemini is the last 20 messages.
