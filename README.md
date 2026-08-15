# Manifest

A single-page tool that turns a Stremio debrid addon into a season download list.
Search a series, pick a season, and it probes every episode against your addon,
picks the best source for each, and hands the links off to JDownloader, aria2, or
the browser.

**Live:** https://lironb2003.github.io/Downloadio/

## Setup

Open the site, hit **settings**, and paste the configured addon URL you already use
in Stremio — for example:

```
https://torrentio.strem.fun/realdebrid=<your-api-token>/manifest.json
```

Your Real-Debrid token lives in that URL, so it is kept in the browser's
`localStorage` and never leaves the page. Nothing is committed to this repo, and
there is no server side.

## How sources are chosen

- Sources Real-Debrid has already cached rank above uncached ones. An uncached
  link doesn't return the episode — it redirects to a placeholder video until the
  debrid service finishes pulling the torrent — so those are never auto-selected.
- Within that, the source with the most seeders wins.
- **Prefer** and **Max size** only break ties between equally-seeded torrents.

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
