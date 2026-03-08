# Release Pipeline Guide: Shipping Home Still CLI Tools

> **Audience:** AI agents and human contributors productionizing a new Home Still CLI tool.  
> **Goal:** Cross-platform binary distribution via Homebrew (macOS/Linux), Scoop + WinGet (Windows), and `cargo install` — with a single `git push vX.Y.Z` triggering the entire pipeline automatically.

---

## Overview

Every Home Still CLI tool follows this distribution model:

| Channel | Platform | Command | Effort |
|---|---|---|---|
| `cargo install` | All | `cargo install --git https://github.com/home-still/<tool> <crate>` | Zero (already works) |
| Homebrew tap | macOS + Linux | `brew install home-still/tap/<tool>` | Medium (one-time) |
| Scoop bucket | Windows | `scoop install <tool>` | Low (one-time) |
| WinGet | Windows | `winget install HomeStill.<Tool>` | Medium (one-time) |
| Chocolatey | Windows (enterprise) | `choco install <tool>` | High — add only on demand |

Binary targets to ship on every release:

| Target | Runner | Compiler |
|---|---|---|
| `x86_64-apple-darwin` | `macos-13` | native cargo |
| `aarch64-apple-darwin` | `macos-14` | native cargo |
| `x86_64-unknown-linux-gnu` | `ubuntu-latest` | native cargo |
| `aarch64-unknown-linux-gnu` | `ubuntu-latest` | **cargo-zigbuild** |
| `x86_64-pc-windows-msvc` | `windows-latest` | native cargo |

**Why `cargo-zigbuild` for ARM64 Linux, not `cross`?**  
`cross` (cross-rs) last released in February 2023 and its Docker images have not been updated since 2022. Each invocation pulls a 1–2 GB image with no built-in caching. `cargo-zigbuild` is actively maintained by the rust-cross org, requires no Docker, adds only ~20 seconds of setup, and supports explicit glibc version pinning (`.2.28`) for broad compatibility. It also aligns with the musl compilation already used in the Kubernetes pipeline.

---

## Prerequisites

### 1. Cargo.toml settings

Confirm the binary crate uses these release profile settings (they should already be set workspace-wide):

```toml
[profile.release]
opt-level    = 3
lto          = "fat"
codegen-units = 1
panic        = "abort"
strip        = "symbols"
```

Confirm TLS uses `rustls` with `ring` as the crypto backend — **not** `aws-lc-rs`. The `aws-lc-rs` backend has known cross-compilation failures for `aarch64-linux`. Check your `Cargo.lock` after adding dependencies:

```toml
# If using reqwest:
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
```

### 2. Fix `.gitmodules` (blocker if using `hs-style`)

CI runners cannot clone SSH submodules without keys. Use HTTPS:

```ini
# .gitmodules
[submodule "hs-style"]
    path = hs-style
    url  = https://github.com/home-still/hs-style.git
```

Commit and push this before creating any workflows. Verify with:

```bash
git submodule sync
git submodule update --init --recursive
```

### 3. Required secrets

Configure these in the tool repo's Settings → Secrets and variables → Actions:

| Secret | Purpose | Where to get it |
|---|---|---|
| `TAP_GITHUB_TOKEN` | PAT to push commits to `home-still/homebrew-tap` | GitHub → Settings → Developer settings → PAT (Classic), `repo` scope on the tap repo |
| `SCOOP_GITHUB_TOKEN` | PAT to push commits to `home-still/scoop-bucket` | Same PAT works if it has `repo` scope on the bucket repo |
| `WINGET_TOKEN` | PAT to open PRs against `microsoft/winget-pkgs` | Same PAT, needs `public_repo` scope |

---

## Step 1 — CI workflow (PR and push checks)

File: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always

