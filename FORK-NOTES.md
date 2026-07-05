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

### Packaging / permanent install

A temporary add-on (`about:debugging`) disappears on Firefox restart. For a
build a `.zip`/`.xpi`:

```bash
git checkout firefox-build
npx web-ext build --overwrite-dest   # writes web-ext-artifacts/*.zip (rename to .xpi)
```

A permanent install needs a **signed** `.xpi`. Two paths:

- **Regular Firefox** installs only signed extensions. Sign it as an *unlisted*
  add-on via a free Mozilla add-on account + API key:
  `npx web-ext sign --channel=unlisted --api-key=… --api-secret=…`
  → produces a signed `.xpi` installable in normal Firefox.
- **Developer Edition / Nightly / ESR** can install an unsigned `.xpi` after
  setting `xpinstall.signatures.required = false` in `about:config`.

## If you ever re-attempt an upstream PR

Exclude `docs/plans/`, `docs/dev/`, and `FORK-NOTES.md` — they must not ship
upstream. Base the PR branch directly on `upstream/main`.
