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

## How sources are chosen

- Sources the debrid service has already cached rank above uncached ones. An
  uncached link doesn't return the episode — it redirects to a placeholder video
  until the service finishes pulling the torrent — so those are never
  auto-selected.
- Within that, the source with the most seeders wins.
- **Prefer** and **Max size** only break ties between equally-seeded torrents.

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
