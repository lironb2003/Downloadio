# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Downloadio is a single static page (`index.html`, ~1350 lines) that turns a Stremio
debrid addon into a season download list. Search a series → pick a season → probe
every episode against the addon → pick the best source per episode → hand the links
to JDownloader (clipboard) or the browser (download queue).

## Build / run / deploy

There is no build step, no dependencies, no tests, and no package manager. All HTML,
CSS, and JS live inline in `index.html`.

Serve locally:

```bash
python -m http.server 8000
```

`file://` works but browsers disable `localStorage` in some contexts there, so
settings won't persist. The auto-download mode needs `showDirectoryPicker`
(Chromium over https), so it falls back to manual on `file://`.

Deploy: GitHub Pages serves `main` root directly — pushing to `main` publishes.
`.nojekyll` disables Jekyll. Repo settings → Pages is "Deploy from a branch",
`main`, `/ (root)`.

## Architecture

Everything is module-level state in one `<script>`; there is no framework and no
component tree. Rendering is manual: mutate state, then call the matching
`render*` function.

**External services** (no server side, no API keys in the repo):
- Cinemeta (`v3-cinemeta.strem.io`) — series search and episode metadata.
- Torrentio (`torrentio.strem.fun`) — stream lists. The addon URL is *derived*
  from `{provider, apikey}` in `localStorage`, never stored whole (`cfg.addon`).
- Optional user-supplied CORS proxy prefix (`via()`).

**State globals**: `SHOW`, `EPISODES`, `SEASON`, `SOURCE` (a pinned `bingeGroup`),
plus `RUN`/`ABORT` — `RUN` is a monotonic counter that invalidates stale probe
runs and `ABORT` drops their in-flight fetches. Any code path that leaves a season
must bump `RUN` and call `cancelProbes()`, or season switches stack live requests.

**Flow**: `search()` → `openShow()` → `renderShow()` → `selectSeason()` →
`probeSeason()` (4 concurrent workers over the episode list) → per-episode
`syncOne()` / `renderRow()` / `renderSources()` / `updateDock()`.

**Render functions and what they own**: `renderStrip()`/`clipStrip()` (the episode
pip grid, folded to 4 rows), `renderTable()`/`renderRow()`/`renderAlts()` (episode
rows), `renderSources()` (the season-wide source picker), `updateDock()` (bottom
bar), `renderQueue()` (download queue panel). `syncOne(i)` is the cheap path that
updates just one pip + checkbox without a re-render.

**localStorage keys** are all `mf.*`: `mf.provider`, `mf.apikey`, `mf.proxy`,
`mf.dlmode`, `mf.pref`, `mf.maxgb`, `mf.recent`. `mf.addon` is a legacy key
migrated on load. All access goes through the `store` shim, which falls back to an
in-memory object when `localStorage` throws.

## Domain rules that are easy to break

These encode behaviour of the debrid services, not preference — changing them
changes what users actually get:

- **Cache state is a hard sort key**, not a bonus (`tier()` before `seeders` before
  `rank()` in `pickStreams`). An uncached link does not 302 to the file — it serves
  a placeholder video until the service finishes pulling the torrent. Uncached
  episodes are never auto-selected (`e.on = pick.cached !== false`).
- **Seeders decide the pick**; the Prefer/Max-size settings in `rank()` only break
  ties between equally-seeded torrents.
- `parseStream()` reads cache state from `s.name` only — release *titles* contain
  words like "instant" and false-positive.
- Size uses the 💾-tagged value when present; release names carry stray sizes that a
  loose regex grabs first.
- Torrentio drops requests under load and the rejection has no CORS headers, so it
  surfaces as a bare `TypeError: Failed to fetch`. `getJSONRetry()` backs off and
  retries; don't collapse it back to `getJSON` for stream calls.
- Episodes are grouped into season-wide sources by `behaviorHints.bingeGroup`,
  falling back to the infohash in the resolve URL.

## Download queue

A page gets no completion callback for browser-managed downloads and the debrid CDN
sends no CORS headers, so the bytes can't be watched in-page either. Consequences
baked into the code:

- Each URL is handed to its **own hidden iframe**, not an anchor click. Debrid links
  are cross-origin, so `download` is ignored and a click is a top-level navigation —
  the next click replaced the previous one mid-headers and downloads went missing.
- Starts go through `qRelease()`, a single promise chain enforcing `QUEUE_GAP_MS`
  spacing, so concurrent triggers (next presses, the folder watcher, the initial
  fill) can't claim the same item or start back-to-back.
- Auto mode (`qTick`) diffs the watched folder against a snapshot taken *before*
  each trigger, and counts an episode done only when a non-temp file matching *its*
  name appears — with five transfers running, Chrome's `Unconfirmed N.crdownload`
  temps are indistinguishable. `next` stays available in auto mode so a missed
  detection can't strand the queue.
- `qBuild()` collapses items sharing a URL: a two-part finale in a season pack is
  one `.mkv` and shows as one row (`S10E17+E18`).

## Conventions

- Comments explain *why* a non-obvious rule exists (usually a debrid/browser quirk).
  Preserve them when touching that code; they are the only record of the failure
  that motivated it.
- DOM is built with `createElement` + `textContent` for anything containing
  addon-supplied text (release names, filenames); `innerHTML` is used only for
  static markup.
- CSS uses the `--void/--panel/--ink/--violet` custom-property palette at the top of
  the `<style>` block. Fonts are Familjen Grotesk (UI) and JetBrains Mono (`.mono`,
  all technical/metadata text).
