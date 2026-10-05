---
name: webinar-archive-mining
description: >-
  Old webinars, back catalog, evergreen webinar content, resurface a past webinar, "what can we
  reuse". Scores archived segments against an angle you supply, then routes clips and a blog
  post. A webinar you just ran is webinar-production.
---

# Webinar Archive Mining

Pulls the segments of old webinars that answer a question your market is asking now, cuts them
into social clips, and drafts a blog post per webinar. The archive is the input and the angle is
the filter: without an angle this skill has no way to rank anything, so it asks for one and stops
until it has it.

## When to use
- Webinar recordings from past quarters that nobody has opened since
- A campaign, launch or theme that old recordings already speak to
- "What can we reuse", "anything in the back catalog about X"
- A recording you just made belongs in `webinar-production`, not here

## Inputs
- **The angle, required.** The topic, campaign or question this quarter. Two to five phrases a
  speaker would actually say, not a category name. "Agent evaluation" is a category; "how do you
  know the agent is right", "evals", "hallucination" are phrases. Without them, STOP and ask
- **How far back.** A date, or "everything". This decides `createdAfterDate`
- **Which webinars.** One named recording, or the whole archive ranked
- **Workspace:** `./workspace`, or `RIVERSIDE_WORKSPACE` if set

## Workflow

1. **Pin the angle before searching.** Write it to `assets/angle.md`: the question, the phrases,
   and who the clips are for. Every candidate below is judged against this file, and a candidate
   that only matches the category rather than a phrase does not count as a match. If the user
   gave a category and no phrases, propose phrases and get them confirmed. No angle, no run.
2. **Set the window explicitly.** `search_recording_transcripts_exact` **defaults to the last 120
   days**, which excludes the entire archive this skill exists to search. Pass `createdAfterDate`
   on every call, set to the user's date or to a year well before the oldest webinar. Say the
   window out loud in the report, so a thin result is read as a narrow window rather than an
   empty archive.
3. **Find the candidates.** Run `search_recording_transcripts_exact` once per phrase with that
   window. Add `search_riverside` with `{"entity": "TAKE", "field": null, "value": null}` for
   fuzzier wording, 10 results a page. Keep only recordings that are webinars, confirming with
   `platform_get_project` when a title is ambiguous. List what you found: recording title, date,
   which phrases hit, how many times.
4. **Rank on evidence, and show the evidence.** One row per candidate segment in
   `assets/candidates.md`: recording, date, the phrase it matched, the verbatim line, and one
   sentence on how it answers the angle. Rank by how directly the segment answers the angle, and
   say so per row. Never score a segment the transcript does not support, and never rank on a
   paraphrase of what you assume the speaker meant. A segment nobody can read a line from is not
   a candidate. Show the ranked list and let the user pick before anything is cut.
5. **Declare a canonical cut per webinar, on a clone.** For each picked webinar, read
   `canonical.md` if the folder exists, or run `../riverside-skills/scripts/new-recording SLUG`
   (exit 1: the folder exists, use it). An old webinar's on-demand edit is usually published and
   linked, so **never cut it**. Run `editing_clone_edit` and declare the clone in `canonical.md`
   with a History row naming the angle. The published on-demand version stays untouched.
6. **Check the cut is current.** `editing_get_revision` on the clone, then
   `../riverside-skills/scripts/stale-check SLUG`. Exit 2: STOP, no canonical cut. Exit 1: assets
   from an older cut are logged, list them and ask whether this run replaces them.
7. **Read the transcript on the clone's timeline.** `editing_read_aligned_transcript` on the
   declared edit and revision, paged with `startMs` and `endMs`. Every timestamp from here on
   comes off this playable timeline. An archived webinar has usually been trimmed at least once,
   so raw recording time and the published cut no longer agree.
8. **Clips.** Hand `edit_id@revision`, the picked segments with their playable ranges, and the
   angle to `clip-selection`, which scores and cuts. This skill does not cut clips itself.
