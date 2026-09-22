---
name: content-quality-gates
description: >-
  Use for "check this before it goes out", "is this ready to post", "review this clip, title or caption". Gates clearance, verbatim quotes, numbers, staleness and platform limits, then scores. Judges only: the skill that made the asset fixes it.
---

# Content Quality Gates

The judge for everything that leaves the building: clips, thumbnail briefs, titles, captions, show notes, case studies and written posts. Hard gates run first, only what clears them gets scored, and the verdict comes back as PASS or REVISE with numbered fixes. It never writes or rewrites the asset it judges. The skill that made the asset applies the fixes, then the judge runs again.

## When to use
- Before any asset built from a recording goes to a platform, a customer, a prospect or a newsletter
- When a skill in this pack hands over a finished draft or package
- When the user asks whether something is ready, or whether a quote or number will hold up
- Again after every round of fixes. A fix can break a quote or push a caption over its cap

## Inputs
- **The asset**, as a file in `workspace/episodes/SLUG/assets/` or pasted text. For a package, every piece
- **SLUG** of the recording it came from, with `canonical.md`, `clearance.md` and `publish-log.md` in that folder
- **Destination** per asset: YouTube, YouTube Shorts, TikTok, Instagram, Facebook, LinkedIn, X, newsletter, blog or sales team. Ask if missing
- **Sponsored or partner:** yes or no. Ask if missing. Never infer it from the copy
- **Your voice file**, optional: the path to your own voice or style guide
- The scripts run from the folder that holds `workspace/`, called by their path from this skill's base directory. They find the workspace from the current directory or `RIVERSIDE_WORKSPACE`, never from the skill folder

## Workflow

### 1. Set up the review
Name each asset and its type. Then pick the review mode. The context that wrote a thing is the worst judge of it, because it reads its own intent instead of the text. If a subagent is available, give it only the asset, the SLUG, the destinations and this file, never the drafting conversation, and mark the verdict `review: independent`. If not, continue here and mark it `review: same-context`. Never label a same-context review independent.

### 2. Run the gates
Each gate passes or fails. Any FAIL makes the verdict REVISE with no score. Run all six so one cycle returns the full fix list, unless a gate says STOP. Gates A and D run once per recording, the rest once per asset.

