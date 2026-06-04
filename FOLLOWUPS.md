# FOLLOWUPS

Open work on the install pipeline. Each item is independent — they can be picked up in any order.

Mark items done by replacing `[ ]` with `[x]` in the checklist. The headings below give the full context, commands, and verification steps for each.

## Checklist

- [x] [Publish corvid-installer as a public GitHub repo](#1-publish-corvid-installer-publicly) — done; `origin` set, `main` in sync.
- [ ] [Narrow `MANAGERS_TOKEN` scope to just `homebrew-corvid`](#2-narrow-managers_token-scope)
- [ ] [Buy a short domain and deploy the Cloudflare Worker](#3-deploy-the-corvidsh-cloudflare-worker)
- [ ] [Update install one-liners in Corvid-lang README to use `corvid.sh`](#4-switch-readme-install-commands-to-corvidsh)
- [ ] [Mirror the operations runbook into `Corvid-lang/docs/`](#5-mirror-the-runbook-into-corvid-lang)
- [ ] [Add a calendar reminder for `MANAGERS_TOKEN` rotation](#6-set-a-calendar-reminder-for-pat-rotation)
- [ ] [Address the 26 Dependabot alerts on Corvid-lang](#7-address-dependabot-alerts)
- [ ] [Decide on Linux aarch64 + Windows aarch64 prebuilt support](#8-extend-the-release-matrix)
- [ ] [Add a `corvid self update` CLI command (upstream feature)](#9-add-corvid-self-update)

---

## 1. Publish corvid-installer publicly

Right now this repo lives only on your local disk. Pushing it to GitHub gives it a stable home, makes the runbook in [README.md](./README.md) discoverable to other contributors, and lets future-you reference these docs from anywhere.

### Steps

1. Create an empty repo (no README, no `.gitignore`, no license — keep it blank): https://github.com/organizations/Micrurus-Ai/repositories/new — name `corvid-installer`, public.
2. Add the remote and push:

   ```sh
   cd 'C:\Users\SBW\OneDrive - Axon Group\Documents\GitHub\corvid-installer'
   git remote add origin https://github.com/Micrurus-Ai/corvid-installer.git
   git push -u origin main
   ```

3. Verify: visit https://github.com/Micrurus-Ai/corvid-installer and confirm the README renders with all sections.

### Why public

The README references public Corvid-lang URLs and contains no secrets. Public visibility makes it linkable from Corvid-lang's main README ("see corvid-installer for the operations runbook"). If you'd rather keep it private, that's also fine — the operations docs work either way.

### Alternative: archive instead

The canonical files are mirrored into Corvid-lang. If you'd prefer not to maintain two repos, you can leave this as a local-only working dir and rely on commit history here as your audit trail. Pick whichever feels less like overhead.

---

## 2. Narrow `MANAGERS_TOKEN` scope

The PAT created during v0.1.0 setup currently has Repository access including 2 repositories (visible as "Repositories 2" in the UI). Ideally it should only have `Micrurus-Ai/homebrew-corvid` selected — minimum scope for what `publish-managers.yml` actually does.

### Steps

1. Go to https://github.com/settings/personal-access-tokens, click `corvid-managers-bot`.
2. Under **Repository access** → **Only select repositories**, click **Select repositories**.
3. Confirm only `Micrurus-Ai/homebrew-corvid` is in the list. Remove any others.
4. Click **Update** at the top.
5. Click **Regenerate token** (red button, top-right) — without regenerating, scope changes don't always apply to the existing token value.
6. Copy the new `github_pat_…` value.
7. Update the secret: https://github.com/Micrurus-Ai/Corvid-lang/settings/secrets/actions → `MANAGERS_TOKEN` → **Update secret** → paste.
8. Verify: trigger `publish-managers.yml` via `workflow_dispatch` with tag `v0.1.0`. Should run in ~30s with conclusion `success` and exit code 0 (no-op since the formula is already at v0.1.0).

### Why narrow scope matters

If this PAT ever leaks (compromised CI logs, accidental commit, etc.), an attacker can only modify what the token has access to. Currently that's two repos; ideally one. Standard least-privilege.

---

## 3. Deploy the corvid.sh Cloudflare Worker

The Worker code lives in [Corvid-lang/web/](https://github.com/Micrurus-Ai/Corvid-lang/tree/main/web) and is ready to deploy. Once live, end-users get the shortest possible install commands:

```sh
curl -fsSL corvid.sh | sh        # macOS / Linux
irm corvid.sh | iex              # Windows
```

### Steps

1. **Pick a domain.** `corvid.sh` is the example. Alternatives: `corvid.dev`, `corvid.run`, `getcorvid.dev`, `corvid-lang.com`. Cost: ~$10–30/year depending on TLD.
2. **Register the domain** at any registrar (Namecheap, Porkbun, Cloudflare Registrar — Cloudflare's tends to be the cheapest if available).
3. **Add the domain to Cloudflare** (free plan): https://dash.cloudflare.com/sign-up if you don't have an account, then **Add a site** → enter domain → Free → follow nameserver-change instructions at your registrar. Propagation: minutes to hours.
4. **Install wrangler locally:**

   ```powershell
   npm install -g wrangler
   ```

5. **Deploy the Worker:**

   ```powershell
   cd 'C:\Users\SBW\OneDrive - Axon Group\Documents\GitHub\Corvid-lang\web'
   wrangler login
   wrangler deploy
   ```

6. **Bind your domain:** Cloudflare dashboard → **Workers & Pages** → click `corvid-installer` → **Settings** → **Triggers** → **Custom Domains** → **Add Custom Domain** → enter your domain. SSL certificate auto-provisions.
7. **Verify routing** (replace `corvid.sh` with whatever you bought):

   ```sh
   curl -I https://corvid.sh                       # → 200 text/html (browser path)
   curl -I -A 'curl/8.0' https://corvid.sh         # → 200 text/x-shellscript
   curl -I -A 'PowerShell/7.4' https://corvid.sh   # → 200 text/plain
   ```

### Cost

Free tier: 100,000 requests/day. You'd need 100k+ installs/day before paying anything. At paid tier it's $5/month for 10M.

### What if you skip this

The four existing install paths still work — `corvid.sh` is purely a UX polish that gets developers from "I want to try Corvid" to "type one short command" with the least friction. The long `raw.githubusercontent.com/...` URLs work identically.

---

## 4. Switch README install commands to `corvid.sh`

Only do this **after** [#3](#3-deploy-the-corvidsh-cloudflare-worker). If you switch before the domain resolves, every reader gets a DNS error.

### Steps

1. In `Micrurus-Ai/Corvid-lang/README.md`, find the `## Install` section (currently around line 496).
2. Replace the long URLs with the short form:

   ```diff
   -irm https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.ps1 | iex
   +irm https://corvid.sh | iex

   -curl -fsSL https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.sh | sh
   +curl -fsSL https://corvid.sh | sh
   ```

3. Optional: do the same in the `CORVID_VERSION` example.
4. Commit and push:

   ```powershell
   cd 'C:\Users\SBW\OneDrive - Axon Group\Documents\GitHub\Corvid-lang'
   git add README.md
   git commit -m "docs(install): use corvid.sh short URL"
   git push origin main
   ```

5. Mirror the change in this repo's [README.md](./README.md) (the long URLs are documented in two places: the Quick Install section and the `CORVID_VERSION` example).

---

## 5. Mirror the runbook into Corvid-lang

Right now [README.md](./README.md) here is the only copy of the operations runbook. If on-call needs to debug a failed release at 2am, they probably won't know to look in `corvid-installer`.

### Steps

Two options — pick one, not both:

**Option A: copy a slim runbook into Corvid-lang/docs/**

```sh
cd 'C:\Users\SBW\OneDrive - Axon Group\Documents\GitHub\Corvid-lang'
mkdir -p docs
# Create docs/install-pipeline.md and copy these sections from corvid-installer/README.md:
#   - Architecture (the ASCII diagram)
#   - Cutting a release
#   - Operations runbook (full)
#   - Configuration reference
# Skip end-user install instructions and design decisions — those live in corvid-installer.
```

Commit + push.

**Option B: link from Corvid-lang's main README**

Add a single line to the Install section of `Corvid-lang/README.md`:

```markdown
For maintainers: see the [install pipeline operations runbook](https://github.com/Micrurus-Ai/corvid-installer/blob/main/README.md) for release-cutting, debugging, and on-call material.
```

Requires [#1](#1-publish-corvid-installer-publicly) to be done first.

Option A is more durable (the docs ship with the language). Option B is cheaper to maintain (one source of truth in `corvid-installer`).

---

## 6. Set a calendar reminder for PAT rotation

`MANAGERS_TOKEN` was created with a 1-year expiration. The day it expires, `publish-managers.yml` silently fails on the next release, the Homebrew formula stops bumping, and you'll only notice when a user reports stale `brew install` versions.

### Steps

1. Find the exact expiry: https://github.com/settings/personal-access-tokens → click `corvid-managers-bot` → top of page reads "Expires on …".
2. Add a reminder ~1 week before that date in whatever calendar you use.
3. Reminder body: "Rotate Corvid `MANAGERS_TOKEN` PAT — see [Operations runbook → Rotating MANAGERS_TOKEN](./README.md#rotating-managers_token)."

---

## 7. Address Dependabot alerts

The push of `Cargo.lock` made transitive CVEs visible: GitHub Dependabot now reports 26 vulnerabilities (19 moderate, 7 low) on Corvid-lang. They were always present; we just couldn't see them before.

### Steps

1. Visit https://github.com/Micrurus-Ai/Corvid-lang/security/dependabot.
2. Triage by severity. Moderate ones in `tokio`, `serde`, etc. are usually safe to ignore for a v0.1.x. Anything around `openssl`, `rustls`, or HTTP clients is worth bumping.
3. Most can be addressed by running `cargo update -p <crate>` and committing the new `Cargo.lock`. Some need a `Cargo.toml` version-bump for major-version transitives.
4. After bumping, run the smoke job locally to make sure nothing breaks:

   ```sh
   cargo build --release --locked --target x86_64-unknown-linux-gnu -p corvid-cli
   ```

This is a hygiene task, not a critical one. Treat it as ongoing maintenance.

---

## 8. Extend the release matrix

The current `release.yml` builds for 4 targets (Linux x86_64, macOS Intel, macOS Apple Silicon, Windows x86_64). Two notable gaps:

- **Linux aarch64** (`aarch64-unknown-linux-gnu`) — common on M-series Macs running Linux VMs, AWS Graviton, Raspberry Pi.
- **Windows ARM64** (`aarch64-pc-windows-msvc`) — Surface Pro X and newer Snapdragon laptops.

Both fall back to source build today, which works but takes 5–15 min.

### Steps

Edit `Micrurus-Ai/Corvid-lang/.github/workflows/release.yml`. Add to the matrix:

```yaml
- { os: ubuntu-latest,  target: aarch64-unknown-linux-gnu, archive: tar.gz, cross: true }
- { os: windows-latest, target: aarch64-pc-windows-msvc,   archive: zip }
```

For Linux aarch64, add a step before Build to install the cross-compile linker:

```yaml
- name: Install aarch64 cross-compiler
  if: matrix.cross
  run: |
    sudo apt-get update
    sudo apt-get install -y gcc-aarch64-linux-gnu
    mkdir -p .cargo
    cat >> .cargo/config.toml <<'EOF'
    [target.aarch64-unknown-linux-gnu]
    linker = "aarch64-linux-gnu-gcc"
    EOF
```

For Windows ARM64, the MSVC toolchain on `windows-latest` cross-compiles natively — no extra setup.

Then add the new targets to the OS/arch detection in `install/install.sh` and `install/install.ps1`.

### Steps to verify

Cut a `v0.1.x-rc1` pre-release tag and watch the new matrix entries succeed before promoting to a real version.

---

## 9. Add `corvid self update`

A built-in upgrade command (rustup-style) means users never have to remember the install URL again.

### Steps (upstream feature)

This is a Corvid-lang change, not an installer change. Add a `corvid self` subcommand group in `crates/corvid-cli/src/cli/`:

- `corvid self update` — re-runs the platform-appropriate install script, replacing the current binary.
- `corvid self uninstall` — removes `~/.corvid` and PATH entries.
- `corvid self version` — shows version, install date, and source (prebuilt vs cargo-built vs brew vs scoop).

Implementation sketch: `update` shells out to `sh -c "curl -fsSL https://corvid.sh | sh"` on Unix and `powershell -c "irm https://corvid.sh | iex"` on Windows. Honors `CORVID_REPO` / `CORVID_VERSION`.

### Why this is worth it

Right now upgrading Corvid means re-finding the install URL. With `self update`, it's `corvid self update`. That's the difference between "I'll upgrade later" and "done."

Lower priority than #1–#5; consider it a v0.2.x feature.

---

## How to use this file

When you finish a task, swap `[ ]` for `[x]` in the [Checklist](#checklist) and commit:

```sh
git commit -am "FOLLOWUPS: complete <item name>"
```

When all items are done, archive this file (delete it, or rename to `FOLLOWUPS-archived-vX.Y.md`) so it's not stale clutter.

New follow-ups go at the bottom under their own `## N. Title` heading; add the matching checkbox at the top.
