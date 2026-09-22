---
name: podcast-recut-and-republish
description: >-
  "Wrong cut went live", "re-cut the episode", "republish the fixed version". Cancels queued posts from the old cut (social_cancel_upload), re-declares canonical, re-pushes every stale asset. First release: podcast-episode-pipeline.
---

# Podcast Recut and Republish

The 8:30pm path. The wrong cut is live, and every chapter, clip, caption and host file downstream points at it. This skill stops what is still queued, re-declares the canonical cut, lets `stale-check` name every asset that now points at the wrong version, and re-derives and re-pushes each one. Done means `stale-check` exits 0. A new export on its own is not done.

## When to use
- The published episode is the wrong cut: a rough cut, a pre-fix version, or one with a segment the guest asked to remove
- The canonical cut changed after assets were built from it
- A clip, post or host file was built from a cut other than the one in `canonical.md`

Not for an episode going out for the first time. That is `podcast-episode-pipeline`.

## Inputs
- The episode folder, with `canonical.md` and `publish-log.md`. If there is no folder, the assets were never logged: create it with `new-recording SLUG`, then rebuild the list with the user from step 1 before anything else
- `studioId` and `productionId` for the social tools, from `platform_get_recording` and `platform_list_productions`
- Which cut is correct, from the user. Never inferred from recency or title
- The date the first asset went out: the earliest `publish-log.md` row, or ask
- Scripts live at `../riverside-skills/scripts/`, relative to this skill's base directory. Run them from the folder that holds `workspace/`, or set `RIVERSIDE_WORKSPACE`

## Workflow

### Phase 1: stop the bleeding

1. **Find every post.** Call `social_list_uploads` with an explicit `from` (a day before the first asset went out) and `to` (at least 90 days ahead, to catch the queue). With no dates it returns the next 30 days only and misses everything already published. Page with `offset` while `hasMore` is true; `total` counts the window before any filter. Match rows to the uploadIds in the `reference` column of `publish-log.md`. For posts in the window that are not in the log (made in the Riverside app, for example), list platform, account, first line of text, status and time, and ask which belong to this episode. Never guess.
2. **Read each one.** `social_get_upload` per uploadId. Read `editable`, `mediaEditable` and `cancellable` separately. They do not move together.
3. **Stop what is queued.** For each post from the wrong cut that is `cancellable`, offer two options: cancel it with `social_cancel_upload`, or hold it by moving `scheduledAt` later with `social_update_upload`. A hold only buys time, because the post still carries the old media. When you cannot yet tell whether a clip's moment changed, hold rather than cancel, and let step 9 decide. Before either call, show the platform, the account or channel name, the exact post text, the visibility, and the scheduled date, time and timezone, and get an explicit yes. The repo's `ask` rule also prompts for both tools; do not rely on it alone. Cancelling is irreversible, and afterwards `social_get_upload` returns an authorization error for that id, so the cancel response is the confirmation. Write it down.
4. **List what is already out.** A post with all three flags false has published or is publishing. Nothing on the MCP can edit, unpublish or delete it. Tell the user which posts, on which accounts, and that removal happens on the platform itself. A `FAILED` post with reasonCode `PLATFORM_CONNECTION_EXPIRED` and a `scheduledAt` can still go out when that account is reconnected, so warn the user before they reconnect it. Size the exposure with `social_get_upload_analytics` and quote its `asOf`. The feed is outside the MCP: if the wrong audio is on the host, or scheduled there, the user pauses or replaces it through `podcast-host-and-rss`.

### Phase 2: re-declare the canonical cut

