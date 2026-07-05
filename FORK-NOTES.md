# Fork maintenance notes

Personal fork of FlowMouse. Upstream declined the search-engine/menu PR
(they want to keep the extension lightweight and gesture-focused), so this
feature lives here. These notes keep future upstream updates cheap to merge.

## Remotes

- `origin`   → your fork  (`PPP01/FlowMouse`)
- `upstream` → maintainer (`Hmily-LCG/FlowMouse`)

## Branch roles

| Branch | Role | Keep it… |
|---|---|---|
| `main` | Pristine mirror of `upstream/main`. Never commit here. | fast-forward only |
| `feature/search-engine-suite` | **Your personal build.** All the feature work squashed into one commit on top of `upstream/main`, plus this notes commit. This is the branch you load as the unpacked extension. | rebased on `upstream/main` |
| `feature/search-links` | Full 130-commit development history + the plans/recon docs + the personal-engines migration snippet (`docs/dev/`). Archive — don't rebase. | untouched |
| `firefox-build` | **Firefox build.** `feature/search-engine-suite` + 3 Firefox commits (Firefox manifest, `menu-patterns.js` via `background.scripts` + `importScripts` guard, Chrome-only entries dropped). Load this in Firefox. See "Firefox" below. | rebased on `feature/search-engine-suite` |
| `firefox-test` | Old Firefox branch (built on the 130-commit history). Backup — safe to delete once `firefox-build` is confirmed. | — |

## Updating from upstream

```bash
git fetch upstream

# 1. keep main a clean mirror
git checkout main
git merge --ff-only upstream/main
git push origin main

# 2. replay your feature onto the new upstream (only ~2 commits → tiny rebase)
git checkout feature/search-engine-suite
git rebase upstream/main
#   resolve conflicts only in files both sides touched, then:
git push --force-with-lease origin feature/search-engine-suite

# 3. (optional) replay the Firefox build on top of the updated feature branch
git checkout firefox-build
git rebase feature/search-engine-suite
```

Because the feature is a **single squashed commit**, conflicts are localized
and rare. Do *not* rebase `feature/search-links` (130 commits = pain); it is
only kept for reference.

## Personal (German) search engines

The neutral catalog ships without region-specific engines. Your own German
engines and `.de` domains live in the browser's synced settings, restored
once via the console snippet:

    docs/dev/migrate-personal-engines.snippet.js   (on the feature/search-links branch)

They are stored as settings data, not code, so they never affect a rebase.

## Firefox

`firefox-build` swaps the Chrome service-worker manifest for a Firefox one.
`web-ext lint` → 0 errors. Known gaps (Firefox lacks the APIs): the JS-transform
sandbox (`offscreen`), engine favicons (`favicon`), and save-as-MHTML
(`pageCapture`). The core search/menu features work.

### npm scripts (run on the `firefox-build` branch)

Run `npm install` once. Signing reads credentials from the environment
(`WEB_EXT_API_KEY` / `WEB_EXT_API_SECRET`) — never commit them.

| Script | What it does |
|---|---|
| `npm run ff:run` | Launch Firefox with the extension, live-reload on save. Dev only — no signing, no version bump. |
| `npm run ff:build` | Build an **unsigned** `.zip` into `web-ext-artifacts/` (runtime files only, via `web-ext-config.mjs`). |
| `npm run ff:bump` | Bump `manifest.json` version (`2.2` → `2.2.1`, then `2.2.2`, …). |
| `npm run ff:sign` | Upload to Mozilla and download a **signed** `.xpi` (channel `unlisted`). |
| `npm run ff:release` | `ff:bump` then `ff:sign` — the one-shot release command. |

A permanent install always needs a **signed** `.xpi` in regular Firefox. (Dev
Edition / Nightly / ESR can install unsigned after setting
`xpinstall.signatures.required = false` in `about:config`.)

### Routine: ship a new version

```bash
git checkout firefox-build
git rebase feature/search-engine-suite      # only if you changed the feature
export WEB_EXT_API_KEY=user:XXXX:XX          # from addons.mozilla.org → Manage API Keys
export WEB_EXT_API_SECRET=XXXX
npm run ff:release                           # bumps version, signs, writes web-ext-artifacts/*.xpi
```

Then install the signed `.xpi` once via **about:addons → gear → Install
Add-on From File**. AMO refuses to sign a version it already signed — that's
why `ff:release` bumps first.

### Auto-update (install the signed `.xpi` only once, ever)

`manifest.json` already points `browser_specific_settings.gecko.update_url`
at `updates.json` on this branch (raw GitHub URL). To make a release
auto-reach your browser:

1. `npm run ff:release` (bumps + signs).
2. Create a GitHub release on your fork and upload the signed `.xpi` as an
   asset (e.g. tag `ff-2.2.1`).
3. Add an entry to `updates.json` — `version` = new manifest version,
   `update_link` = the exact release asset URL — and push `firefox-build`:
   ```bash
   git commit -am "release ff-2.2.1" && git push origin firefox-build
   ```

Firefox polls `updates.json`, sees the higher version, and updates itself —
no more manual re-install. (For this to work, `firefox-build` must be pushed
to `origin` so the raw `update_url` resolves.)

## If you ever re-attempt an upstream PR

Exclude `docs/plans/`, `docs/dev/`, and `FORK-NOTES.md` — they must not ship
upstream. Base the PR branch directly on `upstream/main`.
