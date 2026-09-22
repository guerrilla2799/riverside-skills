---
name: sales-enablement-clips
description: >-
  Objection handling library, sales call clips, rep training, demo snippets, "how do our best reps answer pricing". Finds objections by exact transcript search, cuts each exchange with editing_clone_edit. Messaging research: research-call-mining.
---

# Sales Enablement Clips

Builds a library of objections from recorded sales calls: what the customer said, the best answer
a rep on the team has given on a recorded call, and a clip of that exchange. Demo snippets and other
rep-facing cuts follow the same path. Internal rep training is the default and needs no customer
clearance. A clip that goes to a prospect is external and passes the clearance script first.

## When to use
- Onboarding reps, or a rep asks how others handle an objection
- An objection keeps stalling deals and nobody agrees on the answer
- A rep runs a strong demo segment worth showing the rest of the team
- A clip of a customer on a call is wanted for a prospect (external, gated in step 10)

## Inputs
- **Sales calls in Riverside.** Calls stored in Gong, Zoom, Chorus or a dialer are not visible to
  the MCP. Upload them in the Riverside app to clip them. Transcripts exported into
  `workspace/transcripts/` can supply objection wording for the library, but a clip needs the
  recording in Riverside
- **From the user:**
  - The objection list. Start from "too expensive", "we already use", "not a priority" and
    "need to talk to", then add the variants their buyers use
  - Everyone on the team side who speaks on calls, to tell reps from customers
  - The date range. Exact search covers the last 120 days unless told otherwise
  - Whether any clip will reach a prospect
- **Library:** `workspace/enablement/objection-library.md`, created on the first run
- **Workspace:** `./workspace`, or `RIVERSIDE_WORKSPACE` if set

## Workflow

1. **Search for objection moments.** `search_recording_transcripts_exact` with the objection
   phrases as `searchTerms` (up to 20, OR-combined), `size` up to 50 sessions, and
   `createdAfterDate` for anything older than 120 days. Then `search_riverside` with a TAKE
   filter, `[{"entity": "TAKE", "field": null, "value": null}]`, for looser wording ("pricing",
   "timing", "budget"). Both return short fragments and no timestamps. They locate calls. They are
   never quoted.
2. **Confirm the hits. STOP on anything unplaced.** For each session, pull
   `platform_get_transcript`. Keep a hit only when a customer-side speaker raises the objection.
   A rep saying "some teams find it too expensive" is not one. Unknown speaker labels, and
   multi-take recordings where the `sessionId` may resolve to a sibling take, go to the user.
3. **Pull each exchange.** From the transcript, never from a search fragment: the customer's words
   verbatim, the rep's reply that followed, and what the customer said next. Stamp each with
   recording title, date and timestamp as m:ss. The tool does not zero-pad: `1:5.000` is 1:05.
4. **Rank the answers for each objection.** First by what the customer said next: advanced
   (asked about next steps, booked a follow-up, brought in the decision maker) beats accepted,
   which beats stalled. Then by what the rep did: asked a question before answering, used a specific
   customer outcome, held price. Never rank on polish. Add the deal outcome if the user knows it.
   Show the top two with their evidence and let the user or sales leader pick.
5. **Set up each source call.** `../riverside-skills/scripts/new-recording SLUG` (exit 1: the
   folder exists, use it). If `canonical.md` has no `edit_id`, use the full-call edit from
   `platform_get_project` or `platform_list_edits` (skip `deleted: true`), or create one with
   `editing_create_edit_from_recording`. On `FAILED_PRECONDITION` do not retry: use an existing
   edit or ask the user to open one in Riverside. Fill `canonical.md`, then run
   `../riverside-skills/scripts/stale-check SLUG`. Exit 2: STOP, there is no canonical cut. Exit
   1: list the stale clips, which point at a re-edited call, and ask whether to rebuild them. If
   `editing_get_revision` differs from `canonical.md`, STOP and ask before re-declaring.
6. **Check the pairing. STOP on a splice.** The objection and the answer come from the same
   recording, the answer follows the objection, and nothing between them changes the subject. If
   the best answer came from a different call than the best objection wording, the clip uses the
   answer's own call and the objection that came before it there. The library can list wording
   from many calls. A clip never joins two. If a pairing fails, do not cut it, and say why.
