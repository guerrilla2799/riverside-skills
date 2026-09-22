---
name: customer-interview-engine
description: >-
  Case study, customer interview, testimonial, customer story, pull quotes. Writes the interview guide and consent ask, then drafts only after clearance-check passes. Clip scoring: clip-selection. Unattributed themes: research-call-mining.
---

# Customer Interview Engine

One customer interview becomes a case study, testimonial pull-quotes and a clip shortlist. The
hard part is permission, so a script decides it before a word is drafted. Two phases: before the
interview (the question guide and the written consent ask), and after it (the clearance gate,
verified quotes, flagged numbers, the log).

## When to use
- A customer has agreed to an interview, or is about to
- A case study, testimonial or customer story from a recorded interview
- Pull-quotes or clip candidates for a customer's story

## Inputs
- **Before:** the customer company, the interviewee's name and role, the result the team hopes to
  show, and what the account team knows about the before-state
- **After:** the interview recording in Riverside; its folder `workspace/episodes/SLUG/`; the
  customer's written approval saved into that folder and recorded in `clearance.md`
- **Workspace:** `./workspace`, or `RIVERSIDE_WORKSPACE` if set

## Workflow

### Before the interview

1. **Scaffold.** Run `../riverside-skills/scripts/new-recording SLUG`. Exit 1 means the folder
   exists: use it, do not recreate it. Exit 64 means a bad slug (lowercase, digits, hyphens). Book
   the session in Riverside. The MCP does not schedule recordings.
2. **Write the question guide** to `assets/interview-guide.md`. One lead question per block, two
   follow-ups each:
   1. Situation: "Walk me through your role and what your team owned when this started."
   2. What they tried before: "What were you doing about it before us?"
   3. Why that failed: "Where did that break down, and what did it cost you?"
   4. The decision: "What made you look, why then, and who else did you consider?"
   5. What changed: "What is different in a normal week now?"
   6. The result: "What moved, and how do you measure it?"
   7. What they'd tell a peer: "If someone in your role asked about us, what would you say?"

   Coach the interviewer on three habits. Ask for full-sentence answers that restate the
   question, because the interviewer's voice is cut out of clips. When a number comes up, ask how
   they measured it, so you know what to confirm later. After a strong answer, ask for it again in
   one line.
3. **Write the consent ask** to `assets/consent-request.md`. It is sent by email before the call
   ends, with a reply asked for on the spot. A verbal yes on the call does not count.

   ```text
   Thanks for today. To use this conversation publicly we need your OK in writing. Please reply
   "Approved" to confirm that COMPANY may use the recording of our DATE conversation, including
   your name, title, company name, logo and quotes, in a case study, on our website, in social
   posts and in sales materials. You will see the draft before anything is published. If you
   want anything left out, such as your last name, the logo or specific numbers, say so in your
   reply and we will hold to it.
   ```
4. **Record the evidence.** Save the reply into the folder (for example `approval-DATE.md` with
   the email pasted in) and confirm the file exists with `test -f`. Only then fill `clearance.md`:
   `status: cleared`, `scope: public`, `approved_by` (name, role, company), `approved_on` (the
   date of the reply), `evidence` (the saved file's path), `restrictions` (everything they ruled
   out, in their words). No saved written reply means `status` stays `pending`.

### After the interview

1. **Clearance gate. Always step 1 after the interview.** Find the slug in
   `workspace/episodes/` by customer and date. If none matches, or more than one does, ask. Run
   `../riverside-skills/scripts/clearance-check SLUG public`.
   - **Non-zero exit: STOP.** Show the script's reasons from stderr verbatim. Draft nothing: no
     case study, no quotes, no clip list, no summary of the interview. Do not pull the
     transcript. Offer to draft the approval request email from `assets/consent-request.md`.
   - **Exit 0:** show the `Restrictions:` line from stdout. Each restriction becomes a rule for
     every draft below (first name only, no logo, no revenue figures). If there is none, say so.
