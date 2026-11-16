# microG hybrid passkeys → Android Credential Manager

An unofficial patch for [microG GmsCore](https://github.com/microg/GmsCore) that hands **cross-device passkey
requests** (the `FIDO:/…` QR code a PC browser shows) to Android's **Credential Manager**, so your chosen
password manager can answer them, not only microG's own built-in passkey store.

> Not affiliated with or endorsed by the microG project. Please report problems with this patch here,
> **not** to microG's issue tracker.

## The problem
Scanning a cross-device passkey QR (`FIDO:/…`) on a phone running microG (`GmsCore` 0.3.14+) works at the
transport level (BLE + WebSocket tunnel to the PC), but the decrypted CTAP2 `makeCredential` /
`getAssertion` request only ever reaches microG's own passkey vault. Third-party providers configured under
*Passwords, passkeys & accounts* (Bitwarden, Proton Pass, KeePassDX, …) never get asked, even though they work
fine for passkeys in the phone's own browser.

## What this repo is
Not a fork. It holds **one patch** against official `microg/GmsCore` plus a GitHub Actions workflow that
re-applies, tests and builds it for every new upstream release. All builds happen on GitHub Actions.

```
patches/0001-hybrid-delegate-to-credential-manager.patch   the change (+ its unit test)
.github/workflows/patch-build.yml                          polls upstream, applies, tests, builds, signs, publishes
README.md                                                  this file
```

Ready-made APKs are on the [Releases](../../releases) page, one per upstream microG release (`gmscore-<tag>`).
Only **arm64-v8a, non-Huawei** devices are built: the same variant as official microG's
`com.google.android.gms-….apk` (mapbox maps, default target), with other CPU architectures stripped.
Only the GmsCore APK is built (no companion / Play Store app). Network location offers the same online sources
as official microG: **Positon** (suggested), BeaconDB and a custom URL.

## What the patch does
Upstream flow: `HybridAuthenticateActivity` → `AuthenticatorActivity` (microG vault).

Patched flow: `HybridAuthenticateActivity` → `CredentialManagerBridge` (new file) →
`androidx.credentials.CredentialManager` → chosen provider. The CTAP2 request is converted to WebAuthn JSON,
the PC's `clientDataHash` and `https://<rpId>` origin are forwarded, and the provider's response is converted
back to a CTAP2 response (`attestationObject` / `authenticatorData` + `signature`) for the tunnel.

- **Fallback:** on Android < 14 (API 34), or any error except user cancellation, the original microG vault flow
  runs, so behaviour never gets worse than upstream. If the user cancels the Credential Manager sheet, the
  request fails (the PC shows an error) instead of opening microG's own UI.
- **Not supported:** WebAuthn extensions (PRF / hmac-secret, largeBlob, credProps, …) are dropped when going
  through Credential Manager; the PC receives a response without extension outputs.
- **Tests:** `CredentialManagerBridgeTest.kt` checks the JSON / base64url conversion on the JVM; CI runs it
  before building.
- **Dependencies:** `androidx.credentials` is added to `play-services-fido/core` at the version upstream already
  pins (1.2.0). Don't bump it to 1.5.0+: that needs minSdk 21, and microG supports 19.

## What has to be true for the hand-off to work
1. **Android 14+** (API 34), where the system Credential Manager exists.
2. **Permission:** passing the PC's origin needs `CREDENTIAL_MANAGER_SET_ORIGIN`. It is a *normal* permission
   (AOSP Android 14–16), so the patch just declares it and Android grants it at install. No root, no system app.
3. **Your password manager must trust microG.** Providers only accept a caller-supplied origin from apps on
   their own trusted list (normally browsers). Some let you add apps yourself: add `com.google.android.gms` with
   this repo's signing certificate (SHA-256 `20:41:F0:1C:D7:15:75:99:A6:64:F3:93:70:BE:22:62:BF:B9:91:8E:5B:D9:A5:6B:6C:3E:6C:D4:9F:D8:97:D2`).
   Providers that only trust their built-in list will refuse.
4. **The provider must sign the PC's `clientDataHash`** it is given (as it does for browsers), not its own.

If any of these fails, the request falls back to microG's own passkey store, same as upstream. Which providers
meet 3 and 4 is untested; reports welcome.

## Installing the APKs
These are **release** builds (unminified, like upstream's) of upstream microG with the patch, signed with this
repo's key, **not** microG's. Like any self-built microG they need the usual microG setup (signature spoofing
support in the ROM) and cannot update an official microG install in place: uninstall the official one first.
Later releases from this repo update each other normally.

### Does signature spoofing still work?
**Yes, if your ROM supports it.** microG doesn't use its own signing key for spoofing. The spoofed Google
signature is declared inside the APK (the upstream `fake-signature` module: `FAKE_PACKAGE_SIGNATURE` permission +
`fake-signature` metadata), and that module is part of this build unchanged. Re-signing with this repo's key
doesn't touch it.

It will **not** work if:
- the ROM has no signature spoofing support, or you haven't granted the "fake package signature" permission;
- the ROM only allows spoofing for microG signed with the **official** microG key (some microG-bundled ROMs do);
- microG is preinstalled as a system app by the ROM: Android won't let a differently signed APK replace it.

Check in *microG Settings → Self-Check → Signature spoofing support*: "System has signature spoofing support",
"System grants signature spoofing permission" and "System spoofs signature" must all be ticked.
Spoofing is unrelated to Play Integrity: an unlocked bootloader fails integrity checks either way.

## How the automation works
GitHub cannot trigger a workflow on another repo's release, so the workflow polls every 6 hours, compares the
latest `microg/GmsCore` release tag with this repo's `gmscore-<tag>` releases, and builds only new ones.
Manual run: *Actions → Patch & build microG on new release → Run workflow* (optional tag).
Running it for a tag that's already released rebuilds it and replaces the APK (use this after a patch fix).
Each run also re-enables the schedule, since GitHub turns off scheduled workflows in public repos after 60 quiet days.

Upstream has no release signing config, so CI signs the `*-release-unsigned.apk` outputs with `zipalign` +
`apksigner` using a fixed key from the `SIGNING_KEYSTORE_B64` secret. The key must never change, or users
can't update without uninstalling (and losing their microG data).

Upstream's code only ships BeaconDB as an online location source; official releases add Positon via
`location.online-sources` in `local.properties`. CI writes the same JSON from the `LOCATION_ONLINE_SOURCES` secret.

If applying, testing, building or publishing fails, one issue `Patch build failed for <tag>` is opened and
scheduled runs skip that tag while it is open. Fix `patches/`, then close the issue (or run manually) to retry.

### Running your own copy
Fork this repo, then create a signing key once and store it as a secret:

```sh
keytool -genkeypair -keystore release.keystore -storepass android -keypass android \
  -alias androiddebugkey -dname "CN=microG patched" -keyalg RSA -keysize 2048 -validity 10000
base64 -w0 release.keystore | gh secret set SIGNING_KEYSTORE_B64
```

The workflow expects store password `android` and alias `androiddebugkey`. Back up the keystore.

Also set `LOCATION_ONLINE_SOURCES` to a JSON list of location sources (see upstream
`play-services-location/core/base/build.gradle` for the format). Use your own Positon API key if you have one.

## Credits
All of microG is the work of the **[microG Project](https://microg.org)**: its founder and lead developer
Marvin Wißfeld ([@mar-v-in](https://github.com/mar-v-in)) and the
[GmsCore contributors](https://github.com/microg/GmsCore/graphs/contributors). They built the hybrid / caBLE
transport, the FIDO2 implementation and everything else this patch relies on. This repo only adds a small bridge
on top. If microG is useful to you, consider supporting the project (see [microg.org](https://microg.org)).

Also built on Android Jetpack's [`androidx.credentials`](https://developer.android.com/jetpack/androidx/releases/credentials)
library.

## License
microG GmsCore is licensed under the [Apache License 2.0](https://github.com/microg/GmsCore/blob/master/LICENSE).
The patch in this repo is a derivative work and is released under the same Apache License 2.0. "microG" is the
microG Project's name; using it here only describes what the patch applies to.
