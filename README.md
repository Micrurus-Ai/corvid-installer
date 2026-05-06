# corvid-installer

The install pipeline for the [Corvid](https://github.com/Micrurus-Ai/Corvid-lang) programming language. End users install Corvid in one line; this repo is the development workspace where the install machinery was authored.

> **Canonical locations.** The files in this repo are mirrored into the Corvid-lang repo, and that's where users actually fetch them from at runtime. Do not point users at this repo's URLs — point them at Corvid-lang's:
>
> - `install/install.sh` and `install/install.ps1` → `Micrurus-Ai/Corvid-lang/install/`
> - `release.yml` and `publish-managers.yml` → `Micrurus-Ai/Corvid-lang/.github/workflows/`
> - `worker.js` and `wrangler.toml` → `Micrurus-Ai/Corvid-lang/web/`
> - The Homebrew tap → `Micrurus-Ai/homebrew-corvid`
> - The Scoop bucket → `Micrurus-Ai/scoop-corvid`

---

## Table of contents

- [Install Corvid (end users)](#install-corvid-end-users)
- [What gets installed](#what-gets-installed)
- [Architecture](#architecture)
- [Pipeline components](#pipeline-components)
- [Supported targets](#supported-targets)
- [Configuration reference](#configuration-reference)
- [Cutting a release](#cutting-a-release)
- [Operations runbook](#operations-runbook)
- [Local development](#local-development)
- [Uninstall](#uninstall)
- [Design decisions](#design-decisions)
- [License](#license)

Open follow-up tasks (push this repo, narrow PAT scope, deploy Cloudflare, etc.) live in [FOLLOWUPS.md](./FOLLOWUPS.md).

---

## Install Corvid (end users)

Four supported paths. Pick the one that matches how you already install developer tools.

### 1. One-line bootstrap (universal)

**macOS / Linux:**

```sh
curl -fsSL https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.sh | sh
```

**Windows (PowerShell):**

```powershell
irm https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.ps1 | iex
```

Detects your OS and CPU architecture, downloads the matching prebuilt archive from the latest GitHub Release, drops `corvid` on your `PATH`, and runs `corvid doctor`. Falls back to a `cargo install` from source if no prebuilt is available for your platform.

### 2. Homebrew (macOS, Linux)

```sh
brew install Micrurus-Ai/corvid/corvid
```

This expands to `brew tap Micrurus-Ai/corvid && brew install corvid`. The formula points at the same prebuilt archives, with sha256s verified by Homebrew.

### 3. Scoop (Windows)

```powershell
scoop bucket add corvid https://github.com/Micrurus-Ai/scoop-corvid
scoop install corvid
```

The bucket manifest declares a `checkver`/`autoupdate` block, so Scoop's central bot keeps it current with Corvid releases on its own (~24h cadence after a tag).

### 4. From source via Cargo

```sh
git clone https://github.com/Micrurus-Ai/Corvid-lang
cd Corvid-lang
cargo install --path crates/corvid-cli --locked
```

Slowest path (5–15 min on a cold build), but works on any platform with a Rust toolchain. The bootstrap scripts fall back to this automatically when prebuilt binaries aren't available.

### Verification

After any of the four, in a fresh shell:

```
corvid --help
corvid doctor
corvid tour --list
```

---

## What gets installed

```
~/.corvid/                              (Linux/macOS)
%USERPROFILE%\.corvid\                  (Windows)
├── bin/
│   └── corvid(.exe)                    # the CLI; added to your PATH
├── std/                                # Corvid standard library
│   ├── ai.cor
│   ├── auth.cor
│   ├── effects.cor
│   └── ...
├── LICENSE-MIT
├── LICENSE-APACHE
├── README.md
└── src/                                # only present when source-built
```

Two environment changes:

- `PATH` gains `$CORVID_HOME/bin`.
- `CORVID_HOME` points at the install root. `corvid new <project>` reads this to vendor the stdlib into fresh projects so `import "./std/foo"` works without further setup.

Homebrew installs to its own prefix (`$(brew --prefix corvid)`), Scoop installs to its bucket directory. The directory layout (`bin/` next to `std/`) is identical so the `find_std_source` resolver in `corvid new` finds the stdlib in all three cases.

---

## Architecture

```
                              END USER
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
    curl … install.sh     brew install …      scoop install corvid
            │                    │                    │
            └────────┬───────────┴────────────────────┘
                     │
                     ▼ (downloads matching archive)
       ┌──────────────────────────────────────────────┐
       │     github.com/Micrurus-Ai/Corvid-lang       │
       │     /releases/v<X.Y.Z>/corvid-<target>.*     │
       └──────────────────────────────────────────────┘
                     ▲
                     │ uploaded by
       ┌──────────────────────────────────────────────┐
       │  release.yml (triggered by `git push v*.*.*`) │
       │  builds 4 prebuilt archives in parallel      │
       └──────────────────────────────────────────────┘
                     │
                     ▼ (on success, triggers)
       ┌──────────────────────────────────────────────┐
       │  publish-managers.yml                        │
       │  bumps homebrew-corvid/Formula/corvid.rb     │
       └──────────────────────────────────────────────┘

       Scoop's central bot polls Corvid-lang's releases page
       independently and bumps scoop-corvid every ~24h.
```

Two trigger events drive the whole pipeline:

1. **Push a `v*.*.*` tag** → `release.yml` builds → archives uploaded → `publish-managers.yml` bumps Homebrew. Whole flow: ~15 min start to finish.
2. **Push to `install/**` or `web/**`** → `test-installers.yml` lints + smoke-tests the bootstrap scripts. Catches regressions before users see them.

Nothing else in this pipeline needs human attention between releases.

---

## Pipeline components

### `install/install.sh` — Unix bootstrap

POSIX `sh` script (works under bash, zsh, dash, ash). Invoked via `curl … | sh`. Cross-platform between macOS and Linux, x86_64 and aarch64. ~150 lines.

Behavior, in order:

1. Detect OS (`linux*` / `darwin*`) and arch (`x86_64` / `aarch64`).
2. Build the prebuilt asset URL for the resolved target triple.
3. Try to download the archive with `curl` (or `wget` fallback) into a temp dir.
4. **Fast path:** extract into `$CORVID_HOME`, install `bin/corvid` and `std/`.
5. **Fallback:** if the download 404'd, install `rustup` if needed, `git clone --depth 1` Corvid-lang, run `cargo install --path crates/corvid-cli --locked --root $CORVID_HOME`.
6. Append a `# corvid` block to `~/.bashrc` / `~/.zshrc` / `~/.profile` exporting `CORVID_HOME` and updating `PATH`.
7. Run `corvid doctor` for a final health check.

Configurable via env vars (see [Configuration reference](#configuration-reference)).

### `install/install.ps1` — Windows bootstrap

Same logic as the Unix script, in PowerShell 5.1+. Invoked via `irm … | iex`. ~150 lines.

Two Windows-specific quirks worth knowing:

- The fallback path uses `winget install Rustlang.Rustup` first, then falls back to downloading `rustup-init.exe` from `win.rustup.rs/x86_64`.
- `PATH` is updated via `[Environment]::SetEnvironmentVariable(...)` at User scope (not Machine — no admin needed). Users must open a new terminal to see it.

### `.github/workflows/release.yml` — Build matrix

Triggers on `push: tags: ["v*.*.*"]`. Four parallel jobs:

| OS runner | Target triple | Archive |
| --- | --- | --- |
| `ubuntu-latest` | `x86_64-unknown-linux-gnu` | `.tar.gz` |
| `macos-latest` | `x86_64-apple-darwin` | `.tar.gz` |
| `macos-latest` | `aarch64-apple-darwin` | `.tar.gz` |
| `windows-latest` | `x86_64-pc-windows-msvc` | `.zip` |

Each job:

1. Checkout, install Rust toolchain with the target added.
2. `cargo build --release --locked --target $TARGET -p corvid-cli` (this is the long step — 5–10 min on cold runners).
3. Stage `bin/corvid(.exe)` + `std/` + licenses into `dist/corvid-<target>/`.
4. Archive (tar.gz on Unix, zip on Windows).
5. Upload to the release via `softprops/action-gh-release@v2`.

The `--locked` flag requires `Cargo.lock` to be present and unchanged — it's committed to the repo for exactly this reason.

### `.github/workflows/publish-managers.yml` — Auto-bump Homebrew

Triggers on `workflow_run: workflows: ["release"], types: [completed]` and runs only when the upstream run succeeded with a tag head ref. Also exposes `workflow_dispatch` with a tag input for manual re-runs.

Steps:

1. Resolve the tag and version (`v0.1.0` → `0.1.0`).
2. Download each macOS/Linux release archive, compute sha256s.
3. Checkout `Micrurus-Ai/homebrew-corvid` using the `MANAGERS_TOKEN` PAT.
4. Rewrite `Formula/corvid.rb` with the new version + sha256s (a small Python regex pass).
5. Commit as `corvid <version>` and push.

If the formula is already at that version (e.g., re-running for the same tag), the script's `git diff --cached --quiet` guard exits 0 cleanly — idempotent.

Scoop is intentionally not handled here. The Scoop manifest declares `checkver`/`autoupdate`, so Scoop's central bot tracks Corvid releases on its own. Nothing for us to maintain.

### `.github/workflows/test-installers.yml` — CI

Triggers on `push`/`pull_request` to `install/**`, `web/**`, or this file itself. Three job tiers:

- **`lint`** (~30s): `shellcheck` and `sh -n` on `install.sh`, `node --check` on `worker.js`. Runs on every PR.
- **`pwsh-lint`** (~60s): `Invoke-ScriptAnalyzer` on `install.ps1` with `-ExcludeRule PSAvoidUsingWriteHost` (Write-Host is the right call for an interactive bootstrap). Runs on every PR.
- **`smoke-linux`** (~10 min): runs `install.sh` end-to-end against the current checkout with `CORVID_VERSION=nonexistent-test-tag` to force the source-build fallback path. Asserts that `corvid` lands on `PATH` and that `corvid new testproj` produces a project with `corvid.toml`, `src/main.cor`, and a vendored `std/`. Slow but gates a real regression class.
- **`smoke-matrix`** (macOS + Windows): same end-to-end test, but `workflow_dispatch` only — too slow to run on every PR.

### `web/worker.js` — Cloudflare short-URL Worker

A Cloudflare Worker that turns one origin (e.g. `corvid.sh`) into three responses based on `User-Agent`:

| Caller | Response |
| --- | --- |
| `curl` / `wget` | `install/install.sh` content |
| PowerShell | `install/install.ps1` content |
| Web browser | A branded landing page (inlined HTML) with copy-paste commands auto-highlighted for the visitor's OS |

Scripts are proxied straight from `raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/...` and cached at the edge for 5 min. Editing the install scripts in `main` updates the served content within 5 min, no Worker redeploy required.

Free Cloudflare tier covers ~100k installs/day. Deploy: `wrangler login && wrangler deploy` from `web/`. See [Corvid-lang/web/README.md](https://github.com/Micrurus-Ai/Corvid-lang/blob/main/web/README.md) for full deploy steps and DNS setup.

### `Micrurus-Ai/homebrew-corvid` — Homebrew tap

A standard Homebrew tap (`homebrew-<name>` repo naming convention). One file: `Formula/corvid.rb`, a Ruby class that declares prebuilt URLs and sha256s for macOS Apple Silicon, macOS Intel, and Linux x86_64. The `def install` block places `bin/corvid` and `prefix/std` in the right spots so `find_std_source` resolves the stdlib via the binary's neighborhood.

Auto-bumped on every release by `publish-managers.yml`.

### `Micrurus-Ai/scoop-corvid` — Scoop bucket

A Scoop bucket. One file: `bucket/corvid.json`, declaring the Windows zip URL, sha256, `extract_dir`, and `bin` path. Sets `env_set: { CORVID_HOME: $dir }` so `corvid new` finds the stdlib next to the binary.

Includes `checkver`/`autoupdate`, so Scoop's central bot bumps the manifest automatically when new Corvid releases ship.

### `corvid new` stdlib auto-vendoring (upstream change)

A change in `crates/corvid-driver/src/scaffold.rs` that hooks into `corvid new <project>`: after creating the standard scaffold (`corvid.toml`, `src/main.cor`, `tools.py`, `.gitignore`), the function calls `vendor_std(&project_root)` which:

1. Looks up the system `std/` directory using:
   - `$CORVID_HOME/std` if `CORVID_HOME` is set, else
   - `<exe-dir>/../std` (the layout the installer produces).
2. If found, recursively copies it into `<project>/std/`.
3. If not found, silently no-ops.

This closes the gap between "install Corvid" and "write working code." Before this change, `import "./std/effects"` in a fresh project failed because the resolver is purely path-relative — it has no system search path. With it, every install pathway (curl bootstrap, brew, scoop) produces a `corvid new` flow that just works.

---

## Supported targets

Fast path (prebuilt downloads):

| OS | Architecture | Target triple | Archive |
| --- | --- | --- | --- |
| Linux | x86_64 | `x86_64-unknown-linux-gnu` | `.tar.gz` |
| macOS | Intel | `x86_64-apple-darwin` | `.tar.gz` |
| macOS | Apple Silicon | `aarch64-apple-darwin` | `.tar.gz` |
| Windows | x86_64 | `x86_64-pc-windows-msvc` | `.zip` |

Other targets (e.g. Linux aarch64, FreeBSD, Windows ARM64) automatically use the source-build fallback. To add a new target to the fast path, extend the matrix in `release.yml` and the OS/arch detection in `install.sh` / `install.ps1`.

---

## Configuration reference

### Bootstrap env vars

| Variable | Default | Purpose |
| --- | --- | --- |
| `CORVID_REPO` | `Micrurus-Ai/Corvid-lang` | Source repo to install from. Override to install from a fork. |
| `CORVID_VERSION` | `latest` | Release tag, e.g. `v0.1.0`. |
| `CORVID_HOME` | `$HOME/.corvid` (or `%USERPROFILE%\.corvid` on Windows) | Install root. |

Pin a specific version:

```sh
CORVID_VERSION=v0.1.0 curl -fsSL https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.sh | sh
```

Install from a fork:

```sh
CORVID_REPO=myuser/Corvid-lang curl -fsSL https://raw.githubusercontent.com/myuser/Corvid-lang/main/install/install.sh | sh
```

### Workflow secrets (Corvid-lang)

| Secret | Used by | Purpose |
| --- | --- | --- |
| `MANAGERS_TOKEN` | `publish-managers.yml` | Fine-grained PAT with `Contents: Read and write` scoped to `Micrurus-Ai/homebrew-corvid`. Lets the workflow push the bumped formula. |
| `GITHUB_TOKEN` | `release.yml` | Provided by GitHub Actions. Used by `softprops/action-gh-release@v2` to create the release object and upload assets. No setup needed. |

---

## Cutting a release

```sh
cd Corvid-lang
git checkout main && git pull
git tag v0.1.1
git push origin v0.1.1
```

That triggers everything. ~15 minutes later:

1. ✓ `release.yml` has built and uploaded 4 archives to `https://github.com/Micrurus-Ai/Corvid-lang/releases/tag/v0.1.1`.
2. ✓ `publish-managers.yml` has pushed a `corvid 0.1.1` commit to `homebrew-corvid/Formula/corvid.rb`.
3. ⏳ Scoop's central bot will pick up the new version on its next daily run (within ~24h).

The bootstrap one-liners point at `releases/latest/download/...` and update instantly the moment `release.yml` finishes.

### Verifying a release

```sh
# Are all 4 archives published?
for t in aarch64-apple-darwin x86_64-apple-darwin x86_64-unknown-linux-gnu x86_64-pc-windows-msvc; do
  ext=tar.gz; [ "$t" = "x86_64-pc-windows-msvc" ] && ext=zip
  printf '%-30s %s\n' "$t" "$(curl -sIo /dev/null -w '%{http_code}' "https://github.com/Micrurus-Ai/Corvid-lang/releases/download/v0.1.1/corvid-${t}.${ext}")"
done

# Did the formula bump?
curl -fsSL https://raw.githubusercontent.com/Micrurus-Ai/homebrew-corvid/main/Formula/corvid.rb | grep '^  version'

# Does the install actually work?
CORVID_VERSION=v0.1.1 sh install/install.sh
~/.corvid/bin/corvid --version
```

---

## Operations runbook

### Rotating `MANAGERS_TOKEN`

The PAT used by `publish-managers.yml` expires (currently set to ~1 year). When it does, the post-release Homebrew bump silently fails until rotated.

1. https://github.com/settings/personal-access-tokens → click `corvid-managers-bot` → **Regenerate token**.
2. Verify Repository access still includes `Micrurus-Ai/homebrew-corvid` and Repository permissions still grant `Contents: Read and write`.
3. Copy the new `github_pat_…` value.
4. https://github.com/Micrurus-Ai/Corvid-lang/settings/secrets/actions → click `MANAGERS_TOKEN` → **Update secret** → paste → save.
5. Test: trigger `publish-managers.yml` via `workflow_dispatch` with tag `v<latest>`. Should succeed in ~30s (no-op push) or commit a fresh bump.

### Debugging `release.yml` failures

The build step (`cargo build --release --locked --target X -p corvid-cli`) is the most common failure point. By order of likelihood:

1. **`Cargo.lock` out of sync with `Cargo.toml`.** Run `cargo build` locally, commit the updated `Cargo.lock`, push, retag (or push a `v0.1.x+1`).
2. **One platform breaks specifically.** Usually a target-specific dep (e.g. `rusqlite` needs `libsqlite3-dev` on some Linux distros, or a Windows-specific feature flag is missing). Check the failed job's log on the Actions tab.
3. **Cross-compile linker missing.** Only relevant if you add aarch64-linux to the matrix; needs `gcc-aarch64-linux-gnu` installed on the ubuntu runner.

If `release.yml` succeeded but the GitHub Release object is missing assets, look at the `Upload to release` step — usually a transient network blip; re-run failed jobs.

### Manually bumping the Homebrew formula

If `publish-managers.yml` fails and you need the formula bumped before the next release, do it locally:

```sh
git clone https://github.com/Micrurus-Ai/homebrew-corvid && cd homebrew-corvid

VERSION=0.1.1
for t in aarch64-apple-darwin x86_64-apple-darwin x86_64-unknown-linux-gnu; do
  url="https://github.com/Micrurus-Ai/Corvid-lang/releases/download/v${VERSION}/corvid-${t}.tar.gz"
  sum=$(curl -fsSL "$url" | sha256sum | awk '{print $1}')
  echo "$t  $sum"
done
```

Edit `Formula/corvid.rb` directly:

- Update `version "X.Y.Z"`.
- Replace each of the three placeholder/old `sha256 "..."` values with the new ones (one per `on_arm` / `on_intel` block).

Commit and push:

```sh
git commit -am "corvid $VERSION"
git push origin main
```

### Manually bumping the Scoop manifest

Scoop's central bot usually handles this on its own within 24h. If you need it sooner:

```sh
git clone https://github.com/Micrurus-Ai/scoop-corvid && cd scoop-corvid

VERSION=0.1.1
url="https://github.com/Micrurus-Ai/Corvid-lang/releases/download/v${VERSION}/corvid-x86_64-pc-windows-msvc.zip"
sum=$(curl -fsSL "$url" | sha256sum | awk '{print $1}')
echo "$sum"
```

Edit `bucket/corvid.json` to update the `version`, `architecture.64bit.url`, and `architecture.64bit.hash` fields. Commit and push.

### Re-deploying the Cloudflare Worker

The Worker only needs redeploy when its logic changes — not when the install scripts change (those are proxied at request time). To redeploy:

```sh
cd Corvid-lang/web
wrangler login   # one-time
wrangler deploy
```

To verify routing works after a custom-domain change:

```sh
curl -I https://corvid.sh                     # → 200 text/html
curl -I -A 'curl/8.0' https://corvid.sh       # → 200 text/x-shellscript
curl -I -A 'PowerShell/7.4' https://corvid.sh # → 200 text/plain (install.ps1)
```

### Re-running `publish-managers` for a past release

```
https://github.com/Micrurus-Ai/Corvid-lang/actions/workflows/publish-managers.yml
→ Run workflow → Tag: v0.1.1 → Run workflow
```

Idempotent: if the formula is already at that version, the script exits 0 cleanly without a no-op commit.

---

## Local development

### Testing `install.sh`

In a clean directory, point at this checkout's script and force the cargo fallback so you don't depend on a real release:

```sh
CORVID_VERSION=nonexistent-test-tag \
CORVID_REPO=Micrurus-Ai/Corvid-lang \
CORVID_HOME=/tmp/corvid-test \
sh install/install.sh
/tmp/corvid-test/bin/corvid --help
```

For a faster iteration loop, run inside Docker:

```sh
docker run --rm -it -v "$PWD/install:/install" ubuntu:24.04 bash
# inside:
apt-get update && apt-get install -y curl ca-certificates
CORVID_HOME=/root/.corvid CORVID_VERSION=v0.1.0 sh /install/install.sh
```

### Testing `install.ps1`

In a fresh PowerShell session:

```powershell
$env:CORVID_HOME = "$env:TEMP\corvid-test"
$env:CORVID_VERSION = "v0.1.0"
.\install\install.ps1
& $env:CORVID_HOME\bin\corvid.exe --help
```

To lint locally:

```powershell
Invoke-ScriptAnalyzer -Path install\install.ps1 -Severity Error,Warning -ExcludeRule PSAvoidUsingWriteHost
```

### Testing the Worker locally

```sh
cd web
wrangler dev
# in another terminal:
curl -fsSL http://localhost:8787              # → HTML landing page
curl -fsSL -A 'curl/8.0' http://localhost:8787 # → install.sh content
curl -fsSL -A 'PowerShell/7' http://localhost:8787 # → install.ps1 content
```

### Testing release-pipeline changes without cutting a real version

Push your changes to a branch, then push a pre-release tag:

```sh
git checkout -b ci-test
git push -u origin ci-test
git tag v0.0.99-rc1
git push origin v0.0.99-rc1
```

Watch the Actions tab. When you're done, delete the tag and the Release object:

```sh
git push --delete origin v0.0.99-rc1
gh release delete v0.0.99-rc1 -y    # if gh CLI is installed
```

---

## Uninstall

**macOS / Linux:**

```sh
rm -rf ~/.corvid
# Then remove the `# corvid` block from any of:
#   ~/.bashrc  ~/.zshrc  ~/.profile
```

**Windows:**

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.corvid"
[Environment]::SetEnvironmentVariable('CORVID_HOME', $null, 'User')
# Also strip the .corvid\bin entry from your user PATH:
#   sysdm.cpl → Advanced → Environment Variables → Path (User) → Edit
```

**Homebrew:**

```sh
brew uninstall corvid
brew untap Micrurus-Ai/corvid
```

**Scoop:**

```powershell
scoop uninstall corvid
scoop bucket rm corvid
```

---

## Design decisions

### Why `curl … | sh` and `irm … | iex`

Industry norm. rustup, bun, deno, pnpm, nvm, ohmyzsh, Homebrew itself all ship this way. Developers recognize the pattern, every shell on every platform supports it, no package manager is required, and the script can adapt to the user's environment in ways a static binary download can't. Yes, it requires trusting the URL — but that's true of every install method, and a curl-pipe is *more* auditable than an opaque MSI/PKG.

### Why prebuilt binaries instead of `cargo install`

`cargo install --path crates/corvid-cli --locked` works, but a cold build takes 5–15 minutes. For a brand-new language, that latency is fatal — most evaluators bounce in <60 seconds. Prebuilt archives drop that to ~10 seconds. The cargo path stays as a fallback for unusual platforms and as a proof that the source builds.

### Why a Cloudflare Worker instead of GitHub Pages

Pages can do path-based redirects but not User-Agent-based content negotiation. We need the same URL (`corvid.sh`) to return three different bodies (sh, ps1, HTML) depending on who's asking. A 302 redirect doesn't work because `curl … | sh` doesn't follow redirects without `-L`, and asking every user to remember `-L` defeats the simplicity. The Worker is ~50 lines of JS, free up to 100k requests/day, and carries automatic edge caching.

### Why `corvid new` vendors `std/` automatically

The Corvid module resolver (`crates/corvid-resolve/src/modules.rs`) is purely relative to the importing `.cor` file. There is no search-path mechanism, no `CORVID_STD` env var, no embedded stdlib. So `import "./std/foo"` literally requires a `std/` directory in the user's project. Without auto-vendoring, every new user discovers this the hard way after their first `import` statement breaks the build. Hooking into `corvid new` to copy `$CORVID_HOME/std` into each new project closes the gap between "install Corvid" and "write working code."

### Why `Cargo.lock` is committed

`corvid-cli` is a binary crate. The standard Rust convention for binaries is to commit the lockfile so that release builds are byte-reproducible across contributors and CI. The repo originally had `Cargo.lock` in `.gitignore` (the `cargo new --lib` default, correct for libraries) — that mismatch caused the first `release.yml` run to fail on every platform with `cargo build --locked` because the lockfile wasn't there. Fixing the gitignore + committing the lockfile resolved it.

### Why the install scripts work without the Cloudflare Worker

The `corvid.sh` short URL is polish, not infrastructure. The bootstrap scripts live at stable `raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/...` URLs that work whether or not Cloudflare is involved. If the Worker ever goes down, the long URLs in the README still install Corvid identically. The Worker is purely a UX optimization.

### Why we ship four install paths instead of one

Different developers have different muscle memory:

- Rustaceans expect `cargo install`. We support it, with the bootstrap as fallback.
- macOS devs expect `brew install`. The tap delivers it.
- Windows developers split between `winget`, `scoop`, and just downloading. Scoop is the closest analog to brew; it works.
- Everyone else lands on the curl/irm one-liner.

Maintaining all four is cheap because the heavy lifting (build matrix, sha256 distribution) happens once; each install path is a thin manifest pointing at the same artifacts.

---

## License

Dual-licensed under [MIT](https://opensource.org/licenses/MIT) or [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0), at your option, matching upstream Corvid-lang.
