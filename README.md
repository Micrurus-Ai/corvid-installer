# corvid-installer

One-line installer for the [Corvid](https://github.com/Micrurus-Ai/Corvid-lang) programming language.

## Install

**Windows** (PowerShell):

```powershell
irm https://raw.githubusercontent.com/<owner>/corvid-installer/main/install.ps1 | iex
```

**macOS / Linux**:

```sh
curl -fsSL https://raw.githubusercontent.com/<owner>/corvid-installer/main/install.sh | sh
```

Replace `<owner>` with the GitHub user/org hosting this repo. After install, open a new terminal and run:

```
corvid doctor
corvid tour --list
```

## What it installs

```
~/.corvid/                  (or %USERPROFILE%\.corvid on Windows)
├── bin/
│   └── corvid(.exe)        # the CLI, added to your PATH
├── std/                    # Corvid standard library
└── src/                    # only present when building from source (fallback)
```

Two environment variables are set in your user profile:

- `CORVID_HOME` — points at the install root
- `PATH` — gains `$CORVID_HOME/bin`

## How it works

The installer takes a fast path when a prebuilt release is available, and falls back to a source build otherwise:

1. **Fast path** — downloads `corvid-<target>.{zip,tar.gz}` from the latest [Corvid-lang Release](https://github.com/Micrurus-Ai/Corvid-lang/releases) for your OS/arch, extracts it into `$CORVID_HOME`, wires up `PATH`, runs `corvid doctor`. Takes seconds.
2. **Fallback** — if the release asset is missing (e.g. no release published yet for your platform), the script ensures `rustup`/`cargo` are installed, clones Corvid-lang shallowly, and runs `cargo install --path crates/corvid-cli`. Takes 5–15 minutes on a first run.

Supported targets on the fast path:

| OS      | Architecture     | Target triple                  |
| ------- | ---------------- | ------------------------------ |
| Linux   | x86_64           | `x86_64-unknown-linux-gnu`     |
| macOS   | Intel            | `x86_64-apple-darwin`          |
| macOS   | Apple Silicon    | `aarch64-apple-darwin`         |
| Windows | x86_64           | `x86_64-pc-windows-msvc`       |

Other targets fall back to the source build automatically.

## Overrides

| Env var          | Default                   | Purpose                            |
| ---------------- | ------------------------- | ---------------------------------- |
| `CORVID_REPO`    | `Micrurus-Ai/Corvid-lang` | Source repo to install from        |
| `CORVID_VERSION` | `latest`                  | Release tag, e.g. `v0.1.0`         |
| `CORVID_HOME`    | `~/.corvid`               | Install root                       |

Example — pin a specific release:

```sh
CORVID_VERSION=v0.1.0 curl -fsSL https://raw.githubusercontent.com/<owner>/corvid-installer/main/install.sh | sh
```

## Uninstall

```sh
# macOS / Linux
rm -rf ~/.corvid
# then remove the `# corvid` block from ~/.bashrc / ~/.zshrc / ~/.profile
```

```powershell
# Windows
Remove-Item -Recurse -Force $env:USERPROFILE\.corvid
[Environment]::SetEnvironmentVariable('CORVID_HOME', $null, 'User')
# then strip $env:USERPROFILE\.corvid\bin from your user PATH
```

## For Corvid-lang maintainers — enabling the fast path

The fast path needs prebuilt release archives in the Corvid-lang repo. To turn it on:

1. Copy [`release.yml`](./release.yml) to `.github/workflows/release.yml` in [Corvid-lang](https://github.com/Micrurus-Ai/Corvid-lang).
2. Tag and push a release:
   ```sh
   git tag v0.1.0
   git push origin v0.1.0
   ```
3. The workflow builds `corvid` for Linux / macOS Intel / macOS ARM / Windows and attaches archives to the GitHub Release. The installer picks them up automatically via the `releases/latest/download/...` URL.

Until step 2 runs once, every install will use the slow cargo fallback.

## Hosting this installer

Once this directory is pushed to a public GitHub repo, the raw URLs above work as-is. No GitHub Pages or CDN required — `raw.githubusercontent.com` is sufficient. If you later want a friendlier URL (e.g. `install.corvid-lang.org`), redirect it to the raw URL with a 302.
