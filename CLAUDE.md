# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Downloadio is a single static page (`index.html`, ~1450 lines) that turns a Stremio
debrid addon into a download list. Search a movie or series → for a series pick a
season → probe every item against the addon → pick the best source per item → hand
the links to JDownloader (clipboard) or the browser (download queue).

A movie is carried as a season of exactly one episode (`loadMovie()`), so probing,
the alternatives list, subtitles, the dock and the queue all run unchanged. Only
the addon URL segment, the filename and the surrounding chrome differ.

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
- Cinemeta (`v3-cinemeta.strem.io`) — movie/series search and metadata. Movies and
  series are separate catalogs, so `search()` asks both (`allSettled`, so one being
  down still leaves the other useful) and merges. Cinemeta drops requests the same
  way Torrentio does, so its calls go through `getJSONRetry()` too — a dropped
  catalog would otherwise read as "this title has no movies".
- Torrentio (`torrentio.strem.fun`) — stream lists. The addon URL is *derived*
  from `{provider, apikey}` in `localStorage`, never stored whole (`cfg.addon`).
- OpenSubtitles v3 (`opensubtitles-v3.strem.io`) — subtitle lists, same addon
  protocol, no key. Chosen over the OpenSubtitles.com REST API because that one
  caps free accounts at a handful of downloads per day, which cannot serve a
  season.
- Optional user-supplied CORS proxy prefix (`via()`).

**State globals**: `SHOW` (`SHOW.type` is `"movie"` or `"series"`; `kind()`/`isMovie()`
read it), `EPISODES`, `SEASON` (`null` for a movie), `SOURCE` (a pinned `bingeGroup`),
plus `RUN`/`ABORT` — `RUN` is a monotonic counter that invalidates stale probe
runs and `ABORT` drops their in-flight fetches. Any code path that leaves a season
must bump `RUN` and call `cancelProbes()`, or season switches stack live requests.

**Flow**: `search()` → `openShow()` → `renderShow()` → `selectSeason()` (or
`loadMovie()`) → `probeSeason()` (4 concurrent workers over the episode list) →
per-episode `syncOne()` / `renderRow()` / `renderSources()` / `updateDock()`.

For a movie, `renderShow()` omits the season bar, the pip strip, the tools row and
the source panel — a single item has nothing for them to act on. Everything that
would fill them (`renderStrip`, `renderSources`, the `#btnAll` branch of
`updateDock`) bails on the missing element, so no caller needs a movie check.
The probe bar, `renderRow`, `fileName()` and `qCode()` do branch: a movie has no
`SxxExx`, so the row shows the release year and the file is named `Title.2010.mkv`.
The queue records `item.movie` at build time, because the queue panel outlives the
open title.

**Render functions and what they own**: `renderStrip()`/`clipStrip()` (the episode
pip grid, folded to 4 rows), `renderTable()`/`renderRow()`/`renderAlts()` (episode
rows), `renderSources()` (the season-wide source picker), `updateDock()` (bottom
bar), `renderQueue()` (download queue panel). `syncOne(i)` is the cheap path that
updates just one pip + checkbox without a re-render.

**localStorage keys** are all `mf.*`: `mf.provider`, `mf.apikey`, `mf.proxy`,
`mf.dlmode`, `mf.pref`, `mf.maxgb`, `mf.recent`, `mf.subs`, `mf.sublangs`.
`mf.addon` is a legacy key migrated on load. All access goes through the `store`
shim, which falls back to an in-memory object when `localStorage` throws.
`cfg.sublangs` distinguishes `null` (never set → the `heb,eng` default) from `""`
(the user deselected every language, which must not fall back).

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

## Subtitles

Probed in their own pass (`probeSubs`) *after* `probeSeason` finishes, sharing the
season's `AbortController` and the same `RUN` guard. It is deliberately separate:
the stream links are the product, and a slow or broken subtitle addon must never
delay them.

- `e.subs` holds **every** language the addon returned, unfiltered; `e.subPick` is
  derived from it by `pickSubs()`. So changing the language selection re-picks with
  no network request at all. `SUBS_RUN` tracks which `RUN` has already been
  fetched, so turning subtitles on mid-season fetches exactly once.
- A hand-picked subtitle goes in `e.subFixed[lang]` and survives a re-pick.
- `subRank()` scores candidates against the **already-chosen torrent's** release
  name, because a subtitle timed for a different rip drifts out of sync. When the
  addon returns no name field, every score ties and it falls back to the addon's
  own ordering (its download-count ranking) — an upgrade when metadata exists,
  never a regression when it doesn't.
- `normLang()` resolves 2-letter codes, English names, and OpenSubtitles' own
  extras (`pob`) onto one canonical 3-letter code, and passes unknown codes
  through rather than dropping them. The *filename* uses the 2-letter form —
  players look for `…he.srt`, not `…heb.srt`.
- Files are named from `fileName(e)` + `.{two-letter}.srt`, so they sit beside the
  video under its own name and get loaded automatically. Names are deduped: two
  episodes sharing one file (a two-parter in a season pack) share a subtitle name.
- **Subtitles never enter the download queue.** They are ~50KB from a host that
  isn't the debrid CDN, so none of the queue's spacing or completion-watching
  applies. `saveSubsFor()` fetches the bytes and then either writes into the
  watched folder or triggers a blob download.
- The blob download uses an anchor with `download=`, *not* the queue's hidden
  iframe. A `blob:` URL is same-origin, so `download` is honoured and the filename
  is ours; the iframe hack exists only for cross-origin debrid links and would
  hand naming back to the server. The iframe is the last-resort fallback for when
  the bytes can't be fetched at all (no CORS, no proxy).
- `watchFolder()` asks for `readwrite` instead of `read` when subtitles are on,
  because the page writes the `.srt` files into that folder itself.
- **`Q.dir` doubles as the subtitle destination, and the user's pick wins.** A
  page can't redirect a browser download, so the video lands wherever its
  downloader puts it (the browser's folder, or JDownloader's); the subtitle is
  the only file this page places itself. An earlier version second-guessed the
  pick and sent subtitles after the videos whenever the two disagreed — that
  moved files somewhere nobody chose. `subDest()` now just honours `Q.dir`, and
  `subToast()` always names where the files went.
- **Write permission must be taken at pick time.** `requestPermission()` needs
  user activation, and `saveSubsFor()` runs after `qFill()`, whose
  `QUEUE_GAP_MS` spacing far outlives the click. `watchFolder()` therefore grabs
  readwrite immediately after the picker resolves and caches it in `Q.canWrite`;
  `startQueue()` also resolves it before `qFill()` for a folder held from an
  earlier run.
- **`watchFolder(force)` can re-prompt.** Without `force` it returns early when a
  handle is already held, which meant the first folder picked in a page session
  stuck for good. The settings control passes `force`.
- `renderSubDir()` reads `Q.dir` through `heldDir()`, which try/catches the
  temporal dead zone: the config wiring runs before `const Q` is initialised, and
  `typeof Q` does *not* guard a TDZ read — it throws, and an uncaught throw there
  takes the rest of the settings wiring down with it.

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
