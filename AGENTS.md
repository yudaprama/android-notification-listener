# AGENTS.md — android-notification-listener

Standalone Android app (Kotlin + Compose, Gradle 8.11.1 / AGP via version catalog). Lives in its own git repo: `github.com/yudaprama/android-notification-listener`.

## Build

```sh
./gradlew assembleDebug          # debug build
./gradlew assembleRelease        # release build (signing local only — see below)
```

JDK 17. Android SDK with `compileSdk 35`; Gradle auto-installs missing SDK components where licenses are accepted (GitHub runners ship them pre-accepted).

`gradlew` may lack the executable bit in a fresh clone — `chmod +x gradlew` once (the CI workflow does this itself).

## Release (GitHub Actions)

Workflow: `.github/workflows/release-apk.yml`.

- **Trigger**: every push to `main`.
- Steps: JDK 17 (temurin) → Gradle cache (`gradle/actions/setup-gradle@v4`) → decode signing keystore from secrets → `assembleRelease` signed via `-Pandroid.injected.signing.*` Gradle properties → upload APK artifact + attach to an auto-created GitHub release.
- **Release tag**: `v0.1.<run_number>` — auto-increments on every push (no manual tagging).
- **Output APKs** (assets on the release): `app-release.apk` and a copy named `android-notification-listener-<commit-sha>.apk` (identical bytes). Both are **signed and installable on Android 10+**.
- **No `android-actions/setup-android` step**: the `ubuntu-latest` runner ships the Android SDK with licenses pre-accepted; that action fails on its own `sdkmanager` run here.
- Deprecation annotations in the run log (Node 20 actions, `setup-java` v4) are non-blocking.

### Signing secrets (repo-level GitHub Actions secrets)

| Secret | Value |
|---|---|
| `KEYSTORE_BASE64` | base64 of the keystore file |
| `KEYSTORE_PASSWORD` | keystore password |
| `KEY_ALIAS` | `kawai-release` |
| `KEY_PASSWORD` | key password (same as keystore password) |

**Keystore is the app's identity.** The release APK must always be signed with the SAME key — a different key means users must uninstall before updating. Never lose it:

- Local master copy: `~/kawai-release.keystore` + password in `~/kawai-release.keystore.password` (0600).
- Backup both to a password manager / secure cloud. If lost, there is NO recovery — the app must be reinstalled from scratch on every device.

To rotate/regenerate: `keytool -genkeypair -v -keystore release.keystore -alias kawai-release -keyalg RSA -keysize 2048 -validity 10000`, then re-set the 4 secrets via `gh secret set`.

## Landmines

- **Unsigned release APKs do not install on Android 10+** — always install the signed `app-release.apk` from a release after v0.1.4; older releases' `app-release-unsigned.apk` will fail with "App not installed".
- The workflow uses `gradle/actions/setup-gradle@v4` caching; a change to `gradle-wrapper.properties` (distribution version) is picked up automatically.
