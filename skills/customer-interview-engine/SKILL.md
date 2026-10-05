---
name: customer-interview-engine
description: >-
  Case study, customer interview, testimonial, customer story, pull quotes. Writes the interview
  guide, drafts everything off the recording, then the approval email the customer signs before
  any of it publishes. Clip scoring: clip-selection.
---

# Customer Interview Engine

One customer interview becomes a case study, testimonial pull-quotes and a clip shortlist. The
customer sees the finished package rather than a promise: everything is drafted, the quotes are
verified and the numbers are flagged, and the last thing this skill writes is the email that asks
them to approve the exact material. Two phases: before the interview (the question guide), and
after it (the drafts, then the approval ask).

Every asset here stays a draft in your workspace until the customer's written reply is saved
beside it. Nothing goes public without that reply.

## When to use
- A customer has agreed to an interview, or is about to
- A case study, testimonial or customer story from a recorded interview
- Pull-quotes or clip candidates for a customer's story

## Inputs
- **Before:** the customer company, the interviewee's name and role, the result the team hopes to
  show, and what the account team knows about the before-state
- **After:** the interview recording in Riverside and its folder `workspace/episodes/SLUG/`. The
  customer's written approval is an output of this skill rather than an input to it
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
   6. The result: "What improved, and how do you measure it?"
   7. What they'd tell a peer: "If someone in your role asked about us, what would you say?"

   Coach the interviewer on three habits. Ask for full-sentence answers that restate the
   question, because the interviewer's voice is cut out of clips. When a number comes up, ask how
   they measured it, so you know what to confirm later. After a strong answer, ask for it again in
   one line.
3. **Tell them what happens next, on the call.** No paperwork before the interview, but say the
   shape out loud: the call is recorded, you will draft the case study, and they will see the
   whole thing and approve it in writing before any of it is published. A verbal yes never
   substitutes for that written approval.

### After the interview

