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
  compile, no Apple Developer Program/signing/App Store Connect setup
  needed) exists now; the signed/TestFlight-upload workflow is still not
  built — see "iOS signing / TestFlight" below.

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

### iOS signing / TestFlight — scoped, not built

Deliberately not implemented yet. Needs, on the macOS-runner side:

- Apple Developer Program membership (approved/$99 paid — verify still
  active before starting this)
- Code signing: certificate + provisioning profile, most likely via
  `fastlane match` rather than hand-rolling cert generation in CI
- App Store Connect API key for TestFlight upload
- All of the above stored as GitHub Actions **secrets** on the consuming
  repo, never committed
- Trigger: on-demand (`workflow_dispatch`) or on a tagged release — not
  every push/PR, to keep macOS-minute usage low

This involves real Apple credentials and manual/2FA steps that can't be
fully scripted from here. See the `flutter-ci-templates` board
(`~/workspace/danny_b_project_management/boards/flutter-ci-templates.md`)
for the open decisions blocking this.

## Adopting this in a new repo

1. Add `.github/workflows/ci.yml` calling `flutter-ci.yml` (see example
   above), with your repo's own `flutter-version` and `working-directory`.
2. Push and confirm the run goes green in the Actions tab.
3. If it's the first repo other than the pilot (`BaconChallenge-Flutter`)
   to adopt this, update this README's pin table and the
   `flutter-ci-templates` board with what you verified.
