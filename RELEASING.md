# Releasing ForgePlayer → the website

How a ForgePlayer build reaches this site, and the gotchas that bit us the
first time we landed it (v0.0.5, 2026-06-17). Read this before cutting a
release.

---

## The pipeline (end to end)

```
push tag vX.Y.Z              (in liquid-releasing/forgeplayer)
   │
   ▼
release.yml                  builds Windows + macOS + Linux,
   │                         compiles the NSIS installer (Setup.exe)
   ▼
publishes a GitHub Release   → liquid-releasing/forgeplayer-RELEASES
   │                           (assets: Setup.exe, windows.zip, macos.zip, linux.tar.gz)
   ▼
repository_dispatch          event_type = "release-published", client_payload.tag
   │
   ▼
sync-version.yml             (in THIS repo) reads the release and
   │                         commits latest-version.json  → the version badge
   ▼
Cloudflare Pages deploy      (wrangler) serves index.html + latest-version.json
```

Two repos hold the release; this repo only holds the **marketing site + the
version badge**. The binaries live in `forgeplayer-releases`.

---

## How the site serves downloads

- **Version badge / hero text** ← `latest-version.json` in this repo, written
  by `sync-version.yml` from the dispatched tag. Updates for **any** release.
- **Download buttons** ← hardcoded GitHub "latest" redirect URLs in
  `index.html`:
  `https://github.com/liquid-releasing/forgeplayer-releases/releases/latest/download/<asset>`

`/releases/latest/` is a server-side redirect to the newest release's assets —
no per-release URL edit needed, **as long as the release is the "Latest" one**.

---

## ⚠️ THE GOTCHA: pre-release hides the cut from the website

**GitHub's `/releases/latest/` EXCLUDES pre-releases.** If a release is
published with `prerelease: true`:

- the **badge updates** to the new version (sync passes the explicit tag), but
- the **Download buttons still serve the previous full release** — because
  `/releases/latest/` skips the pre-release.

Result: the site says "vX" but clicking Download gives you "vX-1". That mismatch
is exactly what bit us on v0.0.5 (badge said 0.0.5, buttons served 0.0.4).

### The rule we ship by

**Publish as a normal full "Latest" release** (`prerelease: false`). Convey
beta status with the site's standing **"Pre-release software" note** (in
`index.html`) + the **body banner** in the release notes — NOT with GitHub's
prerelease flag. This is how the alpha shipped, and how every cut should.

`release.yml` is set to `prerelease: false` for this reason — don't flip it.

### If a release accidentally went out as a pre-release

```bash
gh release edit vX.Y.Z --repo liquid-releasing/forgeplayer-releases \
  --prerelease=false --latest
# then re-check:
gh api repos/liquid-releasing/forgeplayer-releases/releases/latest --jq .tag_name
```

The badge needs no action — `sync-version.yml` already wrote the right tag.

---

## Build gotchas (the NSIS installer step, first-time pain)

The Windows installer (`ForgePlayer-Setup.exe`) is built in `release.yml` and
took three tries to land. If it breaks again, it's almost certainly one of:

1. **NSIS isn't preinstalled on `windows-latest`.** The workflow installs it:
   `choco install nsis -y --no-progress` (Chocolatey is on the runner).
2. **makensis resolves relative paths against the SCRIPT's directory, not the
   working dir.** So `branding\...` / `dist\...` in `installer\forgeplayer.nsi`
   wouldn't resolve. Fix: the workflow passes the **absolute repo root** as
   `/DROOTDIR=<abs>` and the `.nsi` builds every asset path from `${ROOT}`.
   Don't reintroduce bare relative paths or `${__FILEDIR__}` (it expands
   relative and gets re-anchored → doubled path).

`Publish Release` is gated on all three platform builds passing, so an
installer failure blocks the whole release (nothing partial ships) — which is
the safe behavior, but means the installer step must be green.

---

## TODO / known gaps

- **The site only links the portable `windows.zip`, not `Setup.exe`.** The
  installer (the double-click `.forge` association) is the headline of v0.0.5
  but isn't surfaced on the site yet. Add a "Download installer (.exe)" button
  pointing at
  `…/releases/latest/download/ForgePlayer-Setup.exe`.
- Consider deriving the download URLs from `latest-version.json`'s tag in JS so
  the site could serve a pinned/pre-release tag if we ever needed to — today it
  relies on the release being "Latest".

---

## Cut-a-release checklist

1. Land all changes on `forgeplayer` `main`; tests green.
2. Refresh the "What's new" bullets in `release.yml` (release body).
3. **Check the Discord invite hasn't lapsed** — see below. `release.yml`
   writes it into the release body, so a dead invite ships permanently.
4. Tag and push: `git tag -a vX.Y.Z -m "ForgePlayer X.Y.Z (beta)" && git push origin vX.Y.Z`.
5. Watch the run; the Windows **NSIS** step is the fragile one.
6. Confirm the release is **Latest, not pre-release**:
   `gh api repos/liquid-releasing/forgeplayer-releases/releases/latest --jq .tag_name`
7. Confirm the badge: `latest-version.json` in this repo shows the new tag
   (sync-version commits it automatically).
8. Verify on the live site: badge = new version, Download buttons fetch the new
   binaries, and the Discord link in the nav still resolves.

---

## The Discord invite (checklist step 3)

One Discord server backs every Forge app, and the invite currently in use is
a **timed** one: `UHdJFhEZF`, expiring **2026-09-30**. That matters at release
time because the invite is not only on this site — it is written into the
GitHub Release body by `release.yml` and compiled into shipped builds
elsewhere in the family. Neither can be corrected after the fact.

Check it before tagging:

```bash
code=$(git grep -hoE 'discord\.gg/[A-Za-z0-9]+' | head -1 | cut -d/ -f2)
curl -sS "https://discord.com/api/v10/invites/$code?with_expiration=true"
```

- `"expires_at": null` — permanent invite, proceed.
- A date after the next expected cut — proceed.
- A date before it, or `"Invite is expired."` — **stop.** Get a fresh invite
  (a never-expiring one ends this whole class of problem) and roll it across
  every forge repo before tagging. It lives in `README.md`, `index.html`,
  `mkdocs.yml`, `docs/**`, `about.py`, `ui.py`, and the release-notes body in
  `.github/workflows/release.yml`.

This site deploys on push to `main`, so a link fix here is live within a
minute; verify with `curl -sL https://forgeplayer.app | grep discord.gg`.

To **re-run** after a CI fix, move the tag:
`git push origin :refs/tags/vX.Y.Z && git tag -d vX.Y.Z && git tag -a vX.Y.Z -m ... && git push origin vX.Y.Z`