1. **Pin the source cut.** Find the recording with `platform_list_recordings`, `search_riverside`
   (TAKE filter, the customer's name) or `platform_get_project`. If two recordings or takes match
   the date, list them and ask. If `canonical.md` has no `edit_id`, use an existing edit from
   `platform_get_project` or `platform_list_edits` (skip `deleted: true`), or make one with
   `editing_create_edit_from_recording`. On `FAILED_PRECONDITION` do not retry: use an existing
   edit or ask the user to open one in Riverside. Confirm the edit with the user, then fill
   `recording_id`, `edit_id`, `revision`, `declared`, `declared_by` and `reason`.
2. **Check the cut is current.** Run `../riverside-skills/scripts/stale-check SLUG`. Exit 2:
   STOP, the canonical cut is not declared. Exit 1: assets from an older cut are logged. List them
   and ask whether this run replaces them. Then call `editing_get_revision` on the edit. If it
   differs from `canonical.md`, the edit changed after it was declared: STOP and ask whether to
   re-declare it, with a History row.
3. **Read the transcript on the canonical timeline.** `editing_read_aligned_transcript` on the
   canonical edit and revision, paged with `startMs` and `endMs` for a long interview. Save it to
   `assets/transcript.md` and confirm which speaker is the customer. Every timestamp in this skill
   comes from this playable timeline, never raw recording time, because cuts shift time.
4. **Draft the case study** to `assets/case-study-draft.md`: a headline, then Situation, What they
   tried before, What changed, and The result in their words. Each section is a short paragraph
   plus the customer's own quote with its timestamp. The narrative states nothing the customer
   did not say or the user did not supply. Mark user-supplied facts "(source: our team)".
5. **Verify every quote. Cut or label what fails.** For each quote, call
   `editing_resolve_transcript_selection` on the canonical edit and revision with
   `intent: "remove"`, the quote as a `text` selection, and the customer's `speaker` label. The
   resolver never changes the edit, and its payload is never run here. A resolved quote is
   verbatim, and its playable range is the timestamp. A null payload means the words are not in
   the transcript as written: cut the quote, or take it out of quote marks and label it
   "(paraphrase)". Verify each piece of an ellipsis quote. If the resolver rejects a long quote,
   split it at sentence breaks. Report checked, verified, cut and relabeled.
6. **Flag every number.** Every figure, percentage, multiple, duration, amount, and words such as
   "half", "double" or "twice" get `[confirm with customer]` right after them, in quotes and
   narrative alike. Customers round up in conversation: "cut onboarding in half" can be 38% in
   their own dashboard. Then scan the draft with
   `grep -nE '[0-9]|percent|half|double|twice|triple|third'`. Every hit that is a figure must
   carry the flag. A flag comes off only when the customer confirms the number in writing, saved
   into the folder beside the approval.
7. **Pull-quotes** to `assets/pull-quotes.md`: three to five verified quotes under 25 words, each
   with its timestamp and the attribution you intend to use (name, title, company, logo), which
   the customer confirms or narrows in step 10. Favor the specific before-and-after, the moment of
   decision, and the answer to the peer question. Run the same number scan on them.
8. **Clip shortlist** to `assets/clip-shortlist.md`, for `clip-selection`: three to six moments,
   each with the playable start and end from the resolver, the verbatim line, and why it might
   work. Head the file with `edit_id@revision`. This skill does not score or cut clips.
9. **Log everything as a draft.** Add a row per asset to `publish-log.md`: case study draft,
   pull-quotes, clip shortlist. `built_from` is `edit_id@revision` from `canonical.md`,
   `reference` is the file path, `state` is `draft`. The log has no clearance column, so the asset
   cell carries it: `case study draft (approval pending, sent DATE)`. Leave `clearance.md` at
   `status: pending` until the reply lands. Nothing in this folder is cleared yet, and the log is
   where that stays visible.
10. **Write the approval email. This is the last thing the skill produces.** Draft
    `assets/approval-request.md`: the case study, the pull-quotes with their attribution, a plain
    description of each shortlisted clip, and every `[confirm with customer]` figure listed out
    for them to confirm or correct. Send it from your own mail client, not from here.

    ```text
    Thanks again for your time on DATE. Here is everything we would like to publish from our
    conversation: the case study draft, the quotes we would use, and the moments we would cut
    as short video clips. Nothing goes out until you reply.

    Two things we need from you in writing:

    1. Your OK for COMPANY to use the recording of our DATE conversation, including your name,
       title, company name, logo and quotes, in a case study, on our website, in social posts
       and in sales materials. Reply "Approved" to confirm.
    2. The figures marked below. Please confirm each as written, or send me the right number.

    If you want anything left out, such as your last name, the logo, a particular figure or a
    specific clip, say so in your reply and we will hold to it.
    ```

    **When the reply arrives,** save it into the folder (for example `approval-DATE.md` with the
    email pasted in), confirm the file exists with `test -f`, and only then fill `clearance.md`:
    `status: cleared`, `scope: public`, `approved_by` (name, role, company), `approved_on` (the
    date of the reply), `evidence` (the saved file's path), `restrictions` (everything they ruled
    out, in their words). Then re-read every draft against each restriction, list how each was
    honored, take the flags off the figures they confirmed, and update the `publish-log.md` rows.
    A figure they did not confirm keeps its flag and stays out.

### The publish gate

Clearance is checked at the boundary where something becomes public, not before drafting. Before
the case study, a quote or a clip goes anywhere external, run
`../riverside-skills/scripts/clearance-check SLUG public`. Non-zero exit: STOP the external
release and show the script's reasons from stderr verbatim. `content-quality-gates` and
`distribution-and-scheduling` both run this check, so a draft with no saved approval cannot reach
a platform through them. Internal review inside your own team is fine at any point.

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
- **Before:** `assets/interview-guide.md`
- **After:** `assets/transcript.md`, `assets/case-study-draft.md`, `assets/pull-quotes.md`,
  `assets/clip-shortlist.md`, `assets/approval-request.md`, and three `publish-log.md` rows at
  `state: draft`
- **Once the reply lands:** the saved approval file, a filled `clearance.md`, confirmed figures
  with their flags removed, and the log rows updated with the restrictions
- **Prints:** quotes checked, verified, cut and relabeled, every flagged number, and once the
  reply is in, the clearance result and how each restriction was honored
- **At the publish boundary:** the `clearance-check` result. A non-zero exit stops the release

## Rules & quality bar
- **Draft first, approve last.** The customer approves finished material, not a plan to write it
- **The script decides clearance at the publish boundary.** A non-zero exit stops the release,
  whatever state the drafts are in
- **Verbatim or out of quote marks.** Every quote is verified against the canonical edit
- **Every number flagged until confirmed in writing.** The grep scan catches misses
- **Restrictions are rules,** applied to every draft as soon as the reply names them
- **Playable timestamps only,** from the canonical edit at the logged revision
- **No invented narrative.** Every claim traces to the customer's words or a marked team fact
- **Nothing publishes on a verbal yes.** Only the saved written reply clears it

## Related skills
- Feeds: `clip-selection`, which scores and cuts the shortlist, then `social-asset-pack`
- Feeds: `content-quality-gates`, which judges the case study and pull-quotes before public use
- Publishing: `distribution-and-scheduling`. This skill never publishes
- Pairs with: `research-call-mining`, for patterns across many calls with no attribution