7. **Cut the clip.**
   - `editing_clone_edit` from the canonical edit, `type: "HIGHLIGHT"`, titled with the objection
     and the source call. The canonical edit is never cut in place.
   - `editing_read_aligned_transcript` on the clone, `detail: "words"`, with `startMs` and `endMs`
     around the exchange. Clip times come from this playable timeline. Recording times from step 3
     only locate it, because cuts shift time.
   - Find the exchange's bounds: the customer's first word, and the rep's last word (or the
     customer's reply, if it shows the answer landed). Take their playable times from the
     word-level read above. Never apply a payload that covers the exchange itself: it would cut
     the very moment the clip is for.
   - `editing_resolve_transcript_selection`, `intent: "remove"`, with two `time_range`
     selections: 0 to the first word's start, and the last word's end to the timeline's end (the
     last row's `playableEndMs` in an unwindowed read). Only `remove` is available. Riverside's
     editing guide lists `keep` and `move` as rejected, so never plan around them.
   - `editing_cut_time_ranges` with the payload's input unchanged and `expectedRevision`, only
     when `readyToApply` is true.
   - Read the clone's transcript back. It starts on the customer's objection and ends on the rep's
     answer, in order, with nothing from elsewhere. If not, discard the clone and redo it.

   Removing fillers or pauses inside the exchange is fine (`editing_remove_fillers`,
   `editing_remove_pauses`). Removing words that change what either side said is not. Reps often
   watch on mute, so add captions with `editing_get_captions_presets`, then `editing_set_captions`.
8. **Demo snippets.** Same path. Find the segment by the phrases reps open a demo with ("let me
   show you", "I'll share my screen"), confirm it with the user, and clip one contiguous stretch.
9. **Log in two places.** In the source call's `publish-log.md`: asset `objection clip: NAME`,
   `built_from` the canonical `edit_id@revision`, destination `internal enablement`, reference the
   clip's edit id, state `draft` until the sales leader approves it. In
   `workspace/enablement/objection-library.md`, one row per clip: objection, customer words
   (verbatim, recording, timestamp), best answer (verbatim, rep role, timestamp), clip edit id,
   source (SLUG, recording id), `built_from`, and clearance (`internal`, or `public, cleared DATE`).
10. **Gate any clip leaving the company. STOP if it fails.** Sent to a prospect, put in a prospect
    deck or posted anywhere: run `../riverside-skills/scripts/clearance-check SLUG public` on the
    source call. Non-zero exit: STOP, show the script's stderr verbatim, keep the clip internal,
    and offer to draft the consent request to the customer who speaks on it. Exit 0: show the
    restrictions and honor them in picture and sound. If the restriction is "no company name" and
    they say it on the clip, cut it or the clip fails. Update the library's clearance cell. To
    send the file, export with `exports_create_export` (this repo's `ask` rule prompts before it
    runs), poll `exports_get_export` until `COMPLETED`, and download it from the Riverside exports
    folder: the MCP gives no link. Posting goes through `distribution-and-scheduling`, never here.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 1 Search | `search_recording_transcripts_exact` | Up to 20 phrases, OR-combined. Last 120 days unless `createdAfterDate` |
| 1 Search | `search_riverside` | TAKE filter. 10 results a page. Explicit `null` for unused parameters |
| 2 Read | `platform_get_transcript` | Recording time, one stamp per sentence. For locating only |
| 5 Source | `platform_get_project` · `platform_list_edits` · `editing_create_edit_from_recording` | Skip deleted edits. `FAILED_PRECONDITION` means use an existing edit |
| 5 Check | `editing_get_revision` | Confirms `canonical.md` still matches the edit |
| 7 Cut | `editing_clone_edit` | Branch the clip. Never cut the canonical edit |
| 7 Cut | `editing_read_aligned_transcript` → `editing_resolve_transcript_selection` → `editing_cut_time_ranges` | Word IDs in, cut points out. Apply only when `readyToApply` is true |
| 7 Polish | `editing_remove_fillers` · `editing_remove_pauses` · `editing_get_captions_presets` · `editing_set_captions` | Inside the exchange only |
| 10 Send | `exports_create_export` · `exports_get_export` | Renders on every call. No download link |

## Output
- **Writes:** one clip edit per objection or demo snippet in Riverside,
  `workspace/enablement/objection-library.md`, and a `publish-log.md` row per clip in each source
  call's folder
- **Prints:** objections searched, hits confirmed, the ranked answers per objection with
  evidence, pairings refused and why, clips cut with their edit ids, and each clip's clearance

## Rules & quality bar
- **One exchange, one recording, in order.** Step 6 refuses to splice a rep's answer onto a
  different customer's question
- **Customer-side words make the objection.** A rep naming an objection is not evidence of one
- **Rank by what the customer said next,** not by how smooth the rep sounded
- **Cut clips only on a clone of the canonical edit**
- **Quote only from transcripts**
- **Internal by default. Outside the company only through the script in step 10**
- **Every clip logged twice:** the source call's publish log and the library
- **Never share clips with `includePreviewUrl`.** It is a studio-wide token that cannot be
  revoked. Reps watch in Riverside or get the export

## Related skills
- Pairs with: `research-call-mining`, which reads the same calls for messaging evidence
- Pairs with: `customer-interview-engine`, when a customer on a call becomes a case study
- Feeds: `content-quality-gates`, for any prospect-facing clip and its copy
- Publishing: `distribution-and-scheduling`. Scoring marketing clips: `clip-selection`