**A. Clearance.** For any asset that features a customer, prospect or guest, and whenever you cannot tell who speaks in it, run `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP: verdict REVISE, the script's stderr quoted verbatim as fix 1, and the consent request routed to `customer-interview-engine` (customers) or `podcast-guest-ops` (guests). Nothing else is worth fixing until written approval exists. On exit 0, read every restriction printed on stdout and check the asset against each one (first name only, no logo, no revenue figures). A breach is a FAIL.

**B. Quotes.** Every span shown as someone's words (quote marks, a pull quote, a quote card, on-screen text attributed to a speaker) carries the recording name and a timestamp. Missing either is a FAIL. Then verify each one:
1. Run `search_recording_transcripts_exact` with the quote as a search term. A term holds 256 characters at most, so split a longer quote into consecutive phrases. The tool covers only the last 120 days by default. For an older recording, pass `createdAfterDate` set to a date before it was recorded (the date is on `platform_get_recording`). Confirm the match is the cited recording, since a multi-take session can match a sibling take
2. No match: read the transcript at the cited time, with `platform_get_transcript` for recording time or `editing_read_aligned_transcript` with `startMs` and `endMs` for a time on the canonical cut. The quote passes only if the words appear in that order with nothing dropped except filler (um, uh, a stutter), and any other cut is marked with an ellipsis and keeps the meaning. Anything else is a FAIL: quote it exactly, or drop the quote marks and paraphrase
3. The cited time falls inside the sentence that holds the words. If the user says the transcript misheard, they check the audio at that time and the verdict notes `checked against audio by NAME`. The judge never decides that alone

**C. Numbers.** List every number in the asset: money, percentages, counts, durations, dates, rankings. Each traces to a transcript timestamp or a written source with a path or link. A count the asset proves on its face (a title that says three over a clip that lists three) needs neither. A number a customer, prospect or guest said about their own business passes only when it is marked confirmed in writing, with the path to that confirmation. One still flagged "confirm with customer" is a FAIL for anything leaving the building. So is rounding, turning "about a third" into "33%", or dropping the speaker's qualifier (roughly, in one region, last quarter).

**D. Staleness.** Run `../riverside-skills/scripts/stale-check SLUG`.
- Exit 1: FAIL. List the stale rows it prints. If this asset is one, the fix is a rebuild from the current cut by the skill that made it. If only other assets are stale, the fix is `podcast-recut-and-republish`, or marking the old rows `superseded` or `retired` once replaced
- Exit 2: FAIL. No canonical cut is declared, or a file is missing. The fix is declaring the canonical cut (`podcast-episode-pipeline`) first
- Exit 0: confirm this asset has its own row in `publish-log.md`. No row is a FAIL, because `stale-check` cannot see an asset nobody logged. Then call `editing_get_revision` on the `edit_id` in `canonical.md`. A revision other than the one in `canonical.md` means the cut changed after it was declared. That is a FAIL, and the user decides whether to re-declare

**E. Disclosure.** If the asset is sponsored, paid, gifted, affiliate or made with a partner, the disclosure sits in plain words in the first line ("Sponsored by NAME", "Paid partnership with NAME"), inside the part the platform shows before the fold. `social_get_publishing_guidelines` states that fold per platform. A TikTok post plan also sets `isBrandedContent` to true. A disclosure only at the end, only in hashtags past the fold, or only in a comment is a FAIL. Not sponsored: n/a.

**F. Platform limits.** For each platform destination, call `social_get_publishing_guidelines` with `platform` (pass `YouTube` for Shorts, which has its own section in the result). Call `social_get_connected_platforms` for per-account limits such as the X character limit and TikTok's maximum duration, using the `studioId` from `platform_get_recording`. Check text, title, hashtag and mention counts against the caps, counted the way the guidelines say. Check clip duration, measured on the canonical edit's playable timeline, and aspect ratio against the video constraints. For YouTube, call `editing_get_export_publish_data` on the edit: a non-null `youtubeCode` is a likely copyright match and a FAIL until the user decides. Any cap exceeded is a FAIL naming the limit and the measured value. Newsletter, blog and sales team: n/a.

If any gate failed, skip to step 5.

### 3. Score it
Score each dimension 1–5 against the anchors. PASS needs every dimension at 4 or 5. A dimension at 3 or below becomes a fix that says what would move it to 5. Title options and clip candidates are scored one at a time. Clip points, chapter marks and thumbnail frames are read on the canonical edit's playable timeline (`editing_read_aligned_transcript` on the `edit_id` in `canonical.md`), never raw recording time, because cuts shift time.

| Clip | A 1 looks like | A 5 looks like |
|---|---|---|
| Stands alone | Needs the episode to make sense. Opens on "so, yeah" | A stranger gets the point with no setup |
| Opening | Throat-clearing, "great question", an intro | The first sentence is the claim or the tension |
| Ending | Trails off, or cuts mid-word | Ends on the landed point, cut on a word boundary |
| Specificity | Advice anyone could give | A concrete step, number or example the buyer can use |

| Title | A 1 looks like | A 5 looks like |
|---|---|---|
| Specific | "A great conversation about growth" | Names the concrete thing and the stake |
| Promise kept | The asset never delivers what the title claims | Paid off inside the asset |
| Front-loaded | The key words sit where the platform truncates | The key words come first |
| Voice | Hype, all caps, bait | Sounds like the host saying it |

| Caption or post copy | A 1 looks like | A 5 looks like |
|---|---|---|
| First line | Restates the title, or "In this episode" | Works alone before the fold |
| Specificity | Abstractions and adjectives | One concrete, sourced detail from the recording |
| Fit | Promises something the clip does not show | Sets up exactly what plays or follows |
| Ask | A generic CTA, or none where one was needed | One specific next step, or none by design |

| Thumbnail brief | A 1 looks like | A 5 looks like |
|---|---|---|
| Legibility | Small text on a busy frame | Readable at phone-feed size |
| Frame | Mid-blink, mouth half open, looking away | The expression matches the title's tension |
| Pairing | Repeats the title word for word | Adds what the title leaves out |

A thumbnail brief is not scored until it has text of four words or fewer, a frame timestamp on the canonical playable timeline and a composition note. Each missing piece is a fix. There is no image tool on this MCP, so the brief plus an ffmpeg frame grab is the asset.

Long-form written posts (newsletter, blog, LinkedIn text) are scored on the caption card. Show notes get three chapter times spot-checked on the canonical playable timeline. A case study must carry the situation, what they tried before, what changed, and the result in the customer's words. A missing part is a fix.

### 4. Run the tells pass on all prose
If the user gave a voice file, read it by path now. Its rules sit on top of this list and win where they conflict. Never copy it into this repo: skills point at knowledge files, and a copy drifts until the judge enforces a voice the user dropped. Every hit is a fix, quoted with its location.

| Tell | What it looks like | Fix |
|---|---|---|
| Hollow verbs | unlock, leverage, supercharge, seamless, robust, game-changer, delve, elevate, empower, streamline, harness | Name the action that happened |
| Reveal constructions | "It's not X, it's Y". "The real X is". "X didn't do Y, it revealed Z" | State the claim plainly |
| Rule-of-three padding | Three adjectives or three parallel clauses where one carries the point | Keep the one that carries it |
| Stacked em dashes | Two or more in a paragraph | A period, a comma, or a new sentence |
| Generic CTA | "Let me know in the comments", "Thoughts?" | A specific ask, or none |
| Hedge stacking | "Arguably", "in many ways", "to some extent" on one claim | One qualifier at most |
| Empty openers | "In today's...", "It's worth noting" | Delete the phrase |
| Closing recap | A last paragraph that restates the first | Stop at the last real point |

### 5. Return the verdict
One block per asset. Each fix names the exact line and quotes the text.

```
content-quality-gates: REVISE · cycle 1 of 3 · review: independent
asset: clip-03 for LinkedIn · built_from EDIT_ID@REVISION
gates: A pass · B FAIL · C pass · D pass · E n/a · F FAIL
score: not scored, gate failed
fixes:
1. [B] caption line 2: "we cut churn in half" is not in the transcript at 14:05. Quote it exactly or paraphrase without quote marks
2. [F] caption: 3,240 characters against a 3,000 cap. Cut 240
```

A package passes only when every piece passes. Hand the fix list to the skill that made each piece: `clip-selection` for clip points, `social-asset-pack` for captions and per-platform copy, `transcript-to-written` for long-form, `customer-interview-engine` for case studies, `podcast-show-notes-and-chapters` for show notes and chapters, `podcast-episode-pipeline` for episode titles and thumbnail briefs. PASS clears the asset for the user's sign-off. It is not that sign-off.

### 6. Re-judge
After the fixes, run the whole review again from step 2 as a new cycle, because a fix can break a quote. Cap at three cycles. After a third REVISE, stop and show the user all three fix lists.

## Riverside tools
| Step | Tool | Note |
|---|---|---|
| 2B | `search_recording_transcripts_exact` | Verbatim check. Last 120 days unless `createdAfterDate` is passed. Terms up to 256 characters |
| 2B, 2F | `platform_get_recording` | Recording date and `studioId`. Never set `includePreviewUrl`: it returns a studio-wide, non-revocable share token |
| 2B | `platform_get_transcript` | Recording time, one stamp per sentence. Not zero-padded: `1:5.000` is 1m05s |
| 2B, 3 | `editing_read_aligned_transcript` | Canonical-cut times for quotes, clip points, chapters and thumbnail frames |
| 2D | `editing_get_revision` | Is the declared canonical revision still current |
| 2E, 2F | `social_get_publishing_guidelines` | Caps, the fold, video constraints. Read it fresh each review |
| 2F | `social_get_connected_platforms` | Per-account limits |
| 2F | `editing_get_export_publish_data` | YouTube Content ID. A non-null `youtubeCode` stops the asset |

## Output
- Prints the verdict block for each asset, and one line for a package
- Appends every cycle's verdict, failures included, to `workspace/episodes/SLUG/assets/reviews.md`. The failed cycles show which making skill is weak
- Never edits the asset, `canonical.md`, `clearance.md` or `publish-log.md`, and never changes a row's state. Approval belongs to the user

## Rules & quality bar
- **Judge only.** No rewrites, no quick fixes. The fix list goes back to the skill that made the asset
- **Scripts decide clearance and staleness.** The model never overrides an exit code
- **A gate FAIL means no score.** A strong clip with an unverified quote is REVISE
- **Ask, never guess:** who speaks, whether it is sponsored, whether the transcript misheard
- **Label the review mode honestly.** Same-context when it was
- **Read-only on Riverside.** No publishing, no edits, no share links
- **Three cycles, then the user**

## Related skills
- Called by: every skill in this pack that makes an outbound asset
- Hands fixes to: the skill that made the asset (step 5)
- Publishing after PASS and sign-off: `distribution-and-scheduling`, the only skill that calls `social_upload_create`
- Consent requests: `customer-interview-engine`, `podcast-guest-ops`
- Stale assets: `podcast-recut-and-republish`
- Not for skill files: `skill-capture-loop`