jobs:
  check:
    name: Check (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - uses: Swatinem/rust-cache@v2

      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets -- -D warnings
      - run: cargo test --workspace
```

**Notes:**
- `submodules: true` is sufficient unless `hs-style` itself has nested submodules, in which case use `submodules: recursive`.
- If `hs-style` is a private repo, pass `token: ${{ secrets.TAP_GITHUB_TOKEN }}` (or a dedicated read-only PAT) to the checkout step.
- Run this on all three OS families to catch platform-specific issues early.

---

## Step 2 — Release workflow (tag-triggered)

File: `.github/workflows/release.yml`

Replace `<TOOL>` with the binary name (e.g., `paper-fetch`) and `<CRATE>` with the crate name (e.g., `paper-fetch-cli`).

```yaml
name: Release

on:
  push:
    tags: ["v*"]

env:
  CARGO_TERM_COLOR: always
  TOOL: <TOOL>           # e.g. paper-fetch
  CRATE: <CRATE>         # e.g. paper-fetch-cli

jobs:
  # ── Phase 1: Build all platform binaries ──────────────────────────────────
  build:
    name: Build ${{ matrix.target }}
    runs-on: ${{ matrix.runner }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - target:  x86_64-apple-darwin
            runner:  macos-13
            archive: tar.gz
          - target:  aarch64-apple-darwin
            runner:  macos-14
            archive: tar.gz
          - target:  x86_64-unknown-linux-gnu
            runner:  ubuntu-latest
            archive: tar.gz
          - target:  aarch64-unknown-linux-gnu
            runner:  ubuntu-latest
            archive: tar.gz
            use_zigbuild: true
          - target:  x86_64-pc-windows-msvc
            runner:  windows-latest
            archive: zip

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.target }}

      - name: Install cargo-zigbuild + zig (ARM64 Linux only)
        if: matrix.use_zigbuild
        run: |
          pip3 install ziglang --break-system-packages
          cargo install cargo-zigbuild

      - name: Add target
        run: rustup target add ${{ matrix.target }}

      - name: Build (zigbuild)
        if: matrix.use_zigbuild
        run: >
          cargo zigbuild --release
          --target ${{ matrix.target }}.2.28
          -p ${{ env.CRATE }}

      - name: Build (native)
        if: "!matrix.use_zigbuild"
        run: >
          cargo build --release
          --target ${{ matrix.target }}
          -p ${{ env.CRATE }}

      - name: Package (Unix)
        if: matrix.archive == 'tar.gz'
        shell: bash
        run: |
          ASSET="${{ env.TOOL }}-${{ github.ref_name }}-${{ matrix.target }}.tar.gz"
          tar -czf "$ASSET" \
            -C "target/${{ matrix.target }}/release" "${{ env.TOOL }}"
          shasum -a 256 "$ASSET" > "$ASSET.sha256"
          echo "ASSET=$ASSET" >> $GITHUB_ENV

      - name: Package (Windows)
        if: matrix.archive == 'zip'
        shell: pwsh
        run: |
          $asset = "${{ env.TOOL }}-${{ github.ref_name }}-${{ matrix.target }}.zip"
          Compress-Archive `
            -Path "target\${{ matrix.target }}\release\${{ env.TOOL }}.exe" `
            -DestinationPath $asset
          $hash = (Get-FileHash $asset -Algorithm SHA256).Hash.ToLower()
          "$hash  $asset" | Out-File -Encoding ascii "$asset.sha256"
          echo "ASSET=$asset" | Out-File -FilePath $env:GITHUB_ENV -Append

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ env.TOOL }}-${{ matrix.target }}
          path: |
            ${{ env.ASSET }}
            ${{ env.ASSET }}.sha256

  # ── Phase 2: Create GitHub Release ────────────────────────────────────────
  release:
    name: Publish GitHub Release
    needs: build
    runs-on: ubuntu-latest
    outputs:
      version: ${{ github.ref_name }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: artifacts
          merge-multiple: true

      - name: Combine checksums
        run: cat artifacts/*.sha256 > checksums-sha256.txt

      - uses: softprops/action-gh-release@v2
        with:
          files: |
            artifacts/*
            checksums-sha256.txt
          generate_release_notes: true
          draft: false
          fail_on_unmatched_files: true

  # ── Phase 3: Update Homebrew tap ──────────────────────────────────────────
  homebrew:
    name: Update Homebrew tap
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: Trigger tap update
        env:
          VERSION: ${{ needs.release.outputs.version }}
        run: |
          gh workflow run update-formula.yml \
            --repo home-still/homebrew-tap \
            --field tool=${{ env.TOOL }} \
            --field version=$VERSION \
            --field repo=home-still/${{ env.TOOL }}
        env:
          GH_TOKEN: ${{ secrets.TAP_GITHUB_TOKEN }}

  # ── Phase 4: Update Scoop bucket ──────────────────────────────────────────
  scoop:
    name: Update Scoop bucket
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: Trigger bucket update
        run: |
          gh workflow run update-manifest.yml \
            --repo home-still/scoop-bucket \
            --field tool=${{ env.TOOL }} \
            --field version=${{ needs.release.outputs.version }}
        env:
          GH_TOKEN: ${{ secrets.SCOOP_GITHUB_TOKEN }}

  # ── Phase 5: Publish to WinGet ────────────────────────────────────────────
  winget:
    name: Submit WinGet manifest
    needs: release
    runs-on: ubuntu-latest
    steps:
      - uses: vedantmgoyal9/winget-releaser@main
        with:
          identifier: HomeStill.${{ env.TOOL }}
          token: ${{ secrets.WINGET_TOKEN }}
```

**Key design decisions:**
- **Two-phase pattern** (artifact upload → single release job) eliminates all race conditions. Never run `softprops/action-gh-release` inside a matrix job.
- Pin to a specific `softprops/action-gh-release` version (not `@v2`) once the pipeline is stable — versions v2.3.0–v2.3.1 had assertion crashes on Ubuntu/macOS.
- `fail_on_unmatched_files: true` catches packaging bugs immediately.
- The `glibc .2.28` suffix on the zigbuild target produces binaries compatible with RHEL 8, Ubuntu 18.04, and newer — a safe baseline for a CLI tool.

---

## Step 3 — Homebrew tap (`home-still/homebrew-tap`)

Create the repo `home-still/homebrew-tap` with this structure:

```
homebrew-tap/
├── Formula/
│   └── <tool>.rb.template
└── .github/
    └── workflows/
        └── update-formula.yml
```

### Formula template

File: `Formula/<tool>.rb.template`

```ruby
class <ToolPascalCase> < Formula
  desc "<One-line description>"
  homepage "https://github.com/home-still/<tool>"
  version "__VERSION__"
  license "GPL-3.0-only"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/home-still/<tool>/releases/download/v__VERSION__/<tool>-v__VERSION__-aarch64-apple-darwin.tar.gz"
      sha256 "__SHA256_MACOS_ARM64__"
    else
      url "https://github.com/home-still/<tool>/releases/download/v__VERSION__/<tool>-v__VERSION__-x86_64-apple-darwin.tar.gz"
      sha256 "__SHA256_MACOS_X86_64__"
    end
  end

  on_linux do
    if Hardware::CPU.arm?
      url "https://github.com/home-still/<tool>/releases/download/v__VERSION__/<tool>-v__VERSION__-aarch64-unknown-linux-gnu.tar.gz"
      sha256 "__SHA256_LINUX_ARM64__"
    else
      url "https://github.com/home-still/<tool>/releases/download/v__VERSION__/<tool>-v__VERSION__-x86_64-unknown-linux-gnu.tar.gz"
      sha256 "__SHA256_LINUX_X86_64__"
    end
  end

  def install
    bin.install "<tool>"
  end

  test do
    system "#{bin}/<tool>", "--version"
  end
end
```

### Tap update workflow

File: `.github/workflows/update-formula.yml`

```yaml
name: Update Formula

on:
  workflow_dispatch:
    inputs:
      tool:
        required: true
      version:
        required: true
      repo:
        required: true

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download checksums
        run: |
          gh release download "v${{ inputs.version }}" \
            --repo "${{ inputs.repo }}" \
            --pattern "checksums-sha256.txt" \
            --dir /tmp/checksums
        env:
          GH_TOKEN: ${{ github.token }}

      - name: Extract SHAs and render formula
        run: |
          TOOL="${{ inputs.tool }}"
          VERSION="${{ inputs.version }}"
          CHECKSUMS=/tmp/checksums/checksums-sha256.txt

          sha_macos_arm64=$(grep    "aarch64-apple-darwin.tar.gz"        "$CHECKSUMS" | awk '{print $1}')
          sha_macos_x86=$(grep      "x86_64-apple-darwin.tar.gz"         "$CHECKSUMS" | awk '{print $1}')
          sha_linux_arm64=$(grep    "aarch64-unknown-linux-gnu.tar.gz"   "$CHECKSUMS" | awk '{print $1}')
          sha_linux_x86=$(grep      "x86_64-unknown-linux-gnu.tar.gz"    "$CHECKSUMS" | awk '{print $1}')

          TEMPLATE="Formula/${TOOL}.rb.template"
          OUTPUT="Formula/${TOOL}.rb"

          sed \
            -e "s/__VERSION__/${VERSION}/g" \
            -e "s/__SHA256_MACOS_ARM64__/${sha_macos_arm64}/g" \
            -e "s/__SHA256_MACOS_X86_64__/${sha_macos_x86}/g" \
            -e "s/__SHA256_LINUX_ARM64__/${sha_linux_arm64}/g" \
            -e "s/__SHA256_LINUX_X86_64__/${sha_linux_x86}/g" \
            "$TEMPLATE" > "$OUTPUT"

      - name: Commit and push
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "Formula/${{ inputs.tool }}.rb"
          git commit -m "chore: bump ${{ inputs.tool }} to v${{ inputs.version }}"
          git push
```

**First-time user setup:**
```bash
brew tap home-still/tap
brew install home-still/tap/<tool>
```

---

## Step 4 — Scoop bucket (`home-still/scoop-bucket`)

Scoop is the recommended primary Windows channel: no moderation, no admin rights required, auto-updates built in.

Create the repo `home-still/scoop-bucket`. Use the [`ScoopInstaller/BucketTemplate`](https://github.com/ScoopInstaller/BucketTemplate) as a starting point — it includes CI that validates manifests automatically.

### Manifest template

File: `bucket/<tool>.json`

```json
{
  "version": "__VERSION__",
  "description": "<One-line description>",
  "homepage": "https://github.com/home-still/<tool>",
  "license": "GPL-3.0-only",
  "architecture": {
    "64bit": {
      "url": "https://github.com/home-still/<tool>/releases/download/v__VERSION__/<tool>-v__VERSION__-x86_64-pc-windows-msvc.zip",
      "hash": "__SHA256_WINDOWS_X86_64__"
    }
  },
  "bin": "<tool>.exe",
  "checkver": {
    "github": "https://github.com/home-still/<tool>"
  },
  "autoupdate": {
    "architecture": {
      "64bit": {
        "url": "https://github.com/home-still/<tool>/releases/download/v$version/<tool>-v$version-x86_64-pc-windows-msvc.zip"
      }
    },
    "hash": {
      "url": "$url.sha256"
    }
  }
}
```

The `checkver` + `autoupdate` fields let Scoop's own tooling auto-detect new releases and open PRs against your bucket — zero ongoing maintenance after the first version.

### Bucket update workflow

File: `.github/workflows/update-manifest.yml` (in `home-still/scoop-bucket`)

```yaml
name: Update Manifest

on:
  workflow_dispatch:
    inputs:
      tool:
        required: true
      version:
        required: true

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download Windows checksum
        run: |
          gh release download "v${{ inputs.version }}" \
            --repo "home-still/${{ inputs.tool }}" \
            --pattern "*windows-msvc.zip.sha256" \
            --dir /tmp/chk
        env:
          GH_TOKEN: ${{ github.token }}

      - name: Update manifest
        run: |
          TOOL="${{ inputs.tool }}"
          VERSION="${{ inputs.version }}"
          SHA=$(cat /tmp/chk/*.sha256 | awk '{print $1}')

          jq \
            --arg v  "$VERSION" \
            --arg h  "$SHA" \
            '.version = $v | .architecture["64bit"].hash = $h' \
            "bucket/${TOOL}.json" > /tmp/manifest.json
          mv /tmp/manifest.json "bucket/${TOOL}.json"

      - name: Commit and push
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "bucket/${{ inputs.tool }}.json"
          git commit -m "chore: bump ${{ inputs.tool }} to v${{ inputs.version }}"
          git push
```

**First-time user setup:**
```powershell
scoop bucket add home-still https://github.com/home-still/scoop-bucket
scoop install home-still/<tool>
```

---

## Step 5 — WinGet (`microsoft/winget-pkgs`)

WinGet ships pre-installed on Windows 10/11 and is moderated in hours rather than days. Use the [`vedantmgoyal9/winget-releaser`](https://github.com/vedantmgoyal9/winget-releaser) action (already included in the release workflow above) for automated PR submission after the first manual version.

### First submission (manual, one-time)

1. Install `komac`: `winget install Komac`
2. Generate manifests from the first release URL:
   ```powershell
   komac create --identifier HomeStill.<ToolPascalCase> \
     --version 0.1.0 \
     --urls https://github.com/home-still/<tool>/releases/download/v0.1.0/<tool>-v0.1.0-x86_64-pc-windows-msvc.zip \
     --submit
   ```
3. The tool opens a PR against `microsoft/winget-pkgs`. Review takes 4–24 hours.

### Subsequent versions

Fully automated by the `winget` job in the release workflow. No manual action needed.

---

## Step 6 — Chocolatey (optional, enterprise only)

Add Chocolatey only when enterprise users specifically request it. The moderation queue for new packages takes 3–14 days; updates 1–7 days. The effort-to-reach ratio is poor compared to Scoop + WinGet for a developer-focused CLI.

If you proceed, add these files to the tool repo:

**`choco/<tool>.nuspec`**
```xml
<?xml version="1.0" encoding="utf-8"?>
<package>
  <metadata>
    <id><tool></id>
    <version>__VERSION__</version>
    <title><Tool></title>
    <authors>Home Still</authors>
    <projectUrl>https://github.com/home-still/<tool></projectUrl>
    <licenseUrl>https://www.gnu.org/licenses/gpl-3.0.html</licenseUrl>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <description>__DESCRIPTION__</description>
    <tags><tool> cli academic rust</tags>
  </metadata>
</package>
```

**`choco/tools/chocolateyInstall.ps1`**
```powershell
$ErrorActionPreference = 'Stop'
$packageName = '<tool>'
$url64       = '__URL__'
$checksum64  = '__CHECKSUM__'

$packageArgs = @{
  packageName    = $packageName
  unzipLocation  = "$(Split-Path -parent $MyInvocation.MyCommand.Definition)"
  url64bit       = $url64
  checksum64     = $checksum64
  checksumType64 = 'sha256'
}

Install-ChocolateyZipPackage @packageArgs
```

Add a `chocolatey` job to the release workflow that replaces placeholders and runs `choco pack && choco push`. Requires a `CHOCOLATEY_API_KEY` secret from your chocolatey.org account.

---

## Implementation checklist for a new tool

Work through this in order. Each step has a clear verification gate before proceeding.

### Phase A — Repository basics
- [ ] `.gitmodules` uses HTTPS URLs for all submodules
- [ ] `cargo fmt --all --check` passes locally
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` passes locally
- [ ] `cargo test --workspace` passes locally
- [ ] `Cargo.toml` release profile has `lto = "fat"`, `strip = "symbols"`, `panic = "abort"`
- [ ] TLS stack uses `rustls-tls` (not `native-tls`); `Cargo.lock` shows `ring`, not `aws-lc-rs`

### Phase B — CI
- [ ] Create `.github/workflows/ci.yml`
- [ ] Push to main → CI runs green on all three OS runners
- [ ] Open a test PR → CI blocks on fmt/clippy failures as expected

### Phase C — Release pipeline (builds only)
- [ ] Create `.github/workflows/release.yml` with only the `build` and `release` jobs (comment out `homebrew`, `scoop`, `winget`)
- [ ] Push tag `v0.0.1-rc.1` → verify all five targets build and a GitHub Release is created
- [ ] Confirm each archive extracts a working binary: `./paper-fetch --version`
- [ ] Confirm `checksums-sha256.txt` lists all five targets

### Phase D — Homebrew
- [ ] Create `home-still/homebrew-tap` if it doesn't exist
- [ ] Add `Formula/<tool>.rb.template`
- [ ] Add `.github/workflows/update-formula.yml`
- [ ] Add `TAP_GITHUB_TOKEN` secret to the tool repo
- [ ] Uncomment the `homebrew` job in the release workflow
- [ ] Push tag `v0.0.2-rc.1` → verify `Formula/<tool>.rb` is committed to the tap
- [ ] `brew tap home-still/tap && brew install home-still/tap/<tool>` installs and runs correctly

### Phase E — Windows (Scoop + WinGet)
- [ ] Create `home-still/scoop-bucket` from `ScoopInstaller/BucketTemplate`
- [ ] Add `bucket/<tool>.json` manifest
- [ ] Add `.github/workflows/update-manifest.yml` to the bucket repo
- [ ] Add `SCOOP_GITHUB_TOKEN` secret to the tool repo
- [ ] Uncomment the `scoop` job
- [ ] First WinGet submission: run `komac create` manually, wait for approval
- [ ] Add `WINGET_TOKEN` secret to the tool repo
- [ ] Uncomment the `winget` job
- [ ] Push tag `v0.1.0` → full end-to-end release

### Phase F — Verify install UX
```bash
# macOS/Linux
brew install home-still/tap/<tool>
<tool> --version

# Windows (PowerShell)
scoop bucket add home-still https://github.com/home-still/scoop-bucket
scoop install home-still/<tool>
<tool> --version

winget install HomeStill.<ToolPascalCase>
<tool> --version

# Any platform
cargo install --git https://github.com/home-still/<tool> <crate>
<tool> --version
```

---

## Troubleshooting

**`ring` fails to cross-compile for `aarch64-unknown-linux-gnu`**  
Check your `Cargo.lock` — if `aws-lc-rs` appears as a dependency (introduced by rustls ≥ 0.23 without explicit feature selection), override it:
```toml
[dependencies]
rustls = { version = "0.23", default-features = false, features = ["ring"] }
```

**`cargo-zigbuild` fails with `xcrun` errors**  
This only happens when cross-targeting Darwin from Linux. All Darwin builds run natively on macOS runners — this is not a problem for the build matrix above.

**`softprops/action-gh-release` crashes with `Assertion failed: !closed_`**  
Caused by versions v2.3.0–v2.3.1. Pin to a specific version: `softprops/action-gh-release@v2.5.0` or later.

**Homebrew formula fails with `sha256 mismatch`**  
The `update-formula.yml` workflow reads checksums from `checksums-sha256.txt`. If the release workflow is interrupted and some artifacts are missing from that file, the formula will have wrong hashes. Re-run the release workflow manually via `workflow_dispatch`.

**WinGet PR is rejected with schema validation errors**  
Run `winget validate --manifest <path>` locally before submitting. The `komac` tool produces valid manifests; manual edits to YAML files are the usual cause.

**Submodule checkout fails in CI with authentication errors**  
`hs-style` is a public repo — the default `GITHUB_TOKEN` should be sufficient. If the submodule is ever made private, pass `token: ${{ secrets.TAP_GITHUB_TOKEN }}` (a PAT with read access to the submodule repo) to `actions/checkout`.

---

## Reference: archive naming convention

Archives must follow this exact pattern (the update workflows parse it):

```
<tool>-<tag>-<target>.<ext>
```

Examples:
```
paper-fetch-v0.1.0-x86_64-apple-darwin.tar.gz
paper-fetch-v0.1.0-aarch64-apple-darwin.tar.gz
paper-fetch-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
paper-fetch-v0.1.0-aarch64-unknown-linux-gnu.tar.gz
paper-fetch-v0.1.0-x86_64-pc-windows-msvc.zip
```

Checksums use `.sha256` extension appended to the archive name:
```
paper-fetch-v0.1.0-x86_64-apple-darwin.tar.gz.sha256
```

The combined `checksums-sha256.txt` file uses standard `shasum -a 256` format:
```
a3f2...  paper-fetch-v0.1.0-x86_64-apple-darwin.tar.gz
b7d1...  paper-fetch-v0.1.0-aarch64-apple-darwin.tar.gz
...
```
