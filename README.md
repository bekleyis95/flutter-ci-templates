# flutter-ci-templates

Reusable GitHub Actions workflows shared across Deniz's Flutter repos
(`BaconChallenge-Flutter`, `HarmonicEngine`, `TrueCompass-Flutter`,
`truecompass-flutter-legacy`), so CI config lives in one place instead of
being duplicated per repo.

## Why GitHub Actions, not Codemagic

Decided: GitHub Actions. Keeps CI in the same place as the code/PRs, no
third-party service/billing relationship to manage for passion projects.

## Runner cost split (important, don't undo)

macOS GitHub Actions runners cost ~10x Linux minutes. Deniz has stated
repeatedly across projects that minimizing cost is a real standing
preference for these passion projects, not a one-off call. So:

- **Linux runners**: everything that doesn't need a Mac —
  `flutter pub get`, `flutter analyze`, `flutter test`, Android builds.
  `flutter-ci.yml` (analyze/test) runs on every push/PR;
  `flutter-android-distribute.yml` (build + Firebase App Distribution)
  is on-demand/tag-triggered only — see below, same "don't run heavy
  jobs on every push" principle even though it's Linux, since it's still
  extra minutes and an actual distribution to a real device each time.
- **macOS runners**: reserved for iOS builds, triggered on-demand or on a
  tagged release — never on every push/PR. Two tiers: `flutter-ios-build.yml`
  (unsigned build sanity check — does the Xcode/CocoaPods side even
  compile) and `flutter-ios-distribute.yml` (signed build + TestFlight
  upload) — see below for both.

## Workflows

### `flutter-ci.yml` — Linux checks (`workflow_call`)

Checkout → pin Flutter SDK → `flutter pub get` → `flutter analyze` →
`flutter test`, on `ubuntu-latest`.

Inputs:

| Input | Required | Default | Notes |
|---|---|---|---|
| `flutter-version` | yes | — | Exact Flutter version (e.g. `3.44.6`). Pick a release whose bundled Dart SDK satisfies the consuming repo's `pubspec.yaml` → `environment.sdk` constraint. Don't reuse another repo's pin without checking — these repos are on different Dart constraints. |
| `working-directory` | no | `.` | Directory containing that repo's `pubspec.yaml`. `HarmonicEngine`'s Flutter package root is `harmonic_engine/`, not the repo root — pass that explicitly. |
| `flutter-channel` | no | `stable` | |

Example caller (`.github/workflows/ci.yml` in the consuming repo):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  flutter-ci:
    uses: bekleyis95/flutter-ci-templates/.github/workflows/flutter-ci.yml@main
    with:
      flutter-version: "3.44.6"   # match your repo's own Dart constraint, don't copy blindly
      # working-directory: "harmonic_engine"   # only needed if pubspec.yaml isn't at repo root
```

Known-good pins, as of 2026-07-27 (verify against the target repo's own
`pubspec.yaml` before reusing — these drift as repos bump their Dart
constraint):

| Repo | `environment.sdk` (Dart) | Matching Flutter version |
|---|---|---|
| `BaconChallenge-Flutter` | `^3.12.2` | `3.44.6` (bundles Dart 3.12.2) |
| `HarmonicEngine` (`harmonic_engine/`) | `^3.8.1` | `3.32.1` (bundles Dart 3.8.1) |
| `TrueCompass-Flutter` (`true_compass/`) | `^3.5.2` | not yet resolved to an exact Flutter patch — do this when wiring that repo, don't guess |

### `flutter-android-distribute.yml` — Android build + Firebase App Distribution (`workflow_call`)

Checkout → pin Flutter SDK → `flutter pub get` → `flutter build apk --release`
→ upload to Firebase App Distribution, on `ubuntu-latest`. No Play Console,
no store account, no $25 fee, no review process — Firebase App Distribution
only needs a Firebase project already wired to the Android app (all four
Flutter repos already have one for Auth/Firestore). Play Store listing is a
separate, later, out-of-scope decision.

Inputs:

| Input | Required | Default | Notes |
|---|---|---|---|
| `flutter-version` | yes | — | Same pin rules as `flutter-ci.yml`. |
| `working-directory` | no | `.` | Same as `flutter-ci.yml`. |
| `flutter-channel` | no | `stable` | |
| `firebase-app-id` | yes | — | The Firebase Android app's `mobilesdk_app_id` (from `google-services.json`). Not a secret — identifies the app, doesn't grant access. |
| `testers` | no | `""` | Comma-separated tester emails. Use this and/or `testers-groups`. |
| `testers-groups` | no | `""` | Comma-separated Firebase App Distribution testers group aliases. |
| `release-notes` | no | `"Automated build via GitHub Actions."` | |

Secrets:

| Secret | Required | Notes |
|---|---|---|
| `firebase_service_account` | yes | JSON key **content** (not a file path) for a service account with the "Firebase App Distribution Admin" role in the target Firebase project. Underscored, not hyphenated — GitHub repo secret names can't contain hyphens, so this has to line up with a real repo secret name. Set as a GitHub Actions secret on the *consuming* repo (reusable-workflow secrets aren't inherited across repos automatically — the caller must pass them, e.g. via `secrets: inherit`). |

Getting a service account key (one-time, per Firebase project, done by
whoever owns the Firebase console — this can't be scripted from a CI
session):

1. Firebase console → Project settings → Service accounts → "Generate new
   private key" (or reuse the existing Admin SDK service account if it
   already has, or is granted, the App Distribution Admin role via
   Google Cloud IAM).
2. Add the downloaded JSON's full content as a GitHub Actions secret named
   `FIREBASE_SERVICE_ACCOUNT` on the consuming repo (Settings → Secrets and
   variables → Actions → New repository secret).

Example caller, on-demand only (`.github/workflows/distribute-android.yml`
in the consuming repo):

```yaml
name: Distribute Android build

