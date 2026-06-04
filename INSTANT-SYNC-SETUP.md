# Optional: instant upstream-sync trigger (Corvid-lang side)

This repo already auto-syncs `install.sh` / `install.ps1` / `release.yml` from
`Micrurus-Ai/Corvid-lang` on a **daily schedule** (see
[`.github/workflows/sync-installers.yml`](./.github/workflows/sync-installers.yml)).
That alone keeps the mirror current within a day, with zero setup.

If you want **instant** updates (mirror refreshes within ~30s of an upstream
change instead of waiting for the daily run), add the workflow below to the
**Corvid-lang** repo. It fires a `repository_dispatch` at this repo whenever the
canonical install files change on `main`, which the `repository_dispatch:
corvid-lang-install-changed` trigger in `sync-installers.yml` is already wired to
receive.

This is **purely optional** — the daily schedule covers correctness; this only
reduces latency.

---

## Step 1 — Create a token for the cross-repo ping

The default `GITHUB_TOKEN` inside a Corvid-lang workflow **cannot** dispatch to a
different repo, so you need a small scoped token.

1. https://github.com/settings/personal-access-tokens → **Generate new token**
   (fine-grained).
2. **Resource owner:** `Micrurus-Ai`.
3. **Repository access** → **Only select repositories** → pick **only**
   `Micrurus-Ai/corvid-installer`.
4. **Repository permissions** → **Contents: Read and write** (this is what the
   "Create a repository dispatch event" API requires; *Metadata: Read* is added
   automatically).
5. Generate, copy the `github_pat_…` value.
6. In **Corvid-lang** → **Settings → Secrets and variables → Actions → New
   repository secret**:
   - Name: `INSTALLER_SYNC_TOKEN`
   - Value: the PAT.

> Set a calendar reminder to rotate it before it expires — like
> `MANAGERS_TOKEN`, an expired PAT makes this silently stop firing (the daily
> schedule still keeps the mirror correct, so it degrades gracefully).

---

## Step 2 — Add the workflow to Corvid-lang

Create `.github/workflows/notify-installer-mirror.yml` in **Corvid-lang**:

```yaml
name: notify-installer-mirror

# When the canonical install files change on main, ping
# Micrurus-Ai/corvid-installer so its sync-installers workflow refreshes the
# mirror immediately instead of waiting for its daily schedule.
on:
  push:
    branches: [main]
    paths:
      - "install/install.sh"
      - "install/install.ps1"
      - ".github/workflows/release.yml"

# We authenticate to the OTHER repo with INSTALLER_SYNC_TOKEN, so this job's
# own GITHUB_TOKEN needs no permissions.
permissions: {}

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Dispatch corvid-installer sync
        env:
          GH_TOKEN: ${{ secrets.INSTALLER_SYNC_TOKEN }}
        run: |
          gh api repos/Micrurus-Ai/corvid-installer/dispatches \
            -f event_type=corvid-lang-install-changed
```

`gh` is preinstalled on GitHub-hosted runners, so no install step is needed.

### curl equivalent (if you'd rather not use `gh`)

```bash
curl -fsSL -X POST \
  -H "Authorization: Bearer ${{ secrets.INSTALLER_SYNC_TOKEN }}" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/Micrurus-Ai/corvid-installer/dispatches \
  -d '{"event_type":"corvid-lang-install-changed"}'
```

---

## Step 3 — Verify the round-trip

1. Make a trivial change to `install/install.sh` in Corvid-lang on `main`
   (e.g. tweak a comment) and push.
2. Corvid-lang **Actions** tab → `notify-installer-mirror` should run and
   succeed in a few seconds.
3. corvid-installer **Actions** tab → `sync-installers` should fire
   (event: `repository_dispatch`) within a few seconds and either commit the
   change or report "Already in sync."
4. Confirm the mirrored file in corvid-installer matches the upstream edit.

If step 2 fails with a 403/404, the token scope is wrong — re-check that
`INSTALLER_SYNC_TOKEN` has **Contents: write** on `corvid-installer` and that
the repo is in its selected-repositories list.

---

## How the two pieces fit together

```
Corvid-lang push (install/** or release.yml)
        │
        ▼
notify-installer-mirror.yml  ──repository_dispatch──▶  corvid-installer
  (this doc, Corvid-lang side)   event:                  sync-installers.yml
                                 corvid-lang-install-       (already live here)
                                 changed                         │
                                                                 ▼
                                                   fetch + guard + commit-if-changed

Daily schedule (06:17 UTC) drives the same sync-installers.yml regardless,
so the mirror stays correct even if this dispatch path is never set up.
```
