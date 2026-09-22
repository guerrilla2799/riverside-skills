---
name: podcast-host-and-rss
description: >-
  Upload to Transistor, Buzzsprout, Captivate, "update the RSS feed", loudness, ID3 tags. Downloads the Riverside export, preps it with ffmpeg, uploads to the host, keeps the GUID stable. Editing the cut: podcast-episode-pipeline.
---

# Podcast Host and RSS

The seam between Riverside and the podcast feed. The Riverside MCP renders the episode, but it gives no download link and does not manage any podcast host, Riverside's own hosting included. This skill takes the finished export out of the Riverside exports folder, prepares the file locally with ffmpeg, puts it on the host by hand or through the host's API, and checks the RSS metadata that podcast apps read. One rule outranks the rest: an episode's GUID never changes, including when its audio is replaced.

## When to use
- An approved episode is exported and needs to reach Transistor, Buzzsprout, Captivate or Riverside hosting
- A re-cut needs the audio replaced on an episode already in the feed (`podcast-recut-and-republish` sends it here)
- Loudness, ID3 tags, cover art or chapter embedding on a local file
- An RSS metadata check before or after release

Not trims, cards, captions or chapters inside the edit: those happen in Riverside's editor through `podcast-episode-pipeline`. Not YouTube or social posts: `distribution-and-scheduling`.

## Inputs
- A `COMPLETED` export of the canonical revision, from `podcast-episode-pipeline`. Its export id is in `publish-log.md`
- Titles, show notes and the chapter list, from `podcast-show-notes-and-chapters`
- Episode number, season number, episode type and the explicit flag, from the user
- Episode art if the episode has its own. Square. Apple asks for 1400 to 3000 px
- Host access: a login for a manual upload, or an API key in an environment variable for an API upload. Never write a key into any file
- `ffmpeg` and `ffprobe` installed locally
- Scripts live at `../riverside-skills/scripts/`, relative to this skill's base directory. Run them from the folder that holds `workspace/`, or set `RIVERSIDE_WORKSPACE`

## Workflow

