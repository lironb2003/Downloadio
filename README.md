# Manifest

A single-page tool that turns a Stremio debrid addon into a season download list.
Search a series, pick a season, and it probes every episode against your addon,
picks the best source for each, and hands the links off to JDownloader, aria2, or
the browser.

**Live:** https://lironb2003.github.io/Downloadio/

## Setup

Open the site, hit **settings**, pick your debrid provider, and paste its API key.
The **Find … API Key** link goes straight to that provider's key page. Supported:
Real-Debrid (default), AllDebrid, Premiumize, Debrid-Link, TorBox, and Offcloud.

The key is kept in the browser's `localStorage` and is only used to build the
Torrentio addon URL. Nothing is committed to this repo, and there is no server
side. Pasting a full Stremio addon URL into the key box also works — the provider
and key are read back out of it.

## Recently opened

Series you actually open are remembered (searching alone doesn't count) and shown
as poster cards under the search box whenever nothing else is on screen. The last
12 are kept, newest first, in `localStorage`.

## How sources are chosen

- Sources the debrid service has already cached rank above uncached ones. An
  uncached link doesn't return the episode — it redirects to a placeholder video
  until the service finishes pulling the torrent — so those are never
  auto-selected.
- Within that, the source with the most seeders wins.
- **Prefer** and **Max size** only break ties between equally-seeded torrents.

## Download queue

**Download** runs five at a time, oldest episode first — handing the browser
everything at once just gets you about six parallel transfers splitting your
bandwidth. Two modes, chosen in settings:

- **Auto** watches the folder your browser saves into and releases the next
  episode as each file lands, keeping five going. It asks once for the folder
  (read-only) and needs the File System Access API, so Chrome or Edge over https;
  opening `index.html` straight off disk may not qualify. If it isn't available
  that run falls back to manual and says so.
- **Manual** starts five and holds the rest; each press of **next** releases one
  more.

Detection is deliberately narrow, because a page gets no completion callback for
downloads the browser manages and the debrid CDN sends no CORS headers, so the
bytes can't be watched here either. With five transfers running, the in-progress
temp files can't be told apart — Chrome names them `Unconfirmed 123456.crdownload`
— so an episode counts as finished only when a file with *its* name appears that
wasn't there when it started. Progress percentages show up only when the browser
happens to name the temp file after the real one. **next** is available in auto
mode too, so a missed detection never strands the queue.

Episodes that share a file are downloaded once: a two-part finale in a season
pack is a single `.mkv`, and the queue shows it as one row (`S10E17+E18`).

For a hands-off run, the aria2 export with `-j1` downloads them sequentially
outside the browser.

## One source for a whole season

Episodes are matched to their torrent via the addon's `bingeGroup`, so
**one source for all…** lists the releases that cover the season — usually season
packs — with how many episodes each one covers, its seeders, quality, and average
size per episode. Picking one switches every episode to that release; anything it
doesn't cover keeps its own best source and is marked. **best per episode** puts
it back.

## Deploying

GitHub Pages serves `index.html` straight from the `main` branch root. There is no
build step and no dependencies — push to `main` and the site updates.

Repo settings → Pages is set to **Deploy from a branch**, `main`, `/ (root)`.

`.nojekyll` disables Jekyll processing, which this static file doesn't need.

## Local use

Open `index.html` in a browser, or serve the folder over HTTP:

```
python -m http.server 8000
```

Opening the file over `file://` works too, but browsers disable `localStorage`
there in some contexts, so your settings won't persist between reloads.
