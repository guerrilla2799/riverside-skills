---
name: transcript-to-written
description: >-
  Use for "turn this call into a blog post", "newsletter from this episode", "LinkedIn text post from the transcript", "write a recap". Long-form from platform_get_transcript, quotes verbatim with timestamps. Clip copy is social-asset-pack.
---

# Transcript to Written

Long-form writing from a recording: a newsletter issue, a blog post, a LinkedIn text post, or an internal recap. Every quote is verbatim and carries the recording name and timestamp, paraphrase is labeled as paraphrase, and no number appears that nobody said. The draft goes to `content-quality-gates` before anyone treats it as finished.

## When to use
- A call, interview or episode holds an argument worth writing up
- Someone wants the newsletter or blog version of an episode
- The team needs a written recap of a call they missed
- Not for copy that rides on a clip (`social-asset-pack`), a customer case study (`customer-interview-engine`), or show notes and chapters (`podcast-show-notes-and-chapters`)

## Inputs
- `workspace/episodes/SLUG/`, created with `../riverside-skills/scripts/new-recording SLUG`. External drafts also need `canonical.md` filled in
- From the user: the format, the audience (internal or external), the angle, a target length, and the path to their voice or style file if they have one
- Recordings stored in other tools are out of reach here. Upload them in the Riverside app. `research-call-mining` covers transcript exports
- Script paths are relative to this skill's base directory. The workspace is `./workspace` unless `RIVERSIDE_WORKSPACE` is set

## Workflow

1. **Pin the format and the audience.** Newsletter issue, blog post, LinkedIn text post, or internal recap, and internal or external. If the text will be posted with a clip, STOP and route to `social-asset-pack`.
2. **Pick the source.**
   - `canonical.md` has `edit_id` and `revision`: read `editing_read_aligned_transcript` on that edit with `revision` set to the declared one. Timestamps are on the canonical cut's playable timeline, which is what readers can play. `built_from` = `edit_id@revision`.
   - No canonical cut, internal draft: `platform_get_transcript` with the recording's `sessionId`. It returns prose with one `Speaker (m:s.mmm)` stamp per sentence, timed from the start of the session. Minutes and seconds are not zero-padded: `1:5.000` is 1:05. A speaker with no session offset is timed from their own track start, so their times do not line up with anyone else's, so say so beside any quote from them. `built_from` = `recording:RECORDING_ID`, which `stale-check` flags once a canonical cut is declared.
   - No canonical cut, external draft: STOP. Declare the cut first: list edits with `platform_get_project` and ask which is canonical, or create one with `editing_create_edit_from_recording`. On `FAILED_PRECONDITION` use `platform_list_edits`, and if no edit exists, ask the user to create one in Riverside. An editor may have cut material on purpose, a guest's request included, and reader-facing timestamps must match the published cut, because cuts shift time.
3. **Identify the speakers.** List each speaker label with one sample line and ask which is the user and what role each other speaker holds. Never infer a customer's name or company from context, and never resolve a generic label such as `speaker-0` by guessing.
4. **Clear before drafting.**
   - External draft that quotes or draws on anyone but the user: run `../riverside-skills/scripts/clearance-check SLUG public`.
   - Internal recap with anyone but the user: run `../riverside-skills/scripts/clearance-check SLUG internal`.
   - The user is the only speaker: no check.

   Non-zero exit: STOP. Report the script's stderr verbatim and offer to draft the consent request instead (`customer-interview-engine` for customers, `podcast-guest-ops` for guests). For an internal recap, tell the user that team use needs `status: internal-only` or `cleared` in `clearance.md`. The user sets it. This skill never edits `clearance.md` to make a check pass. Exit 0: print the `Restrictions:` line and honor it in every sentence and attribution.
5. **Load the voice.** Ask for the path to the user's voice or style file. Read it where it lives at draft time. Never paste a copy into this skill or the workspace: skills point at knowledge files and do not carry copies of them. With no file, write plain, direct and short.
6. **Build the quote bank before the draft.** Pull every line the piece might use into a list with:
   - the verbatim text, and the speaker's role
   - the recording name and timestamp
   - the clock: `edit` for the canonical timeline, `rec` for session time

   Then verify each quote with `search_recording_transcripts_exact`, one quote per call, with `createdAfterDate` set to the day before the recording, because the tool only looks back 120 days by default. Split long quotes at punctuation and search each run (256 characters at most). A quote with no match in this recording's `sessionId` is not verbatim: correct it from the transcript or relabel it as paraphrase.