5. **Identify the correct cut.** List the edits from `platform_get_project` (drop `deleted: true`) with title, type, last update and the head from `editing_get_revision`. The user names the correct `edit_id` and revision. If `canonical.md` already names it (the wrong post came from some other cut), leave the file alone and go to step 7.
6. **Update `canonical.md`.** Set `edit_id` and `revision`, and add a History row with the date and why it changed ("rev 9 went live; rev 12 removes the segment the guest asked to cut"). That row is the audit trail for the whole recut.
7. **Run `../riverside-skills/scripts/stale-check SLUG`.** Exit 2: STOP and report the missing file or empty field. Exit 0 while a wrong post is known to be live: that post was never logged. Log it with the `built_from` it really came from (write `unknown` if nobody can tell, and the check will flag it, which is correct), then run the check again. Exit 1: every line on stdout is a row to propagate.
8. **Clearance gate before anything is re-published.** Run `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP re-publishing: report the stderr verbatim and offer to draft the consent request with `podcast-guest-ops`. The takedowns from Phase 1 still stand. On exit 0, print the restrictions and honor them in every re-derived asset.

### Phase 3: propagate

9. **See what moved.** If the correct cut is the same edit at another revision, `editing_compare_revisions` from the old revision to the new one lists every created, modified and deleted cut, clip and scene with its playback position, plus `durationBeforeMs` and `durationAfterMs`. If it is a different edit, that tool cannot diff across edits: read `editing_read_aligned_transcript` on both and match moments by their words.
10. **Build the propagation checklist** in `assets/recut-checklist.md`, one line per stale row: asset, what to re-derive, where to re-push, owner, done.

    | Stale asset | Re-derive | Re-push |
    |---|---|---|
    | Host audio | `exports_create_export` with `rstlRevisionId` set to the new revision, then local prep | `podcast-host-and-rss` replaces the audio on the same host episode, so the GUID stays and apps do not show a duplicate |
    | YouTube full episode, published | A new post from the new cut, after `editing_get_export_publish_data` shows no `youtubeCode` | Cannot be replaced through the MCP. The user removes or privates the old video on YouTube, and `distribution-and-scheduling` creates the new post |
    | Chapters, show notes, titles | `podcast-show-notes-and-chapters` re-reads `editing_read_aligned_transcript` on the new `edit_id@revision` and re-times everything | Host episode description, the new YouTube post's description. A published description is edited on the platform by hand |
    | Clip whose moment changed | `clip-selection` re-cuts it from the new edit | Scheduled: cancel and re-create, because `social_update_upload` cannot swap media. Published: the user removes it on the platform and `distribution-and-scheduling` posts the new one |
    | Clip whose moment did not change | Nothing, if step 9 shows no created, modified or deleted entity inside its range and the user agrees | Leave it live. Log a fresh row with the same reference and the new `built_from`, note "verified unchanged", and mark the old row `superseded` |
    | Scheduled post whose text is wrong (a timestamp or title in the caption) | `social-asset-pack` corrects the text | `social_update_upload` with only the changed fields, after the summary and an explicit yes. LinkedIn comments and X replies are replaced wholesale, so send the full list back |
    | Thumbnail | A new frame grab if the frame moved | YouTube Studio, by hand. The MCP has no thumbnail field for YouTube |
    | Guest share pack | `podcast-guest-ops` rebuilds it | Tell the guest what changed and send the new files |

11. **Work the checklist.** Every re-created post goes through `distribution-and-scheduling`: final summary, explicit yes, and a `social_get_upload_status` check. Only `COMPLETED` counts as live. Before any immediate publish, say it cannot be undone through the MCP.
12. **Update the log.** Log each new asset with the new `built_from`. Mark an old row `superseded` only when its replacement is logged and the old one is confirmed gone (the cancel response, or the user confirming removal on the platform). An old post removed with nothing replacing it is `retired`. Never delete a row.
13. **Re-run `stale-check SLUG` until it exits 0.** Each line it still prints goes back to step 10. Only then report the recut done.

### Phase 4: tell people

14. **Draft the audience note** when a wrong version was public. The user posts it; send the draft through `content-quality-gates` first.
    - If the wrong version exposed something the guest asked to cut, tell the guest privately before anyone else. That is a clearance incident first
    - Three lines at most: what was wrong, that it is fixed, where the right version is. "The first upload of episode 47 was an early cut. The corrected episode is up now, same place in your podcast app."
    - Skip the story of how it happened and skip the long apology
    - Say that podcast apps which already downloaded the old file can keep playing it until the listener deletes and re-downloads it, when the difference matters (a wrong number, a removed segment)
    - Post it where the wrong version was seen: a pinned comment on the new video (by hand on the platform), the next newsletter or episode intro

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 1 | `social_list_uploads` | Needs `studioId` and `productionId`. Always pass explicit `from` and `to` |
| 2 | `social_get_upload` | Branch on the three flags, never on status |
| 3 | `social_cancel_upload` · `social_update_upload` | Behind the `ask` rule. Summary and explicit yes first. Cancel is irreversible |
| 4 | `social_get_upload_analytics` | Collected on a schedule. Quote `asOf` |
| 5 | `platform_get_project` · `platform_list_edits` · `editing_get_revision` | Drop soft-deleted edits |
| 9 | `editing_compare_revisions` · `editing_read_aligned_transcript` | Compare works within one edit only |
| 10 | `exports_create_export` · `exports_get_export` · `editing_get_export_publish_data` | Pin `rstlRevisionId`. No download link, so the file comes from the exports folder |
| 11 | `social_get_upload_status` | Live means `COMPLETED` |

## Output
- Writes: a History row in `canonical.md`, `assets/recut-checklist.md`, new and `superseded` or `retired` rows in `publish-log.md`, and the audience note draft in `assets/`
- Prints: what was cancelled or held, what is still live and must come down on the platform, the checklist, and the final `stale-check` line

## Rules & quality bar
- **Stop the bleeding first.** Queued posts from the wrong cut are handled before any re-derivation
- **`stale-check` decides what is stale,** and its exit 0 decides when the recut is done
- **Nothing published is edited through the MCP.** Published posts are removed by the user on the platform and replaced with new posts
- **Media on a scheduled post changes by cancel and re-create,** text by `social_update_upload`
- **The host keeps the GUID.** Replace the audio on the existing episode, never delete and re-create it
- **Never delete a log row.** Superseded and retired rows are the history of what went out

## Related skills
- Owners of the re-derived assets: `podcast-show-notes-and-chapters`, `clip-selection`, `social-asset-pack`, `podcast-host-and-rss`
- All re-posting: `distribution-and-scheduling`
- The guest side: `podcast-guest-ops`
- First release: `podcast-episode-pipeline`