2. **Pin the source cut.** Find the recording with `platform_list_recordings`, `search_riverside`
   (TAKE filter, the customer's name) or `platform_get_project`. If two recordings or takes match
   the date, list them and ask. If `canonical.md` has no `edit_id`, use an existing edit from
   `platform_get_project` or `platform_list_edits` (skip `deleted: true`), or make one with
   `editing_create_edit_from_recording`. On `FAILED_PRECONDITION` do not retry: use an existing
   edit or ask the user to open one in Riverside. Confirm the edit with the user, then fill
   `recording_id`, `edit_id`, `revision`, `declared`, `declared_by` and `reason`.
3. **Check the cut is current.** Run `../riverside-skills/scripts/stale-check SLUG`. Exit 2:
   STOP, the canonical cut is not declared. Exit 1: assets from an older cut are logged. List them
   and ask whether this run replaces them. Then call `editing_get_revision` on the edit. If it
   differs from `canonical.md`, the edit moved after it was declared: STOP and ask whether to
   re-declare it, with a History row.
4. **Read the transcript on the canonical timeline.** `editing_read_aligned_transcript` on the
   canonical edit and revision, paged with `startMs` and `endMs` for a long interview. Save it to
   `assets/transcript.md` and confirm which speaker is the customer. Every timestamp in this skill
   comes from this playable timeline, never raw recording time, because cuts shift time.
5. **Draft the case study** to `assets/case-study-draft.md`: a headline, then Situation, What they
   tried before, What changed, and The result in their words. Each section is a short paragraph
   plus the customer's own quote with its timestamp. The narrative states nothing the customer
   did not say or the user did not supply. Mark user-supplied facts "(source: our team)".
6. **Verify every quote. Cut or label what fails.** For each quote, call
   `editing_resolve_transcript_selection` on the canonical edit and revision with
   `intent: "remove"`, the quote as a `text` selection, and the customer's `speaker` label. The
   resolver never changes the edit, and its payload is never run here. A resolved quote is
   verbatim, and its playable range is the timestamp. A null payload means the words are not in
   the transcript as written: cut the quote, or take it out of quote marks and label it
   "(paraphrase)". Verify each piece of an ellipsis quote. If the resolver rejects a long quote,
   split it at sentence breaks. Report checked, verified, cut and relabeled.
7. **Flag every number.** Every figure, percentage, multiple, duration, amount, and words such as
   "half", "double" or "twice" get `[confirm with customer]` right after them, in quotes and
   narrative alike. Customers round up in conversation: "cut onboarding in half" can be 38% in
   their own dashboard. Then scan the draft with
   `grep -nE '[0-9]|percent|half|double|twice|triple|third'`. Every hit that is a figure must
   carry the flag. A flag comes off only when the customer confirms the number in writing, saved
   into the folder beside the approval.
8. **Pull-quotes** to `assets/pull-quotes.md`: three to five verified quotes under 25 words, each
   with its timestamp and the attribution the restrictions allow. Favor the specific
   before-and-after, the moment of decision, and the answer to the peer question. Run the same
   number scan on them.
9. **Clip shortlist** to `assets/clip-shortlist.md`, for `clip-selection`: three to six moments,
   each with the playable start and end from the resolver, the verbatim line, and why it might
   work. Head the file with `edit_id@revision`. This skill does not score or cut clips.
10. **Check restrictions, then log.** Re-read every draft against each restriction and list how
    each was honored. Add a row per asset to `publish-log.md`: case study draft, pull-quotes, clip
    shortlist. `built_from` is `edit_id@revision` from `canonical.md`, `reference` is the file
    path, `state` is `draft`. The log has no clearance column, so the asset cell carries it:
    `case study draft (cleared public DATE; restrictions: first name only)`.
11. **Send it back to the customer.** Draft `assets/review-request.md`: the case study, with every
    flagged number listed for them to confirm. Nothing is final until they reply in writing.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| Find | `platform_list_recordings` · `search_riverside` · `platform_get_project` | TAKE filter on search. Ask when two takes match |
| Pin | `platform_list_edits` · `editing_create_edit_from_recording` | Skip `deleted: true`. `FAILED_PRECONDITION` means use an existing edit |
| Check | `editing_get_revision` | Cheap check that `canonical.md` still matches the edit |
| Read | `editing_read_aligned_transcript` | Playable timeline. Page with `startMs` and `endMs` |
| Verify | `editing_resolve_transcript_selection` | Read-only. A null payload means the quote is not verbatim |
| Never | `platform_get_recording` with `includePreviewUrl` | Studio-wide, non-revocable share token |

## Output
- **Before:** `assets/interview-guide.md`, `assets/consent-request.md`, and a filled
  `clearance.md` once the written reply is saved
- **After:** `assets/transcript.md`, `assets/case-study-draft.md`, `assets/pull-quotes.md`,
  `assets/clip-shortlist.md`, `assets/review-request.md`, and three `publish-log.md` rows
- **Prints:** the clearance result and restrictions, quotes checked, verified, cut and relabeled,
  every flagged number, and how each restriction was honored
- **On a failed gate:** the script's reasons, and an offer to draft the approval request. Nothing
  else

## Rules & quality bar
- **The script decides clearance, first thing after the interview.** A failed check ends the run with nothing drafted
- **Verbatim or out of quote marks.** Every quote is verified against the canonical edit
- **Every number flagged until confirmed in writing.** The grep scan catches misses
- **Restrictions are rules,** checked draft by draft before anything is logged
- **Playable timestamps only,** from the canonical edit at the logged revision
- **No invented narrative.** Every claim traces to the customer's words or a marked team fact
- **The customer sees the draft first.** Public use waits for their reply

## Related skills
- Feeds: `clip-selection`, which scores and cuts the shortlist, then `social-asset-pack`
- Feeds: `content-quality-gates`, which judges the case study and pull-quotes before public use
- Publishing: `distribution-and-scheduling`. This skill never publishes
- Pairs with: `research-call-mining`, for patterns across many calls with no attribution
