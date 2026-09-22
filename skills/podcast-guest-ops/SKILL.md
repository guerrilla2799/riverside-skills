---
name: podcast-guest-ops
description: >-
  Book a guest, "prep brief", "what should I ask", "guest release", guest share pack. Checks past episodes with search_recording_transcripts_exact and records the release in clearance.md. Editing: podcast-episode-pipeline.
---

# Podcast Guest Ops

Everything about a guest that happens outside the edit: booking, the prep brief, the questions that produce clips, the written release, and the pack the guest gets to share once the episode is live. The release decides whether anything else can ship, so this skill captures it into `clearance.md` with its evidence and lets `clearance-check` rule on it.

## When to use
- A guest is booked or about to be ("book a guest for episode 48")
- "Prep brief", "what have we already covered", "what should I ask"
- The release needs to be requested, chased or recorded
- The episode is live and the guest needs clips and copy to share

Not editing or packaging the episode (`podcast-episode-pipeline`), not scoring clips (`clip-selection`), not writing platform copy from scratch (`social-asset-pack`).

## Inputs
- Guest name, role and company, and the episode topic, from the user
- Episode number and a slug in the form `NNN-topic`, from the user or proposed and confirmed
- The Riverside studio link for the session. The user creates the session in Riverside
- The show's first recording date, so searches reach every past episode
- Background on the guest the user supplies, or public pages you cite by link
- Scripts live at `../riverside-skills/scripts/`, relative to this skill's base directory. Run them from the folder that holds `workspace/`, or set `RIVERSIDE_WORKSPACE`

## Workflow

### Before the recording

1. **Create the folder at booking.** Run `../riverside-skills/scripts/new-recording SLUG`, so `clearance.md` exists before the recording does. If it exits 1 because the folder already exists, use that folder.
2. **Booking.** The recording session is scheduled in Riverside itself. The MCP cannot create, schedule or invite anyone to a session. Draft the invite for the user to send, with:
   - Date, time and timezone, and how long to hold
   - The studio link the user pastes from Riverside
   - What the episode covers, in two sentences
   - A short tech list: headphones, a wired connection or strong Wi-Fi, a quiet room, a browser Riverside supports
   - The release to sign, attached or linked

   You draft it, the user sends it.
