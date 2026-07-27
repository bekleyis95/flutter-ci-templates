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
  This is `flutter-ci.yml` below, called on every push/PR.
- **macOS runners**: reserved for **signed iOS builds only**, triggered
  on-demand or on a tagged release — never on every push/PR. (Not built
  yet — see "iOS signing / TestFlight" below.)

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