on:
  workflow_dispatch:   # manual "Run workflow" button
  # push:
  #   tags: ["v*"]     # optional: also trigger on a tagged release

jobs:
  distribute:
    uses: bekleyis95/flutter-ci-templates/.github/workflows/flutter-android-distribute.yml@main
    with:
      flutter-version: "3.44.6"
      firebase-app-id: "1:255573966927:android:921b0bd52f899a3ba8aa3a"
      testers: "someone@example.com"
    secrets: inherit
```

### `flutter-ios-build.yml` — iOS build sanity check, unsigned (`workflow_call`)

Checkout → pin Flutter SDK → `flutter pub get` → `flutter build ios --release
--no-codesign`, on `macos-latest`. Deliberately answers one narrow question —
does the Xcode/CocoaPods side of the repo actually build right now — without
needing any of the Apple Developer Program membership, code-signing
certs/profiles, or App Store Connect setup the real signed/TestFlight
pipeline (below) needs. Produces no distributable artifact; nothing is
uploaded anywhere.

Inputs:

| Input | Required | Default | Notes |
|---|---|---|---|
| `flutter-version` | yes | — | Same pin rules as `flutter-ci.yml`. |
| `working-directory` | no | `.` | Same as `flutter-ci.yml`. |
| `flutter-channel` | no | `stable` | |

Example caller, on-demand only (`.github/workflows/build-ios.yml` in the
consuming repo):

```yaml
name: iOS build sanity check

on:
  workflow_dispatch:   # manual "Run workflow" button

jobs:
  build-ios:
    uses: bekleyis95/flutter-ci-templates/.github/workflows/flutter-ios-build.yml@main
    with:
      flutter-version: "3.44.6"
```

### `flutter-ios-distribute.yml` — iOS signed build + TestFlight upload (`workflow_call`)

Checkout → pin Flutter SDK → `flutter pub get` → `fastlane match appstore
--readonly` (decrypts an already-generated cert/profile from a private git
repo, no Apple login) → `flutter build ipa --release` (signed archive) →
`fastlane pilot upload` (TestFlight), on `macos-latest`. Never needs an
interactive Apple ID/password/2FA prompt in CI — see the doc comment at the
top of the workflow file for why.

Prerequisite, one-time, done by a human on their own Mac (not scriptable):
generate the cert + provisioning profile via `fastlane match appstore`
(stored encrypted in a dedicated private git repo), and an App Store
Connect API key (`.p8` file) from App Store Connect → Users and Access →
Integrations. The consuming repo's `ios/Runner.xcodeproj` must also have
its **Release** configuration set to `CODE_SIGN_STYLE = Manual` with
`DEVELOPMENT_TEAM`/`PROVISIONING_PROFILE_SPECIFIER` matching what `match`
produced — this workflow builds using whatever the Xcode project says, it
doesn't override signing style itself.

Inputs:

| Input | Required | Default | Notes |
|---|---|---|---|
| `flutter-version` | yes | — | Same pin rules as `flutter-ci.yml`. |
| `working-directory` | no | `.` | Same as `flutter-ci.yml`. |
| `flutter-channel` | no | `stable` | |
| `app-identifier` | yes | — | iOS bundle ID (e.g. `com.truecompass.trueCompass`). |
| `team-id` | yes | — | Apple Developer Team ID (e.g. `4YXW55JLH2`). |
| `match-git-url` | yes | — | SSH URL of the private certs repo. Not a secret itself — access control is the deploy key + match password below. |
| `provisioning-profile-specifier` | no | `match AppStore <app-identifier>` | Only set if the profile was renamed from match's default. |

Secrets:

| Secret | Required | Notes |
|---|---|---|
| `match_git_private_key` | yes | SSH private key — a **dedicated deploy key** (read-only) on the certs repo, not a personal key. |
| `match_password` | yes | The passphrase set when `fastlane match` first ran. |
| `app_store_connect_key_id` / `app_store_connect_issuer_id` / `app_store_connect_key_content` | yes | App Store Connect API key — `key_content` is the full `.p8` file contents including BEGIN/END lines. |

Example caller, on-demand only (`.github/workflows/distribute-ios.yml` in
the consuming repo):

```yaml
name: Distribute iOS build to TestFlight

on:
  workflow_dispatch:

jobs:
  distribute:
    uses: bekleyis95/flutter-ci-templates/.github/workflows/flutter-ios-distribute.yml@main
    with:
      flutter-version: "3.44.6"
      app-identifier: "com.truecompass.trueCompass"
      team-id: "4YXW55JLH2"
      match-git-url: "git@github.com:bekleyis95/truecompass-certificates.git"
    secrets: inherit
```

## Adopting this in a new repo

1. Add `.github/workflows/ci.yml` calling `flutter-ci.yml` (see example
   above), with your repo's own `flutter-version` and `working-directory`.
2. Push and confirm the run goes green in the Actions tab.
3. If it's the first repo other than the pilot (`BaconChallenge-Flutter`)
   to adopt this, update this README's pin table and the
   `flutter-ci-templates` board with what you verified.
