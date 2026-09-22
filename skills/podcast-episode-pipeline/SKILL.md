---
name: podcast-episode-pipeline
description: >-
  Run podcast-episode-pipeline on episode N, "produce this episode", "show me the full package". Declares the canonical cut, tightens it in Riverside, builds the review package, exports only on approval. Re-cuts: podcast-recut-and-republish.
---

# Podcast Episode Pipeline

The spine for one episode, from a finished recording to a reviewed package and, only on approval, to the host and YouTube. It declares which cut is canonical before any other work, tightens that cut inside Riverside's editor, hands each asset to the skill that owns it, and stops at the package review by default. Every asset carries the `edit_id@revision` it came from, so a later re-cut can find everything it made stale.

## When to use
- A new episode is recorded and needs chapters, titles, show notes, clips, social copy and a thumbnail
- "Don't publish anything. Show me the full package first." That is this skill's default stop point
- An approved package is ready to export, host and publish

Not for an episode whose assets are already out and now point at the wrong cut. That is `podcast-recut-and-republish`.

## Inputs
- Episode number and recording date, from the user. The recording itself lives in Riverside
- A slug in the form `NNN-topic`: lowercase, digits and hyphens (`047-pricing-objections`). Propose one and confirm it
- For a guest episode, the guest's written release, captured into `clearance.md` by `podcast-guest-ops`. For a solo episode the user fills `approved_by: self (sole speaker)` plus a one-line `evidence` note, because the script fails on an empty `evidence` field
- Optional: a closing card (image or video on disk), the publish targets and the times the user wants
- Scripts live at `../riverside-skills/scripts/`, relative to this skill's base directory. Run them from the folder that holds `workspace/`, or set `RIVERSIDE_WORKSPACE`, because every script resolves the workspace from the current directory

## Workflow

### Phase 1: declare the canonical cut

