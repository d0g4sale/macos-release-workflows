# macos-release-workflows

Reusable GitHub Actions workflows for building, testing, signing, notarizing and
releasing macOS apps built with Xcode.

| Workflow | What it does |
|---|---|
| [`ci.yml`](.github/workflows/ci.yml) | Unsigned build + unit tests. No secrets. |
| [`release.yml`](.github/workflows/release.yml) | Computes the version, runs unit tests, Developer ID–signs, notarizes and staples the app and DMG, and publishes a GitHub Release with `<Name>-X.Y.Z.dmg`, `<Name>-X.Y.Z.zip` and `SHA256SUMS.txt`. |

Pin to the major tag (`@v1`). Breaking changes get a new major tag.

## Using it in an app repo

### 1. `.github/build.env`

Per-app settings, read from the **calling** repo. Plain `KEY=value` lines, no
quotes (values may contain spaces). Whole-line `#` comments and blank lines
are ignored; trailing comments are **not** — they'd become part of the value.

```sh
# DMG volume name and release title
APP_DISPLAY_NAME=My App
PROJECT=My App.xcodeproj
SCHEME=My App
# Unit-test bundle run by CI and as the release gate
UNIT_TEST_TARGET=My AppTests
# Asset slug: My-App-1.2.3.dmg
ARTIFACT_NAME=My-App
# Developer ID team
TEAM_ID=ABCDE12345
# true: run `xcodegen generate` before building
XCODEGEN=false
```

Optional: `.github/assets/dmg_background.png` is used as the DMG background.
The app's `AppIcon.icns`, if present, becomes the volume icon.

### 2. `.github/workflows/ci.yml`

```yaml
name: CI
on:
  push:
    branches: ["main"]
  pull_request:
  workflow_dispatch: {}
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  test:
    uses: d0g4sale/macos-release-workflows/.github/workflows/ci.yml@v1
```

### 3. `.github/workflows/release.yml`

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      bump:
        description: "Version bump (auto reads Conventional Commits since the last tag)"
        type: choice
        options: [auto, patch, minor, major]
        default: auto
      prerelease:
        description: "Mark the release as a pre-release"
        type: boolean
        default: false
  push:
    tags: ["v*"]
permissions:
  contents: write
concurrency:
  group: release
  cancel-in-progress: false
jobs:
  release:
    uses: d0g4sale/macos-release-workflows/.github/workflows/release.yml@v1
    with:
      bump: ${{ inputs.bump || 'auto' }}
      prerelease: ${{ inputs.prerelease == true }}
    secrets:
      BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
      P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
      KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
      NOTARY_KEY_BASE64: ${{ secrets.NOTARY_KEY_BASE64 }}
      NOTARY_KEY_ID: ${{ secrets.NOTARY_KEY_ID }}
      NOTARY_ISSUER_ID: ${{ secrets.NOTARY_ISSUER_ID }}
```

Secrets are passed explicitly rather than with `secrets: inherit`, so the
same caller works from any owner. (Inside the d0g4sale org, `inherit` would
also work.)

### 4. Repository secrets

| Secret | Contents |
|---|---|
| `BUILD_CERTIFICATE_BASE64` | base64 of the Developer ID Application `.p12` |
| `P12_PASSWORD` | password for that `.p12` |
| `KEYCHAIN_PASSWORD` | any random string (throwaway CI keychain) |
| `NOTARY_KEY_BASE64` | base64 of the App Store Connect API key `.p8` |
| `NOTARY_KEY_ID` | that key's ID |
| `NOTARY_ISSUER_ID` | App Store Connect issuer ID |

Inside d0g4sale these can be **organization secrets** shared with the app
repos (Settings → Secrets and variables → Actions), so a certificate or notary
key rotation is a single update. A repository secret with the same name
overrides the organization one.

A release fails up front with the names of any missing secrets.

## Releasing an app

- **Manual (recommended):** Actions → Release → Run workflow, pick a bump.
  The tag is created only after everything is tested, signed and notarized.
- **Tag push:** `git tag v1.2.3 && git push origin v1.2.3` releases that tag.

The bundle's `CFBundleShortVersionString` is stamped from the tag and
`CFBundleVersion` from the caller's workflow run number.

## Changing these workflows

Test on a branch by pointing one app at `@<branch>`, then merge, tag
`vX.Y.Z`, and move the major tag:

```sh
git tag v1.0.1 && git tag -f v1 && git push origin v1.0.1 && git push -f origin v1
```
