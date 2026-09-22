---
name: clip-selection
description: >-
  Use for "find clips", "best moments", "what should we clip", "score this recording". Scores every candidate on a gated scorecard, returns the top five, builds approved clips via editing_clone_edit. Clip copy is social-asset-pack.
---

# Clip Selection

Which moments earn a clip, decided by a written scorecard instead of a gut read. Reads the canonical edit, runs every candidate through three gates, scores the survivors on five dimensions, and returns the top five with the reason each one made the cut. Nothing is built until the user approves specific clips, and each approved clip becomes its own edit, so the canonical cut is never written to.

## When to use
- A recording is done and someone asks what is worth clipping
- A clip list exists and nobody can say why those moments won
- Before `social-asset-pack`, which only writes copy for clips this skill built and logged
- Not for captions text, titles or hooks (`social-asset-pack`), and not for posting (`distribution-and-scheduling`)

## Inputs
- `workspace/episodes/SLUG/canonical.md` with `edit_id` and `revision`. The folder comes from `../riverside-skills/scripts/new-recording SLUG`. The cut is declared by `podcast-episode-pipeline` or in step 1
- `workspace/episodes/SLUG/clearance.md`, read only through `clearance-check`
- Optional: `assets/clip-shortlist.md` from `customer-interview-engine`, headed with the `edit_id@revision` its times were read from
- From the user: target platforms, intended use (public or internal), and which speaker is them
- Script paths are relative to this skill's base directory. The workspace is `./workspace` unless `RIVERSIDE_WORKSPACE` is set

## Workflow

1. **Confirm the canonical cut.** Read `canonical.md`. No folder: run `../riverside-skills/scripts/new-recording SLUG`. Empty `edit_id`: list the recording's edits with `platform_get_project` (drop `deleted: true`), show them, and ask which is canonical. Never pick one. Record it with a History row. Then call `editing_get_revision` on the edit. If the revision differs from `canonical.md`, STOP: tell the user the canonical cut changed since it was declared, and ask whether to re-declare it before scoring. A clone copies the current timeline, so scoring one revision and building from another produces clips nobody scored.
2. **Get the IDs.** `platform_get_edit` on the canonical edit gives the project and source recording (`take`). `platform_get_recording` on that recording gives the `studioId` the captions step needs. Never pass `includePreviewUrl`: it returns a studio-wide share token that cannot be revoked.
3. **Read the transcript on the edit's timeline.** `editing_read_aligned_transcript` with `editId` and `includeParalinguistics: true`. Every timestamp this skill produces comes from this playable timeline, never from raw recording time, because cuts shift time. If the paralinguistics list is empty, energy is scored from the words alone and each energy score is marked `(text)`.
4. **Identify the speakers.** List each speaker label with one sample line and ask which is the user and what role each other speaker holds (customer, guest, colleague). Generic labels such as `speaker-0` are never resolved by guessing from content.
5. **Check what already exists.** `search_riverside` with `searchTerm: null`, `filters: [{entity: "CLIP", field: null, value: null}]` and `nextSearchAfter: null`, then page with the returned `nextSearchAfter`. Keep results whose `sessionId` or `projectId` matches this recording. Clip results match on title and carry no start or end offsets, so compare by topic and duration, and read `publish-log.md` for clips already built. Results can include full-length edits, so ignore anything close to the recording's full duration. Show likely duplicates to the user. Do not drop a moment silently.
6. **List every candidate.** Walk the sentence rows. A candidate is one contiguous run holding one complete thought: a question stays with its answer, a setup with its payoff. Zoom in with `detail: "words"` plus `startMs` and `endMs` to find the first and last word. If a stronger opening line sits one or two sentences in, start there. Moments on a shortlist join the list and get gated and scored like any other. If the shortlist's `edit_id@revision` differs from `canonical.md`, its times are stale, so find each line again by its text. Every candidate goes in the scorecard, including the ones that fail.
7. **Run the gates, in order. A fail removes the candidate.**
   - **Stands alone.** A cold viewer follows it with no prior context. Fails on "like I said", an unexplained "that", or a pointer to an earlier segment.
   - **One idea.** One claim, one story, or one answer. Two ideas make two candidates or none.
   - **Clearance allows the intended use.** For any clip that will leave the company and includes anyone but the user, run `../riverside-skills/scripts/clearance-check SLUG public` once for the recording. For internal-only clips, run `../riverside-skills/scripts/clearance-check SLUG internal`. If it exits non-zero, STOP: report the script's stderr verbatim, name the candidates it blocks, and offer to draft the consent request instead (`customer-interview-engine` for customers, `podcast-guest-ops` for guests). Continue with user-only moments only if the user says so. On exit 0, print the `Restrictions:` line and fail every candidate that breaks it, such as a revenue figure when figures are ruled out.
8. **Score each survivor 1–5 on five dimensions.** Length caps come from `social_get_publishing_guidelines` for each target platform, read at run time.

   | Dimension | 5 | 3 | 1 |
   |---|---|---|---|
   | Hook, first 3 seconds (about 8 words) | A claim, number or tension that works cold | On topic, still warming up | Filler or setup |
   | Specificity | A number, a name, or a story with a before and after | A concrete example, thin on detail | An opinion or general principle |
   | Energy | Laughter, emotion or prosody events inside the range | Steady, clear delivery | Flat, no events, trailing off |
   | Length fit | Inside every target platform's duration cap | Inside some of them | Inside none without losing the payoff |
   | Payoff | Lands, and the clip ends within about 2 seconds | Lands, then trails | Lands outside the range, or never |

   Total out of 25. On a tie, a customer or guest describing their own situation outranks the host explaining a framework. After that, higher Specificity wins, then the shorter clip.
