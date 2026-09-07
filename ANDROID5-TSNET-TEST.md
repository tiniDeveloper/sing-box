# Android 5 Tailscale test build

Based on sing-box v1.13.19 and its original Android client commit. The
`third_party/tailscale` submodule pins the patched SagerNet Tailscale fork;
`go.mod` uses that checkout, not the unpatched module-cache copy.

The patch tolerates a failed `os.Executable()` on Android. The resulting
`tsnet` string is used only for default hostname/directory selection. Other
platforms retain their existing error handling. No SELinux changes are needed.

## Build

The **Android 5 Tailscale test APK** workflow runs on pushes to
`codex/android5-tsnet-fix` and can also be started manually. Fork Actions may
need to be enabled in GitHub's Actions tab first.

It builds only the ARMv7 API 21 libbox library and the `otherLegacyDebug` APK.
Download the `SFA-1.13.19-android5-tsnet-test-armv7` artifact after a successful
run. It includes the APK, SHA-256 checksum and source commit IDs.

The debug package is `io.nekohasekai.sfa.android5test`, separate from the
official app. It uses a CI-generated Android debug signing key, so subsequent
CI builds may require reinstalling the test app. Export its profile first.
This is a diagnostic build, not a production release or automatic updater.

## Device acceptance test

1. Keep the official app installed; export its profile privately.
2. Install the test APK and import the profile. Never commit authentication
   keys, login URLs, Tailscale state or exported profiles to GitHub.
3. Stop the official VPN before starting the test app.
4. Confirm that startup passes `readlink /proc/self/exe` and reaches login.
5. Confirm the node appears in the tailnet, then test a known TCP service by
   Tailscale IP through the configured sing-box route.
6. Restart the app and confirm that the node identity persists.

Passing the APK build does not prove Android 5 runtime compatibility. Record
any subsequent startup error before changing DNS or routing settings.

## Verified on ZTE Blade L3 (2026-09-07)

The ARMv7 APK from workflow run 34049226802 installed successfully on Android
5.0.2 (API 21), kernel 3.10.54. The original `readlink /proc/self/exe`
failure no longer occurs. No root access or SELinux changes were used.

The next error was `x509: certificate signed by unknown authority` during
control-key retrieval. Adding the following top-level configuration field
resolved it using sing-box's bundled trust store, with TLS verification intact:

```json
"certificate": {
  "store": "mozilla"
}
```

The endpoint then reached `Starting -> Running`, received successful control
map updates and established a DERP connection. After stopping and starting the
service again, it loaded its saved state and reached `Running` again. SELinux
remained Enforcing throughout.

The tested profile only contains a Tailscale endpoint. Android app traffic is
not automatically routed through it: a TUN/proxy inbound and suitable routing
must be configured for that use case. A connection to a specific tailnet
application service has not yet been verified.