9. **Social copy.** Hand the approved clips to `social-asset-pack`, which writes the hook,
   caption, title and thumbnail brief per platform to `assets/social-copy.md`. Tell it the angle,
   so the copy frames a two-year-old segment around the question being asked now rather than
   around the original event. Never imply the webinar is recent: the copy names it as a past
   session, and a date in the caption is the simplest way to stay honest.
10. **Blog post, one per webinar.** Hand `edit_id@revision` and the angle to
    `transcript-to-written`, format `blog post`, output `assets/blog-post.md`. It reads the
    aligned transcript from step 7, not `platform_get_transcript`, so its quotes and timestamps
    match the clips and the published cut. Do not stitch several webinars into one post: each
    post stands on one recording, so every quote in it traces to one canonical edit.
11. **Clearance before anything leaves, not before drafting.** Draft and cut freely. Before the
    first clip or post goes out, list every speaker in the canonical transcript and run
    `../riverside-skills/scripts/clearance-check SLUG public`. Non-zero exit: STOP the external
    release, show stderr verbatim, and offer to draft the approval ask for each speaker. Exit 0:
    show the restrictions and honor them. **An old approval is the trap here.** Consent given for
    "the on-demand recording" years ago does not cover a new social clip of a guest's face. Read
    `approved_on` and `restrictions`, and when the scope does not plainly cover social, treat the
    speaker as uncleared and ask again. A guest who cannot be reached stays out of the clips, and
    the blog post quotes them only if the original approval covered written use.
12. **Gate and publish.** `content-quality-gates` judges the clips, the copy and the post.
    `distribution-and-scheduling` publishes or schedules. This skill never calls
    `social_upload_create`.
13. **Log.** A row per asset in `publish-log.md`: each clip, the social copy, the blog post.
    `built_from` is `edit_id@revision` from `canonical.md`. The asset cell carries the angle, so
    next quarter's run can see what this one already used. Done means `stale-check SLUG` exits 0.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 2 Window | `search_recording_transcripts_exact` | Defaults to the last 120 days. `createdAfterDate` on every call |
| 3 Find | `search_riverside` · `platform_get_project` | `TAKE` filter for spoken moments, 10 results a page |
| 5 Clone | `editing_clone_edit` | The published on-demand edit is never cut. Declare the clone |
| 6 Check | `editing_get_revision` | Confirms `canonical.md` still matches the clone |
| 7 Read | `editing_read_aligned_transcript` | Playable timeline. Page with `startMs` and `endMs` |
| Never | `platform_get_recording` with `includePreviewUrl` | Studio-wide, non-revocable share token |

## Output
- `assets/angle.md`, `assets/candidates.md` with the ranked evidence, `assets/social-copy.md`,
  `assets/blog-post.md` (one per webinar), the approved clips in Riverside off the declared clone,
  and a `publish-log.md` row per asset
- **Prints:** the search window, what each phrase matched, the ranked candidates with their
  verbatim lines, the clearance result per speaker with `approved_on`, and the `stale-check` result

## Rules & quality bar
- **No angle, no run.** Relevance needs something to be relevant to
- **Say the search window out loud.** The 120-day default is the quiet failure here
- **Rank on a line somebody said,** shown in the row. Never on an assumed meaning
- **Clone before cutting.** The published on-demand version is live and linked
- **An old approval is not a new one.** Check `approved_on` and scope before a guest is clipped
- **Never imply an old session is new.** The copy names it as a past webinar
- **One webinar per blog post,** so every quote traces to one canonical edit
- **Done means `stale-check` exits 0**

## Related skills
- Hands off to: `clip-selection`, `social-asset-pack`, `transcript-to-written`,
  `content-quality-gates`, `distribution-and-scheduling`
- A webinar you just ran: `webinar-production`. This skill is for the back catalog
- Setup and shared scripts: `riverside-skills`. Tool catalog and seams: `docs/riverside-mcp.md`