9. **Flag numbers.** Any number a customer or guest says inside a clip is marked `confirm with customer` until it is confirmed in writing. The clip can still rank. `social-asset-pack` carries the flag forward and will not approve copy that uses the number.
10. **Report, then STOP.** Write `assets/clip-scorecard.md`. Show the top five and the next five as runners-up. Then wait. Nothing is built until the user names the clips they approve. "Looks good" is not approval of specific clips, so ask which.
11. **Build each approved clip as its own edit.**
    - `editing_get_revision` on the canonical edit. If it no longer matches `canonical.md`, STOP as in step 1.
    - `editing_clone_edit` with `sourceEditId` set to the canonical `edit_id` and a title of `SLUG clip-NN SHORT-TITLE`. Use the returned `editId` and `revision` from here on.
    - `editing_read_aligned_transcript` on the clone with `detail: "words"` around the clip. Word and span IDs are scoped to one edit and revision, so IDs from the canonical edit do not resolve on the clone. The clone's timeline starts identical, so the scored timestamps still hold.
    - `editing_resolve_transcript_selection` on the clone with `intent: "remove"` and two `time_range` selections: 0 to the clip's first-word start, and the clip's last-word end to the end of the timeline. A range may not run past the timeline's duration. If the resolver warns out of bounds, take the bound from the warning and resolve again. Only `remove` is available (`keep` and `move` are rejected). If `readyToApply` is not true or `payload` is null, STOP for this clip, show the warnings, and do not guess cut points.
    - `editing_cut_time_ranges` with `payload.input` exactly as returned.
    - Re-read the clone's transcript. If the first or last words are not the approved ones, or the duration is off by more than a second, STOP and report.
12. **Caption it.** `editing_get_captions_presets` with the `studioId`. Use the `brandCaptionsList` entry with `isDefault: true`, else the lowest `sortOrder`. Read `brandCaptions` only when `brandCaptionsList` is empty. Otherwise ask the user to pick from `presets`. Never invent a style. Then `editing_set_captions` with `presetId`, `studioId`, `show: true` and `expectedRevision` from the last read.
13. **Flag the format.** The verified tool catalog has no call that changes an edit's aspect ratio. Per the guidelines, Shorts, TikTok, Instagram and Facebook take only vertical video, some also square, so a landscape clip fails there. Tell the user which clips need a vertical format set in the Riverside editor, and to do it before `social-asset-pack` runs: a format change bumps the clip's revision, and the copy is written against the revision it read.
14. **Log each clip.** One `publish-log.md` row per clip: date, `clip-NN: SHORT-TITLE`, `built_from` = the canonical `edit_id@revision` from `canonical.md`, destination `riverside edit`, reference = the clip's `editId`, state `draft`. Then hand the clips to `social-asset-pack`.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 1 | `platform_get_project` · `editing_get_revision` | List edits to declare canonical. Cheap check that the declared revision is current |
| 2 | `platform_get_edit` · `platform_get_recording` | Project, source recording, `studioId`. No `includePreviewUrl` |
| 3, 6 | `editing_read_aligned_transcript` | `includeParalinguistics: true` for energy, `detail: "words"` for cut-grade boundaries |
| 5 | `search_riverside` | CLIP filter, explicit nulls, 10 results a page |
| 8 | `social_get_publishing_guidelines` | Duration and aspect-ratio caps per platform |
| 11 | `editing_clone_edit` | New edit per clip. The canonical edit is untouched |
| 11 | `editing_resolve_transcript_selection` → `editing_cut_time_ranges` | `intent: "remove"`. Run the payload only when `readyToApply` is true |
| 12 | `editing_get_captions_presets` → `editing_set_captions` | Brand preset first. Pass `expectedRevision` |

## Output
- Writes: `workspace/episodes/SLUG/assets/clip-scorecard.md` (every candidate, gate results, scores, flags), one Riverside edit per approved clip, one `publish-log.md` row per clip
- Shows, per top-five clip:

```
1. clip-NN   M:SS–M:SS (NN s)   SPEAKER ROLE
   "FIRST LINE, VERBATIM FROM THE TRANSCRIPT"
   Gates pass · Hook 5 · Specific 4 · Energy 4 · Length 5 · Payoff 4 = 22/25
   Why: ONE SENTENCE ON WHY IT MADE THE CUT
   Flags: confirm with customer (NUMBER) · needs vertical format · possible duplicate of CLIP NAME
```

- Then the runners-up, one line each with their total, and every gate failure counted by gate

## Rules & quality bar
- **Gates before scores.** A candidate that fails a gate is never scored into the top five
- **The script decides clearance.** A non-zero `clearance-check` exit stops the run, and the model does not overrule it
- **Timestamps come from the canonical edit's playable timeline,** never raw recording time, because cuts shift time
- **Every candidate is written down,** failures included. A list of winners alone cannot be audited
- **Customer voice beats a host framework on a tie**
- **No build without approval naming specific clips,** and never a write to the canonical edit
- **No invented lines.** Every quoted line is copied from the transcript with its timestamp. Spoken numbers stay flagged until confirmed in writing

## Related skills
- Requires: `riverside-skills` for the scripts and templates, plus a declared canonical cut (`podcast-episode-pipeline` declares it for episodes)
- Feeds: `social-asset-pack` for copy, then `distribution-and-scheduling` for the calendar
- Pairs with: `sales-enablement-clips` for internal clips aimed at reps, and `content-quality-gates` for the check before anything ships
- See also: `docs/riverside-mcp.md`
