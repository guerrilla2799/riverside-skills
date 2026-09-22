---
name: webinar-production
description: >-
  Webinar run-of-show, webinar promo and reminder copy, webinar to on-demand, post-webinar follow-up. Plans the live event, then trims the recording with editing_remove_pauses and cuts. Registration runs in your webinar tool. Clips: clip-selection.
---

# Webinar Production

Plans a webinar, then turns its recording into an on-demand version, chapters, clips and follow-up
emails. The Riverside MCP has no webinar, registration or email tools. Registration, reminders and
the live event itself run in Riverside's webinar product or your webinar platform. This skill
writes the plan and the copy, then does the post-live editing through the MCP.

## When to use
- A webinar is booked and needs a run-of-show, promo copy and reminders
- A webinar just ended and needs an on-demand version, chapters and clips
- A panel, fireside chat or launch stream with the same shape

## Inputs
- **Before:** title, date, start time and timezone, length, speakers (name, role, internal or
  guest), the one thing an attendee should leave with, the call to action, and the platform
- **After:** the recording in Riverside. If the webinar ran on another platform, upload the
  recording in the Riverside app first. Files sent through `media_create_media_upload` land in
  Your Media as editor assets and are not treated as a recording
- **Clearance:** written approval from every guest speaker, and from any attendee who spoke on
  camera or mic, recorded in `clearance.md`
- **Workspace:** `./workspace`, or `RIVERSIDE_WORKSPACE` if set

## Workflow

### Before the live

1. **Scaffold and start clearance early.** Run `../riverside-skills/scripts/new-recording SLUG`
   (exit 1: the folder exists, use it). Send each guest speaker the written consent ask now, not
   after the event, covering the recording, the on-demand version, clips and social posts. Adapt
   the consent ask in `customer-interview-engine`. Replies go into the folder and
   `clearance.md`.
2. **Run-of-show** to `assets/run-of-show.md`. A table: clock time with timezone, minute, segment,
   who is on, what is on screen, producer cue. Default shape for a 45-minute session:
   - T-30: speakers join backstage. Audio, camera, slides and backup phone numbers checked
   - T-5: doors open on a holding slide
   - 0:00 host welcome and housekeeping, including "this is recorded, you'll get the on-demand link"
   - 0:03 speaker intros. 0:05 to 0:30 content in blocks, one poll near 0:15
   - 0:30 to 0:42 Q&A. The moderator reads each question aloud, so attendee audio stays out
   - 0:42 call to action and close. 0:45 end. Speakers stay backstage five minutes

   Roles: host (owns the clock), speakers, producer (slides, polls, backstage, the platform), Q&A
   moderator (triages and reads questions), and a co-host logged in who can take over.
   Backup plan for a speaker dropping: the producer calls the speaker's phone at once, and the
   host pulls the next block or Q&A forward. The host holds the speaker's slides and three talking
   points per block. If the speaker is not back in five minutes, the host or another speaker runs
   the block from the notes. If the host drops, the co-host takes over. If the platform fails, a
   prepared message with a fallback link goes to registrants.
   An attendee who comes on camera or mic is told it is recorded. A verbal OK does not clear a
   clip: their written approval follows by email, or their segment stays out of clips.
3. **Promo and reminder copy** to `assets/promo-copy.md`: the invite email, two LinkedIn posts
   (the speaker announcement, what attendees will learn), a speaker share kit, reminders at 7 days,
   1 day and 1 hour, and a "we're live" note. REGISTRATION-LINK stays a placeholder. Every piece
   goes to `content-quality-gates` before it is sent. Sending happens in the webinar platform or
   email tool. `social_upload_create` only posts a Riverside clip or edit, so a text LinkedIn post
   goes out natively on LinkedIn. A video teaser can go through `distribution-and-scheduling`.

### After the live

4. **Find the recording.** `platform_list_recordings` (newest first) or `platform_get_project`.
   If a rehearsal and the live session are both there, or the recording has several takes, list
   them and ask. Never pick one.
5. **Declare the canonical cut before anything else.** Create the working edit with
   `editing_create_edit_from_recording`, `type: "EPISODE"`, titled "TITLE on-demand". On
   `FAILED_PRECONDITION` do not retry: use an edit from `platform_list_edits` (skip
   `deleted: true`) or ask the user to create one in Riverside. Fill `canonical.md`. To keep a
   full-length version with every question, run `editing_clone_edit` first and declare the clone.