1. **Find the recording.** Use `platform_list_recordings` (newest first) or `search_riverside` with the episode title, then `platform_get_project` for its recordings and edits in one call. Show title, date and duration for each match. If more than one recording fits the date, or none does, list what you found and ask. Never pick one yourself.
2. **Scaffold the folder.** Run `../riverside-skills/scripts/new-recording SLUG`. If it exits 1 because the folder exists (`podcast-guest-ops` creates it at booking), use that folder. If its `canonical.md` already has an `edit_id`, STOP: this episode already has a declared cut. Ask whether this is a re-cut, and if it is, route to `podcast-recut-and-republish`.
3. **Declare the canonical cut before any other work.** List the recording's edits from `platform_get_project` and drop any with `deleted: true`. Show title, type and last update for each and ask the user which one is canonical, even when there is only one. If there are none, create one with `editing_create_edit_from_recording`. On `FAILED_PRECONDITION` do not retry: fall back to `platform_list_edits`, or ask the user to open an edit in Riverside. Read the head with `editing_get_revision`, then fill `recording_id`, `edit_id`, `revision`, `declared`, `declared_by` and `reason` in `canonical.md` and add the first History row. Nothing else in this skill runs until that file is filled.
4. **Clearance gate.** Run `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP drafting: report the script's stderr verbatim, tell the user nothing public will be drafted, and offer to draft the release request with `podcast-guest-ops`. Tightening (step 5) is internal editing and may continue if the user asks. Steps 6 to 14 do not run until the check exits 0. On exit 0, print the `Restrictions:` line if there is one and pass it into every hand-off. Never edit `clearance.md` to make the check pass.

### Phase 2: tighten the cut

5. **Tighten the canonical edit in place, while nothing is built from it.** If `publish-log.md` already has live rows, STOP: changing the cut now makes them stale, so route to `podcast-recut-and-republish`. Otherwise make one write at a time, each passing `expectedRevision` set to the revision in `canonical.md`:
   - **Dead air:** `editing_remove_pauses` with a `thresholdMs` the user agrees to. 1500 is a sensible start
   - **Fillers:** `editing_remove_fillers` with `method` `Smart`, unless the user wants `Cut` or `Mute`
   - **Trailer trim** (pre-show chatter, countdown, post-show talk): this is a cut. Resolve the range with `editing_resolve_transcript_selection` (intent `remove`) and pass its payload to `editing_cut_time_ranges` unchanged, only when `readyToApply` is true
   - **Closing card:** `media_create_media_upload`, upload the bytes with `curl -T FILE -H "Content-Type: MIME" "UPLOAD_URL"`, `media_finalize_media_upload`, poll `media_get_media` until it is processed, then `editing_insert_media_as_scene` with the media id at the end of the timeline (`playableEndMs` from `editing_read_aligned_transcript`). A scene inserted anywhere earlier pushes every later timestamp right

   After each write, read the new head with `editing_get_revision`, set `revision` in `canonical.md`, and add a History row naming the change ("pauses over 1500 ms removed"). If a write is rejected for a stale revision, STOP: someone else changed the cut. Show `editing_compare_revisions` from your revision to the head and ask what to keep. `editing_restore_audio_cleanup` undoes a pause or filler pass.

   When the tighten is done, summarize it with `editing_compare_revisions` from the declared revision to the head: runtime before and after, and cuts grouped by `feature`. Flag it if more than about 10% of the runtime went. Riverside's own editing guide calls 5–10% the safer range for a conversation.

### Phase 3: build the package (drafts only)

6. **Chapters, titles, show notes.** Hand off to `podcast-show-notes-and-chapters`. Timestamps come from the canonical edit's playable timeline (`editing_read_aligned_transcript` on the edit), never from raw recording times, because cuts shift time.
7. **Clips.** Hand off to `clip-selection` on the canonical edit. It scores and cuts. It publishes nothing.
8. **Social copy.** Hand off to `social-asset-pack` for the clips the user picks. For the full episode, first log a row (asset `full episode`, reference the canonical `edit_id`, state `draft`) so `social-asset-pack` can write its per-platform copy the same way and `distribution-and-scheduling` can later post it. The full episode's YouTube title and description come from step 6: the chosen title, the show notes and the chapter list.
9. **Thumbnail brief.** There is no image-generation tool, so the thumbnail is a brief: on-image text of four words or fewer, the frame to grab as a canonical playable timestamp, and the composition (whose face, what expression, where the text sits). If a video export of the canonical revision is already on disk, grab the base frame now with `ffmpeg -ss HH:MM:SS -i EXPORT.mp4 -frames:v 1 -q:v 2 assets/thumbnail-base.jpg`. Otherwise the grab happens right after step 11. Export time equals playable time, because the export renders the edit. The MCP has no thumbnail field for YouTube, so the user sets it in YouTube Studio once the video is up.
10. **Assemble the full package and stop.** Write `assets/package.md` with: canonical `edit_id@revision` and runtime, the tighten summary, chapters, the three title options, show notes, the clip shortlist with the reason for each, social copy per platform, the thumbnail brief, the proposed publish plan (host, YouTube, and each post with platform, account name, date, time and timezone), and every open flag (`[confirm link]`, and "confirm with customer", which this pack uses for any number a guest said aloud). Log the thumbnail brief and anything else not yet logged in `publish-log.md`, state `draft`, `built_from` copied from `canonical.md`. Then send the package through `content-quality-gates`, which fails any asset without a row. Show the user the package path, the verdict, and a one-screen summary.

    **Stop here.** Nothing below runs until the user approves the package in this conversation. Approval of one piece approves that piece only. On approval, set each approved piece's row to `approved`, provided `content-quality-gates` passed it. A row with an open "confirm with customer" flag stays `draft`, and `distribution-and-scheduling` will not post it.

### Phase 4: ship (only after explicit approval)

11. **Export.** Run `../riverside-skills/scripts/stale-check SLUG`. If it exits non-zero, STOP: an asset was built from a different cut. List the rows and fix them before rendering. Check that `editing_get_revision` still equals `canonical.md`. If it changed, STOP and ask. Confirm the render settings with the user (`MP3` or `WAV` for the feed, `1080p` for YouTube) and say that every call renders again. Call `exports_create_export` with `rstlRevisionId` set to the canonical revision. The repo's `ask` rule also prompts for this tool, and this confirmation comes first either way. Poll `exports_get_export` until `COMPLETED` and check the revision it pinned. There is no download link: the file lands in the Riverside exports folder.
12. **Host.** Hand off to `podcast-host-and-rss`: download, loudness, tags, upload, RSS metadata, and the host episode id logged.
13. **YouTube and socials.** Call `editing_get_export_publish_data` on the canonical edit. If any `youtubeCode` is non-null, STOP: name the flagged media and let the user choose between replacing it (a change to the cut, so update `canonical.md` and re-run stale-check) and publishing with a likely claim. Re-check `editing_get_revision` against `canonical.md`, because `social_upload_create` publishes the edit as it stands and takes no revision. Then hand off to `distribution-and-scheduling`, which owns the final summary, the explicit yes, and the `social_get_upload_status` check (once shortly after, then when the user asks). This skill never calls `social_upload_create`.
14. **Close the log.** Every asset has a row in `publish-log.md`: `built_from`, destination, reference (export id, host episode id, uploadId or file path) and state. Run `stale-check SLUG` once more. Report the episode done only when it exits 0.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 1 | `platform_list_recordings` · `search_riverside` · `platform_get_project` | `get_project` returns recordings and edits in one call. Never pass `includePreviewUrl` to `platform_get_recording`: it returns a studio-wide, non-revocable credential |
| 3 | `editing_create_edit_from_recording` · `platform_list_edits` · `editing_get_revision` | `FAILED_PRECONDITION` means fall back, never retry. Drop soft-deleted edits |
| 5 | `editing_remove_pauses` · `editing_remove_fillers` · `editing_restore_audio_cleanup` | Pass `expectedRevision` on every write |
| 5 | `editing_read_aligned_transcript` · `editing_resolve_transcript_selection` · `editing_cut_time_ranges` | Playable milliseconds. Run the payload only when `readyToApply` is true |
| 5 | `media_create_media_upload` · `media_finalize_media_upload` · `media_get_media` · `editing_insert_media_as_scene` | Up to 500 MB: MP4, WEBM, MPEG, WAV, MP3, OGG, JPG, PNG. A scene lengthens the timeline |
| 5 | `editing_compare_revisions` | The tighten summary, and what changed when a write is rejected |
| 11 | `exports_create_export` · `exports_get_export` | Behind the `ask` rule. Calling twice renders twice. Returns an S3 key, not a link |
| 13 | `editing_get_export_publish_data` | A non-null `youtubeCode` is a likely Content ID match |

## Output
- Writes: `canonical.md` (filled, with a History row per intentional change), `assets/package.md`, the drafts the sibling skills write into `assets/`, and one `publish-log.md` row per asset
- Prints at the stop point: the package path, the canonical `edit_id@revision`, runtime before and after, and every open flag
- Prints after shipping: export id, host episode id, each uploadId with its `COMPLETED` status, and the final `stale-check` line

## Rules & quality bar
- **Canonical first.** Step 3 fills `canonical.md` before any edit, cut or draft
- **The scripts decide.** Clearance and staleness come from `clearance-check` and `stale-check` exit codes, never from judgment
- **The package is the default stop.** Export, hosting and publishing need explicit approval in this conversation
- **Every write passes `expectedRevision`,** and every intentional change to the cut gets a History row
- **No fabrication.** Every quote carries recording name and canonical timestamp. A number the guest said aloud is flagged "confirm with customer" (the pack's one flag for guests and customers alike) until confirmed in writing
- **This skill publishes nothing itself.** Posting belongs to `distribution-and-scheduling`, the feed to `podcast-host-and-rss`
- **Name accounts the way the user knows them,** by channel or account name, never by internal id

## Related skills
- Before: `podcast-guest-ops` (booking, questions, release)
- Hands off to: `podcast-show-notes-and-chapters`, `clip-selection`, `social-asset-pack`, `content-quality-gates`, `podcast-host-and-rss`, `distribution-and-scheduling`
- When the cut changes after assets exist: `podcast-recut-and-republish`
- Front door: `riverside-skills`