1. **Staleness gate.** Run `../riverside-skills/scripts/stale-check SLUG`. Exit 2: STOP and report the missing file. Exit 1: STOP if any input to this upload (titles, show notes, chapter list) is among the lines it printed, and route that asset to its owner to rebuild first. The only stale lines allowed through are the old audio row this upload replaces and rows already on `assets/recut-checklist.md` during a re-cut. Outside a re-cut, any stale line is a STOP: route to `podcast-recut-and-republish`.
2. **Clearance gate.** Run `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP: report the script's stderr verbatim and offer to draft the consent request with `podcast-guest-ops`. On exit 0, print the restrictions and honor them in the episode description.
3. **Confirm the export.** Call `exports_get_export` with the export id. It must be `COMPLETED` and pinned to the revision in `canonical.md`. If it is `PENDING`, keep polling. If it is `FAILED` or pinned to another revision, STOP and say which.
4. **Download it.** The MCP returns an S3 key, never a link. The user downloads the file from the Riverside exports folder and gives you the path. Keep it in `workspace/episodes/SLUG/assets/`, which is gitignored. Run `ffprobe` on it and compare its duration with the canonical edit's runtime (in `assets/package.md`, or the `durationAfterMs` a re-cut reported). If it is more than a few seconds off, STOP: it is probably a different render.
5. **Loudness.** Common practice for podcasts is around -16 LUFS integrated for stereo and -19 LUFS for mono, with true peak near -1 dBTP. Check the host's own guidance too. Measure first, and apply only if the file sits more than about 1 LU off target:

   ```bash
   # pass 1: measure, and read the JSON it prints
   ffmpeg -i IN.mp3 -af loudnorm=I=-16:TP=-1:LRA=11:print_format=json -f null -
   # pass 2: apply, feeding back the four measured values
   ffmpeg -i IN.mp3 -af loudnorm=I=-16:TP=-1:LRA=11:measured_I=MI:measured_TP=MTP:measured_LRA=MLRA:measured_thresh=MT:linear=true -ar 44100 -c:a libmp3lame -b:a BITRATE NORM.mp3
   ```

   Pass 2 re-encodes, so set the bitrate the host recommends, and keep `-ar 44100` because `loudnorm` resamples internally. Run pass 1 on `NORM.mp3` to prove it landed. Normalize once: if the export was rendered with gain normalization, measure before adding more.
6. **Tags, art and chapters.** Build an FFMETADATA file from the chapter list in `podcast-show-notes-and-chapters`, one block per chapter:

   ```
   ;FFMETADATA1
   [CHAPTER]
   TIMEBASE=1/1000
   START=0
   END=185000
   title=Why the free trial had to go
   ```

   Then write ID3v2.3 tags, the art and the chapters in one pass, copying streams so nothing re-encodes:

   ```bash
   ffmpeg -i NORM.mp3 -i ART.jpg -i chapters.ffmeta -map 0:a -map 1:v -map_chapters 2 -c copy -id3v2_version 3 \
     -metadata title="EPISODE TITLE" -metadata artist="SHOW" -metadata album="SHOW" -metadata track=N -metadata date=YYYY \
     -metadata:s:v title="Album cover" -metadata:s:v comment="Cover (front)" FINAL.mp3
   ```

   Verify with `ffprobe -show_format -show_chapters FINAL.mp3` (or `ffmpeg -i FINAL.mp3 -f ffmetadata -` where ffprobe is missing): every tag present, chapter count and start times matching the list. If anything is off, fix it and re-run before uploading.
7. **Upload to the host.** By hand in the host's dashboard, or through the host's API. For the API, read the host's current API documentation first and confirm the endpoint, the auth scheme and the upload flow before any call. Never guess an endpoint or reuse one from memory. Riverside's own hosting is managed in the Riverside app, and no MCP tool reaches it. Before any call or click that publishes or schedules, show the user the show name, episode title, episode and season number, publish now or the date, time and timezone, and draft or live. Get an explicit yes. Once apps fetch the feed the episode is out, and pulling it later does not reach copies already downloaded.
8. **RSS metadata checklist.** Check the episode in the host, or in the fetched feed, against every line:
   - Title, matching the approved option
   - Description: the show notes, with every `[confirm link]` resolved or removed
   - Episode number and season number
   - Episode type: full, trailer or bonus
   - Explicit flag, set on purpose
   - Enclosure: URL, length in bytes, and type (`audio/mpeg` for MP3)
   - Duration, publish date and timezone, episode art
   - GUID present, and unchanged if the episode existed before

   If any `[confirm link]` or "confirm with customer" flag is unresolved, STOP publishing until the user resolves it.
9. **Replacing audio on a live episode.** Before touching anything, fetch the feed (`curl -s FEED_URL`) and record the episode's current GUID. Replace the file on the same episode in the host. Never delete and re-create the episode: the new episode gets a new GUID, and podcast apps show it as a second episode. Fetch the feed again: the GUID must be identical and the enclosure length should have changed. If the GUID changed, STOP and tell the user before anything else happens, so it is fixed in the host before apps pick up the duplicate.
10. **Log it.** Add a `publish-log.md` row: asset `episode audio`, `built_from` copied from `canonical.md`, destination the host's name, reference the host's episode id, state `published` or `scheduled`. For a replacement, log the new row and mark the old one `superseded`. Run `stale-check SLUG` again and report done only on exit 0.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 3 | `exports_get_export` | Status and pinned revision. Returns an S3 key, not a download link |

Everything else in this skill happens outside Riverside: the exports folder download, ffmpeg, the host, and the feed.

## Output
- Writes: the prepared file in `assets/` (tagged, normalized, chapters embedded if asked), and one `publish-log.md` row with the host episode id as reference
- Prints: the loudness before and after, the `ffprobe` tag and chapter check, the RSS checklist with each line marked, and for a replacement the GUID before and after

## Rules & quality bar
- **The GUID never changes.** Replace audio on the existing episode. Never delete and re-create it
- **Read the host's API docs before calling it.** No endpoint from memory
- **Verify by running it.** Loudness is re-measured, tags and chapters are read back with `ffprobe`, and the feed is fetched after a replacement
- **The scripts gate the upload.** Both `stale-check` and `clearance-check` exit 0 before the file goes anywhere
- **Loudness targets are common practice.** Where the host publishes its own spec, the host's spec wins
- **API keys live in environment variables,** never in the workspace

## Related skills
- Called by: `podcast-episode-pipeline` (first release), `podcast-recut-and-republish` (replacement)
- Chapter list: `podcast-show-notes-and-chapters`
- YouTube and social: `distribution-and-scheduling`