7. **Hold the numbers.** Every number in the draft traces to a quote-bank line or to a source the user supplied, cited. No rounding, no totals the speaker did not state, no invented outcome. A number a customer or guest said aloud is marked `confirm with customer` until it is confirmed in writing.
8. **Draft to the format.**

   | Format | Shape |
   |---|---|
   | Newsletter issue | Subject line options, the hook, the argument carried by the speaker's own words, what the reader does with it |
   | Blog post | Title, sections that each make one claim, quotes as the evidence, timestamps pointing at the published cut where one exists |
   | LinkedIn text post | Hook inside the first ~140 characters (all that shows before "see more"), 3,000 characters at most. `social_upload_create` needs a clip or an edit, so a text-only post goes up on LinkedIn by hand |
   | Internal recap | Marked internal at the top. What was said, decisions, open questions, next steps with owners |

   Paraphrase carries `(paraphrase)` next to it. Attribution follows the clearance restrictions: first name only, role only, or no company, as ruled.
9. **Write the file and log it.** `workspace/episodes/SLUG/assets/written-FORMAT-YYYY-MM-DD.md`: the draft, then the Sources section in the shape under Output, then open flags. Add a `publish-log.md` row: date, `written FORMAT: TITLE`, `built_from` from step 2, destination (newsletter, blog, LinkedIn by hand, internal), reference = the file path, state `draft`.
10. **Hand to `content-quality-gates`.** It judges, and this skill revises only what the verdict names.
    - After a pass and the user's sign-off, set the row to `approved`.
    - An open `confirm with customer` flag keeps it at `draft`.
    - This skill publishes nothing, and nothing it writes goes through `social_upload_create`.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 2 | `editing_read_aligned_transcript` | Canonical edit at the declared revision. Timestamps on the published timeline |
| 2 | `platform_get_transcript` | Internal drafts with no canonical cut. Session time, minutes and seconds unpadded |
| 2 | `platform_get_project` · `editing_create_edit_from_recording` · `platform_list_edits` | Declaring a canonical cut. Fall back to `platform_list_edits` on `FAILED_PRECONDITION` |
| 6 | `search_recording_transcripts_exact` | Verbatim check per quote. Pass `createdAfterDate`, or it only searches 120 days back |

## Output
- Writes: `workspace/episodes/SLUG/assets/written-FORMAT-YYYY-MM-DD.md` with the draft, its Sources section and open flags. One `publish-log.md` row
- Shows: the draft, the quote bank with each quote's verification result, every `confirm with customer` flag, and any restriction that changed the copy

Sources section, one entry per quote:

```
Q3  "VERBATIM QUOTE"
    SPEAKER ROLE · RECORDING NAME · 14:02 (edit) · exact match: yes
Q4  (paraphrase) SUMMARY OF WHAT WAS SAID
    SPEAKER ROLE · RECORDING NAME · 1:05:31 (rec)
Flag: Q3 contains a spoken number, confirm with customer
```

## Rules & quality bar
- **Clearance runs before drafting,** and the script decides. A non-zero exit stops the draft
- **Every quote is verbatim** with recording name, timestamp and clock, and passed the exact-match check. Anything else is labeled paraphrase
- **No invented numbers,** quotes, outcomes or recordings. Spoken customer numbers stay flagged until confirmed in writing
- **External drafts cite the canonical cut,** never raw material an editor removed
- **Read the voice file by path**
- **Nothing is approved until `content-quality-gates` passes it** and the user signs off

## Related skills
- Requires: `riverside-skills` for the scripts and templates
- Feeds: `content-quality-gates`
- Pairs with: `social-asset-pack` when the piece gets a clip, and `research-call-mining` for themes across many calls
- Boundary: `customer-interview-engine` owns case studies, and `podcast-show-notes-and-chapters` owns show notes
- See also: `docs/riverside-mcp.md`