6. **Trim the dead time. The user approves the ranges.** Read the edit with
   `editing_read_aligned_transcript` and find the pre-show (holding music, mic checks, "we'll give
   people a minute"), dead stretches in Q&A ("let me look at the chat"), and anything after the
   close. Show each range with its first and last words and its length. On a yes, resolve with
   `editing_resolve_transcript_selection`, `intent: "remove"`, and apply with
   `editing_cut_time_ranges` only when `readyToApply` is true, passing `expectedRevision`. Then
   run `editing_remove_pauses` with `thresholdMs: 1500`. A talk needs some air, and a threshold
   under 1000 sounds clipped. `editing_restore_audio_cleanup` with `cleanup: "pauses"` undoes it.
7. **Update the canonical cut on purpose.** Read the new revision with `editing_get_revision`, write
   it into `canonical.md` with a History row ("pre-show and dead Q&A cut, pauses over 1500 ms
   removed"), then run `../riverside-skills/scripts/stale-check SLUG`. Exit 2: STOP, there is no
   canonical cut. Exit 1: assets built from the untrimmed cut are logged. List them, rebuild them,
   and mark the old rows superseded.
8. **Speaker clearance gate. STOP before anything leaves.** List every speaker label in the
   canonical transcript. Run `../riverside-skills/scripts/clearance-check SLUG public`. Non-zero
   exit: STOP the external release of the on-demand version and every clip. Show the script's
   stderr verbatim and offer to draft the consent request for each speaker without one. Exit 0:
   show the restrictions and honor them. The script reads one file, not each voice, so then check
   that every speaker outside your own team is named in `approved_by`. A speaker who is not named
   stays out of clips, and their audio is cut from the on-demand version until they approve.
9. **Chapters.** Hand `edit_id@revision` to `podcast-show-notes-and-chapters`. Chapters come from
   the canonical edit's playable timeline, never raw recording time, because the trims shifted
   every timestamp after them. That skill may write chapter markers into the edit, which bumps
   the revision in `canonical.md` with a History row. Every step after this reads
   `edit_id@revision` fresh from `canonical.md`.
10. **On-demand version.** Export with `exports_create_export`. This repo's `ask` rule prompts
    before it runs, and calling it twice renders twice. Poll `exports_get_export` until
    `COMPLETED`. `FAILED` is final. The MCP returns no download link: download the file from the
    Riverside exports folder and upload it to the webinar platform's on-demand page by hand. A
    YouTube upload goes through `distribution-and-scheduling`.
11. **Clips.** Hand `edit_id@revision` and the list of cleared speakers to `clip-selection`, then
    `social-asset-pack`. `distribution-and-scheduling` publishes. This skill never calls
    `social_upload_create`.
12. **Follow-up email copy** to `assets/follow-up-copy.md`: attendees (thanks, the on-demand link,
    one takeaway or clip, the call to action), no-shows (the on-demand link and the one thing they
    missed), and people who asked a question (the answer, or an offer to talk). ON-DEMAND-LINK
    stays a placeholder until the page exists. It goes through `content-quality-gates` first, then
    out from the webinar platform, where the attendee lists live.
13. **Log and finish.** A row per asset in `publish-log.md`: the on-demand export, chapters, each
    clip, each follow-up email. `built_from` is `edit_id@revision` from `canonical.md`. Done means
    `stale-check SLUG` exits 0.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 4 Find | `platform_list_recordings` · `platform_get_project` | Ask when a rehearsal or second take is present |
| 5 Canonical | `editing_create_edit_from_recording` · `platform_list_edits` | `FAILED_PRECONDITION` means use an existing edit |
| 5 Branch | `editing_clone_edit` | Only to keep a full-length version beside the on-demand cut |
| 6 Trim | `editing_read_aligned_transcript` → `editing_resolve_transcript_selection` → `editing_cut_time_ranges` | Apply only when `readyToApply` is true |
| 6 Air | `editing_remove_pauses` · `editing_restore_audio_cleanup` | 1500 ms default. Needs a loadable transcript |
| 7 Revision | `editing_get_revision` | The value written into `canonical.md` |
| 10 Export | `exports_create_export` · `exports_get_export` | Behind the approval gate. No download link |
| None | Registration, reminders, the live room, attendee lists | Not on the MCP. They live in your webinar platform |

## Output
- **Before:** `assets/run-of-show.md`, `assets/promo-copy.md`, consent asks sent to each guest
- **After:** the on-demand edit in Riverside with `canonical.md` at its final revision, chapters
  via `podcast-show-notes-and-chapters`, an export in the Riverside exports folder,
  `assets/follow-up-copy.md`, and a `publish-log.md` row per asset
- **Prints:** the trim ranges for approval, the new revision, the clearance result with every
  speaker named or missing, the export status, and the `stale-check` result

## Rules & quality bar
- **Say the seam out loud.** Registration and the live event are never promised through the MCP
- **Canonical before any cut,** and every change to it gets a History row in step 7
- **The user approves trim ranges** before `editing_cut_time_ranges` runs
- **Clear each speaker by name.** Step 8 checks each speaker against `approved_by`
- **Playable timestamps only** for chapters and clips
- **Copy is judged before it is sent,** by `content-quality-gates`
- **Done means `stale-check` exits 0**

## Related skills
- Hands off to: `podcast-show-notes-and-chapters`, `clip-selection`, `social-asset-pack`,
  `distribution-and-scheduling`, `content-quality-gates`
- Pairs with: `customer-interview-engine` for the consent wording, when a customer is a speaker
- Setup and shared scripts: `riverside-skills`. Tool catalog and seams: `docs/riverside-mcp.md`
