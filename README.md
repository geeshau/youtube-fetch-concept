# 🎵 YouTube Fetch Concept

> Paste a YouTube playlist link (or just search by name) and get every item back as a song row — with one-click audio & video downloads.

[![Next.js](https://img.shields.io/badge/Next.js-13.5.4-black?logo=next.js)](https://nextjs.org)
[![No API Key](https://img.shields.io/badge/YouTube_API-Not_Required-green)]()
[![License](https://img.shields.io/badge/license-MIT-blue)]()

---

## ✨ What it does

- 🔍 **Search by name** or **paste a playlist URL** — works with regular playlists, search results, and even YouTube's auto-generated "Mix" (RD…) radios.
- 📋 **Every item becomes a row** with thumbnail, channel, views, duration, and published date.
- ⬇️ **One-click downloads** — audio (192 kbps MP3) or video (MP4, 720p cap) for any single song.
- 📦 **Bulk ZIP downloads** — grab the whole playlist as one archive, with one unavailable video skipped instead of failing everything.
- 📊 **Live progress** — per-song ticks, running-song highlight, monotonic overall bar, cancel anytime.
- ✅ **Row selection** — pick specific songs and download just those.
- 📱 **Mobile-first** — responsive ladder from 380px phones up to wide desktops.
- ♾️ **Infinite scroll** — rows load more as you scroll down.
- 🔑 **No YouTube Data API key** — scrapes the public page + continuation tokens.

---

## 🚀 Running it

```bash
npm install
npm run dev        # http://localhost:3000
```

Build for production:

```bash
npm run build
npm run start
```

> ⚠️ **GitHub Pages won't work** — this app needs server-side API routes.
> Deploy to [Vercel](https://vercel.com) (one-click) or any Node host instead.

---

## 🏗️ How it works

No YouTube Data API key required. The app reads the public playlist/search page, pulls the embedded `ytInitialData` JSON out of the HTML, then follows YouTube's own continuation tokens to page through the rest.

```
pages/index.js               UI — search/playlist toggle, song rows, downloads
pages/api/playlist.ts        GET  /api/playlist?url=...
pages/api/search.ts          GET  /api/search?query=...
pages/api/download/start.ts  POST /api/download/start
pages/api/download/status.ts GET  /api/download/status?jobId=
pages/api/download/file.ts   GET  /api/download/file?jobId=
pages/api/download/cancel.ts POST /api/download/cancel?jobId=
lib/youtube.ts               Playlist + search fetching & parsing
lib/download-jobs.ts         Background download jobs (yt-dlp), on-disk state
lib/download-client.js       Browser helper: start → poll → download
lib/download-utils.js        Filename sanitising (shared by server & tests)
lib/search-utils.js          Input classification (URL vs search term)
lib/pagination-utils.js      Page slicing + page-number strip
lib/batch-utils.js           Per-song list helpers (markers, windowing)
styles/globals.css           Dark theme, song rows, progress, responsive
```

---

## 📡 API

```
GET  /api/playlist?url=https://www.youtube.com/playlist?list=PL...
GET  /api/search?query=hulvey
```

```jsonc
{
  "playlistId": "PL...",
  "title": "Top Trending Videos of the Week",
  "channel": "YouTube",
  "videoCount": 19,
  "videoCountLabel": "19 videos",
  "videos": [
    {
      "id": "Vh4O04Bpovw",
      "title": "I Explored A $200,000,000 Forgotten Space Colony",
      "channel": "Yes Theory",
      "channelAvatar": "https://yt3.ggpht.com/...",
      "duration": "16:09",
      "views": "7.4M",
      "publishedAt": "6y ago",
      "thumbnail": "https://i.ytimg.com/vi/Vh4O04Bpovw/hqdefault.jpg",
      "url": "https://www.youtube.com/watch?v=Vh4O04Bpovw"
    }
  ]
}
```

Search results use the same shape with `"isSearch": true` and a synthetic `playlistId` of `search:<query>`.

### Accepted inputs

- `https://www.youtube.com/playlist?list=PL...`
- `https://www.youtube.com/watch?v=...&list=PL...`
- A bare playlist ID (`PL...`)
- A bare video URL — resolves to that video's playlist/mix
- Any name ("hulvey") — switches to search mode

---

## 🎧 Downloads

Downloads run as **background jobs** with live progress instead of one long request.

```
POST /api/download/start           { videoId | url | playlistId, title, format, limit? }
  -> 202 { jobId, status, count? }

GET  /api/download/status?jobId=   -> { phase, percent, message, bytes, fileName, items? }
GET  /api/download/file?jobId=     -> the file, as an attachment
POST /api/download/cancel?jobId=   -> { cancelled }
```

`items` is only present for batch jobs — one entry per song:
`{ id, title, state, percent, bytes?, error? }`, where `state` ∈ `pending | downloading | processing | done | skipped`.

`format` = `audio` (192 kbps MP3) or `video` (MP4, ≤720p). `limit` caps how many playlist items a batch downloads.

The browser starts a job, polls every 700 ms, shows a progress bar on the row, then triggers a native browser download once the file is ready. Files are served as `attachment`s — a 200 MB video never sits in memory or a blob.

A running job can be cancelled from the UI, which kills the yt-dlp process and discards partial files.

**Playlist mode** downloads every item into a ZIP. One unavailable video is skipped rather than failing the whole batch, and the final message reports how many of the total were retrieved.

### 🎵 The per-song list

A batch can cover hundreds of songs, so a single percentage bar says very little. While a batch runs, the UI shows the whole queue and ticks songs off one by one:

```
●  Please, Just Drive Me Home"—Tired Nurse…      Downloading 24%
✓  indie tribe - ROB HELL (Official Music Video)  Done
✓  Hulvey, Torey D'Shaun, Alex Jean — Love Like…  Done
○  IN CHRIST | Blessed in Christ | Ephesians 1:3   Queued
```

| Marker | Meaning |
| --- | --- |
| `●` | The song being downloaded/converted right now |
| `✓` | Finished and in the ZIP |
| `✕` | Unavailable and skipped |
| `○` | Still queued |

The running song also gets its own line above the list with its own percentage, so it's never scrolled out of sight.

The per-song list comes from the server: `runBatch` keeps one entry per song and publishes it in the job status. It's written to its own file (`<jobId>.items.json`) so the status route stays cheap to poll no matter how long the playlist is. Progress updates are throttled to one write every 400 ms; state changes (finish/skip/start) write immediately.

The overall bar is **monotonic** — it never moves backward. Each song owns a slice (`100 / songCount` %) and every phase of that song stays inside its slice, so the bar creeps forward steadily instead of jumping to 99% and back to 0% at every song boundary.

`lib/batch-utils.js` holds the pure helpers the UI shares with the unit tests: markers, state labels, counts, and the windowing that keeps a 500-song list from re-rendering in full on every poll.

### Why it's built this way

The first implementation did the whole download inside a single API request and only then started sending bytes. Two visible problems: the browser sat on a blank page for minutes with no feedback, and any failure dumped the user onto a raw JSON error page that looked like "nothing downloaded."

Two further details worth knowing if you change this code:

- **Job state lives on disk**, in `<tmp>/youtube-fetch-concept/jobs/`. Next.js compiles every API route into its own bundle, so an in-memory `Map` isn't shared between `/api/download/start`, `/api/download/status`, and `/api/download/file` — the status route would answer 404 for a job it never heard of.
- **Every job gets its own directory**, and every batch item gets its own sub-directory. A shared directory plus stem-prefix matching once let a video job pick up a leftover `.m4a` from a previous audio job.

YouTube also blocks some public videos for yt-dlp's default player client (`HTTP Error 403: Forbidden`). `lib/download-jobs.ts` retries the same URL with alternate player clients (`android`, `ios`, `tv`, `web_safari`) before giving up.

---

## 🎧 Mix (radio) playlists

YouTube's auto-generated mixes (`RD…`, e.g. `RDdUVs5Ocou8I`) are supported too:

- `https://youtube.com/playlist?list=RDdUVs5Ocou8I`
- `https://www.youtube.com/watch?v=dUVs5Ocou8I&list=RDdUVs5Ocou8I`

These need a different source: YouTube answers the mix's playlist page with *"This playlist type is unviewable."*, so the items are read from the mix rail on the watch page instead. A mix is randomized per request, so the exact list (and order) can differ between loads. Mixes are finite — no continuation endpoint — so you get the ~20 videos YouTube suggests.

---

## 📱 Responsive layout

Built mobile-first, adapting in `styles/globals.css`:

| Width | Layout |
| --- | --- |
| > 900px | Three columns: 180px thumbnail · title · 190px action stack |
| ≤ 900px | Tracks shrink to 148px / 168px so the title keeps a readable column |
| ≤ 720px | Actions move under the title and span the full row; buttons side by side |
| ≤ 560px | Compact list: 120px poster beside the title, 44px touch targets |
| ≤ 380px | Poster shrinks to 96px and type scales down |

Other details:

- Thumbnail is always 16:9 (fixed height on desktop, `padding-bottom: 56.25%` on phones).
- The two fixed grid tracks live in `--thumb-w` / `--actions-w` custom properties on `.song-row`, so a breakpoint only overrides two numbers.
- `@media (pointer: coarse)` raises hit areas to 44px on touch devices.
- `@media (prefers-reduced-motion: reduce)` disables the hover lift and progress-bar transition.
- Container respects `env(safe-area-inset-*)` so notched phones don't clip content.

At 360px a song row is ~158px tall instead of the 407px it used to be when the thumbnail spanned the full width.

The batch song list follows the same ladder: scroll window shrinks from 260px to 200px on a phone, and the running song's title clamps to two lines so a long name can't push the list off-screen.

---

## 🧪 Tests

```bash
npm test
```

Unit tests cover `lib/pagination-utils.js`, `lib/batch-utils.js`, `lib/search-utils.js`, `lib/download-utils.js`, and `lib/download-client.js`.

---

## 📄 License

MIT — do whatever you want with it. A ⭐ on GitHub is appreciated but not required.
node --test lib/download-utils.test.js lib/search-utils.test.js lib/download-client.test.js lib/pagination-utils.test.js lib/batch-utils.test.js
```

There are also end-to-end scripts in `scripts/` that drive a running server:

```bash
npm run build && npx next start -p 3107

node scripts/test-job-flow.js       # single audio + video + a failure case
node scripts/test-batch-flow.js     # playlist -> ZIP
node scripts/test-batch-items.js    # per-song list: states, ticks, ordering
node scripts/test-cancel.js         # cancel a running job
```

They read `BASE_URL` from the environment (default `http://localhost:3107`),
and `scripts/test-batch-flow.js` accepts `LIMIT` and `PLAYLIST_ID`.

## Notes

- The YouTube Data API is capped at 50 items per request and needs a key.
  This approach walks the playlist's continuation tokens instead, so it returns
  the whole list (a 1653-video channel playlist loads in ~10s).
- `lib/youtube.ts` depends on YouTube's internal page structure. If YouTube
  changes it, `extractItems()` in that file is the place to update.
- `.env` is not required. If you want to add a `YOUTUBE_API_KEY` later, it is
  not read by the current implementation.
