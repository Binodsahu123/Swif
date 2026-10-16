# Swift Browser — Developer Tools Black-Screen Fix Report

## What was observed

The supplied recording shows the Swift DevTools Compose toolbar rendering, but the Chromium DevTools frontend viewport underneath remains solid dark/black for the entire recording. The app is already receiving a frontend URL, so the failure is after target resolution and before the frontend finishes bootstrapping.

## Changes made

1. **Allow-list the actual CDP WebSocket request.** `DeveloperToolsPanel` previously rejected every resource scheme other than HTTP(S), data, blob and about. The frontend's connection URL is `ws://127.0.0.1:<port>/devtools/page/<id>?swiftToken=...`; some Android WebView versions route WebSocket handshakes through `shouldInterceptRequest`. If the Android WebView routed that request through this callback, the old handler returned a synthetic 403 for the exact CDP connection the frontend needs. The recording does not expose Logcat, so this is the strongest code-level cause found, not a device-confirmed root cause. The new handler permits only the loopback WebSocket path, port, target path and per-process token encoded in `ws=`; any other WebSocket remains blocked.
2. **Encode the frontend `ws` query correctly.** `DeveloperToolsEngine` now builds `devtools_app.html` using `Uri.Builder.appendQueryParameter`, treating the whole WebSocket endpoint (including its auth query) as one value. DevTools frontend reads it through `URLSearchParams`, which decodes the endpoint back to the expected WebSocket URL.
3. **Expose startup errors instead of silently showing black.** The DevTools WebView logs JavaScript console messages, main-frame navigation errors and HTTP errors. Initial JavaScript/main-frame failures are shown in a small diagnostic overlay in the panel. Status text now distinguishes loading from a loaded frontend instead of declaring “Connected” as soon as a URL has been created.
4. **Keep Swift as the sole browser/tab owner.** `BrowserScreen`, `browser-engine`, `tab-engine` and the existing site WebView remain authoritative. No upstream `MainActivity`, `TabManager` or browser navigation system was imported.
5. **Keep upstream frontend files unchanged.** The imported `devtools_frontend` tree is not modified by this fix.

## Verification performed

- ZIP integrity of the input archives passed before modification.
- Full frontend tree compared by SHA-256 against the supplied `developer-tools-main.zip`.
- Gradle include/dependency links checked statically.
- Kotlin parser check performed with local `kotlinc`; no syntax/“expecting” diagnostics were produced. Android and Compose types could not be resolved by that standalone parser check.
- The final archive is ZIP-tested after creation.

## Verification not possible here

No Android SDK, Gradle distribution or project Gradle wrapper is present in this workspace, and there is no attached Android device. Therefore a full `:app:assembleDebug` and on-device CDP test have not been run. The new visible diagnostics are intended to make any remaining device-only WebView/CSP/CDP compatibility issue explicit rather than leave a silent black panel.
