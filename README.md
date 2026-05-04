# forgeplayer-web

Static landing site for ForgePlayer. Deployed to **forgeplayer.app** via
Cloudflare (Workers Assets).

## Structure

- `index.html` — single-page site: hero, features, how-it-works, requirements,
  cross-links to the rest of the Liquid Releasing family, download CTA.
- `latest-version.json` — version-badge data. Updated automatically by the
  `.github/workflows/sync-version.yml` workflow when a new release lands on
  `liquid-releasing/forgeplayer-releases`.
- Image assets — product branding (`forgeplayer_*`), forge-metaphor icons
  shared with the FunscriptForge / ForgeAssembler sites (`anvil.png`,
  `oven.png`, etc.), Liquid Releasing badge (`Icon-Only-White.svg`).

## Local preview

```bash
# any static server works — python stdlib is fine:
python -m http.server 8080
# open http://localhost:8080
```

## Deployment

Cloudflare Workers Assets watches the `main` branch of this repo. Every push
auto-builds and deploys to `forgeplayer.app`. No build step — the site is
static HTML + assets.

## Release version badge

The hero's "version" line reads from `latest-version.json` via a small inline
fetch. When the `forgeplayer` release workflow publishes a new tag, it fires
a `repository_dispatch` event against this repo; `sync-version.yml` catches
it, writes the new tag into `latest-version.json`, and commits to `main`.
Cloudflare picks up the new commit and the badge updates within a minute.

## Cross-links

This site cross-links to:

- [funscriptforge.com](https://funscriptforge.com) — FunscriptForge
- [forgeassembler.app](https://forgeassembler.app) — ForgeAssembler
- ForgeYT release artifacts on GitHub

## License

Site content © 2026 Liquid Releasing. ForgePlayer itself is MIT-licensed —
see the [main repo](https://github.com/liquid-releasing/forgeplayer).