3. **Prep brief: what the show already covered.** Search past episodes before writing a single question.
   - `search_recording_transcripts_exact` with the guest's topic phrases and `createdAfterDate` set to the show's first recording date. Left out, the search covers only the last 120 days and silently misses older episodes
   - `search_riverside` with `filters: [{"entity": "TAKE", "field": null, "value": null}]` for looser topic matches. Pass explicit `null` for unused parameters. Results come 10 at a time through `nextSearchAfter`
   - For each hit worth using, `platform_get_transcript` on its `sessionId` for the full sentence and its time. Label those times recording time. If the host will cite a moment on air, re-time it on that episode's canonical edit with `editing_read_aligned_transcript`

   Write `assets/prep-brief.md` with three parts: topics already covered (episode and time), callbacks (a past guest's claim this guest can agree or argue with, quoted verbatim with recording name and timestamp), and gaps the show has not touched. If the searches return nothing, say so. Never describe past coverage you did not find. Background on the guest comes only from what the user supplied or pages you cite by link, with no guessed titles, numbers or employers.
4. **Question design.** Three questions carry the clips:
   - **A specific story.** "Walk me through the week the pipeline number came in at half of plan. What did you do that Monday?" A story has a place, a time and a decision in it
   - **A number.** "What was the number before, and what was it after?" Numbers make titles. Any number the guest gives is flagged "confirm with customer", the pack's shared flag, until they confirm it in writing
   - **A disagreement.** "What does everyone in your role believe that you think is wrong?" Disagreement is the part people share

   Around them: one warm-up, two or three follow-ups drawn from the brief's callbacks, and one close ("what should someone do differently on Monday?"). Leave out abstract prompts such as "what's your philosophy on X". They produce answers nobody clips. Write `assets/questions.md`, each question noting the brief item it came from.
5. **Capture the release.** Draft the release request in plain language: what will be published (the full episode, clips, quotes, the guest's name, title and company), where (the feed, YouTube, social platforms), that the show edits for length and clarity, and any review the guest asks for. When the signed release or approving email comes back, save it into the episode folder (`release.pdf`, or the email saved as a file) and confirm the file exists with `ls`. Then fill `clearance.md`:
   - `status: cleared` and `scope: public`
   - `approved_by`: name, role, company of the person who signed
   - `approved_on`: the date on the approval, YYYY-MM-DD
   - `evidence`: the saved file's path
   - `restrictions`: anything the guest ruled out, and any review they asked for ("guest reviews final cut before publish")

   A verbal yes on the call does not count. Never fill `approved_by` or `evidence` without the document in hand.
6. **Clearance gate before recording day.** Run `../riverside-skills/scripts/clearance-check SLUG public`. Exit 0: tell the user and list the restrictions. Non-zero: report the stderr verbatim and offer to send the release request again. Recording without a release is the user's call, but nothing public ships until this check exits 0, and every downstream skill runs the same check.

### After the episode is live

7. **Gates for the guest pack.** Run `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP: report the stderr verbatim and offer to draft the consent request. Run `../riverside-skills/scripts/stale-check SLUG`. If it exits non-zero, STOP: something was built from a cut other than the canonical one, so route to `podcast-recut-and-republish`. For every post whose link goes in the pack, `social_get_upload_status` must return `COMPLETED`. For the host episode link, the user confirms it is live. A link to anything not yet live stays out.
8. **Build the pack** in `assets/guest-pack/`:
   - Two or three clips the user already approved from `clip-selection`. Clip files come from `exports_create_export` (behind the repo's `ask` rule, and each call renders again) and are downloaded from the Riverside exports folder, because the MCP gives no link. Confirm each render with `exports_get_export`
   - Suggested copy per platform from `social-asset-pack`, written so the guest can post it in their own voice
   - The episode links, and one or two verbatim quote lines with canonical timestamps

   Honor every restriction `clearance-check` printed. The guest posts from their own accounts. This skill publishes nothing.
9. **Hand it over.** Draft the email to the guest with the pack. The user sends it. Log the pack in `publish-log.md`: asset `guest share pack`, `built_from` copied from `canonical.md`, destination `guest`, reference the folder path, state `approved` once the user signs off. If the cut changes later, `stale-check` flags this row and `podcast-recut-and-republish` sends the guest a corrected pack.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 3 | `search_recording_transcripts_exact` | Defaults to the last 120 days. Always pass `createdAfterDate` |
| 3 | `search_riverside` | Fuzzy. `TAKE` filter for spoken moments. Explicit `null` for unused parameters |
| 3 | `platform_get_transcript` | Sentence times from the start of the raw recording. Fine for prep, never for anything published |
| 3 | `editing_read_aligned_transcript` | Re-times a moment on a past episode's canonical edit before the host cites it |
| 7 | `social_get_upload_status` | A link goes in the pack only when its post is `COMPLETED` |
| 8 | `exports_create_export` · `exports_get_export` | Clip files for the guest. S3 key, not a link |

Session scheduling and invites happen in the Riverside app. No MCP tool covers them.

## Output
- Writes to `workspace/episodes/SLUG/`: the filled `clearance.md`, the saved release file, `assets/prep-brief.md`, `assets/questions.md`, `assets/guest-pack/`, and a `publish-log.md` row for the pack
- Prints: the invite and release-request drafts, the brief, the questions, the `clearance-check` result, and the pack contents with every link's live status

## Rules & quality bar
- **The script decides clearance.** `clearance.md` is filled only from a written document saved in the folder, and `clearance-check` rules on it
- **Search before you ask.** No question goes in without the past-episode search, and `createdAfterDate` is always set
- **No invented background.** Every fact about the guest comes from the user or a cited page. Every callback quote carries recording name and timestamp
- **Numbers are flagged** "confirm with customer" until the guest confirms them in writing
- **The guest pack waits** for `clearance-check` exit 0, `stale-check` exit 0, and live links
- **Drafts only.** The user sends every email and invite

## Related skills
- Next: `podcast-episode-pipeline` once the recording exists
- Clips and copy for the pack: `clip-selection`, `social-asset-pack`
- A corrected pack after a re-cut: `podcast-recut-and-republish`
- Front door: `riverside-skills`
