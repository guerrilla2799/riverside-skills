---
name: social-asset-pack
description: >-
  Use for "write the captions", "social copy for these clips", "titles and thumbnails", "copy for each platform". Writes hook, caption, title and thumbnail brief per platform. Never posts: that is distribution-and-scheduling.
---

# Social Asset Pack

Approved clips in, per-platform copy out. For each clip: one hook, then the title, caption and thumbnail brief each platform needs, written to that platform's live caps and filed under the field names `social_upload_create` takes, so `distribution-and-scheduling` can post it without rewriting anything. This skill never publishes.

## When to use
- `clip-selection` has built approved clips and they need copy
- A clip has one caption pasted across every platform and it fits none of them
- Before `distribution-and-scheduling`, which only schedules packs marked `approved`
- Not for long-form writing with no clip attached (`transcript-to-written`), and not for scoring or cutting clips (`clip-selection`)

## Inputs
- `publish-log.md` rows for the clips, with each clip's `editId` in `reference` (written by `clip-selection`)
- `assets/clip-scorecard.md` for each clip's timestamps, speaker roles and number flags
- From the user: target platforms, whether any of it is sponsored and by whom, the link and call to action if there is one, and the path to their voice or style file if they have one
- Script paths are relative to this skill's base directory. The workspace is `./workspace` unless `RIVERSIDE_WORKSPACE` is set

## Workflow

1. **Resolve the clips.** Match every clip the user names to a `publish-log.md` row carrying an `editId`. A post announcing the full episode uses the canonical `edit_id` from `canonical.md` instead. A clip with no row: STOP. It was not scored and built by `clip-selection`, so there is no logged edit to write for. Route to `clip-selection`.
2. **Check staleness.** Run `../riverside-skills/scripts/stale-check SLUG`.
   - Exit 2: STOP. `canonical.md` or `publish-log.md` is missing, or no canonical cut is declared.
   - Exit 1: if any clip in this pack is listed as STALE, STOP. It was cut from an older revision of the canonical edit. Route to `podcast-recut-and-republish` or re-run `clip-selection`. Stale rows for other assets are reported and do not block.
3. **Clear before drafting.** If any clip features a customer or guest (anyone but the user), run `../riverside-skills/scripts/clearance-check SLUG public` before writing a word.
   - Non-zero exit: STOP. Report the script's stderr verbatim and offer to draft the consent request instead (`customer-interview-engine` for customers, `podcast-guest-ops` for guests).
   - Exit 0: print the `Restrictions:` line and apply it to every field, thumbnail text and hashtags included. "First name only" means no surname anywhere, @tags included.
4. **Read each clip's own transcript.** `editing_read_aligned_transcript` on the clip's edit. Record the `revision` it returns in the pack header: `social_upload_create` posts an edit as it stands, so `distribution-and-scheduling` checks that the clip has not changed since this copy was written. Copy may quote only words said inside the clip, verbatim, with the timestamp on that clip's timeline. A number a customer or guest said aloud keeps its `confirm with customer` flag, and a pack that uses it cannot be approved until the number is confirmed in writing.
5. **Read the live caps.** `social_get_publishing_guidelines` for the target platforms. Then `platform_get_recording` for the `studioId`, and `social_get_connected_platforms` for which accounts exist and the X account's `characterLimit`. With no X account connected, draft to 280 and mark the limit unverified. Never write to a cap from memory.
6. **Ask about sponsorship.** Ask whether any clip is sponsored, and do not assume the answer. Sponsored copy opens every caption with a plain disclosure (`Paid partnership with SPONSOR.` or `Sponsored by SPONSOR.`). The pack notes `tiktokData.isBrandedContent: true` so `distribution-and-scheduling` sets it.
7. **Load the voice.** If the user named a voice or style file, read it where it lives at draft time. Never paste a copy into this skill or the workspace. Skills point at knowledge files and do not carry copies. With no file, write plain and direct.
8. **Write the hook, then each platform.** One hook per clip, taken from the clip's own words. Then each target platform:

   | Platform | Field in `social_upload_create` | Write to |
   |---|---|---|
   | YouTube | `youtubeData.title`, `youtubeData.description` | A searchable title. About the first 150 characters of the description show before "Show more" |
   | YouTube Shorts | Same fields, with `youtubePlatform: YOUTUBE_SHORTS` | A short title (the guidelines suggest 20–40 characters) and a short description |
   | TikTok | `tiktokData.title` | TikTok shows it as the caption. Hashtags are how it gets found |
   | Instagram | `instagramData.caption` | Short, ending on the call to action. Hashtag and @tag caps apply |
   | Facebook | `facebookData.description` | The Reel's caption |
   | LinkedIn | `linkedinData.description` | The hook inside the first ~140 characters, because that is all that shows before "see more". Posts go to the member's personal profile, so write in the first person |
   | X | `xData.text` | Inside the account's `characterLimit`, counting every URL as 23 characters |

   With a sponsor, the disclosure comes first, and on LinkedIn the disclosure and the hook both fit inside the first ~140 characters.
9. **Write the thumbnail brief.** There is no image-generation tool on this MCP, so a thumbnail here is a brief:
   - Text of four words or fewer
   - The frame timestamp to grab, on the clip's timeline
   - Composition: who is in frame, where the text sits, what stays clear of platform buttons
   - If the export is already downloaded, the base frame: `ffmpeg -ss M:SS -i EXPORT.mp4 -frames:v 1 workspace/episodes/SLUG/assets/thumb-clip-NN.png`

   The same timestamp fills `instagramData.thumbnailOffset` (seconds) and `tiktokData.videoCoverTimestampMs` (milliseconds). `social_upload_create` has no YouTube thumbnail field, so a custom YouTube thumbnail is set in YouTube Studio. Say so in the brief.
10. **Count every field in code.** Print each field's character count beside its cap. For X, count every URL as 23 and CJK characters and surrogate-pair emoji as 2. The guidelines name `twitter-text` (`parseTweet(text).weightedLength`) as the exact counter, so use it when it is installed. Anything over a cap is rewritten, never cut off mid-sentence.
11. **Write the file and log it.** One file per clip: `workspace/episodes/SLUG/assets/social-pack-clip-NN.md`, in the shape under Output. Add a `publish-log.md` row per pack: date, `social-pack clip-NN`, `built_from` copied from the clip's row, destination `content-quality-gates`, reference = the file path, state `draft`.
12. **Gate, then hand off.** Hand each pack to `content-quality-gates`. It judges and logs its verdict to `assets/reviews.md`. This skill revises only what the verdict names, then the judge runs again.
    - When a pack passes and the user signs off on its copy, set its row to `approved`. The judge never changes a row's state.
    - A pack with an open `confirm with customer` flag stays `draft`.
    - Only `approved` packs go to `distribution-and-scheduling`. This skill never calls `social_upload_create`, `social_update_upload` or `social_cancel_upload`.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 4 | `editing_read_aligned_transcript` | On the clip's own edit, so quotes and timestamps match what plays |
| 5 | `social_get_publishing_guidelines` | Live caps for every metadata field and every platform |
| 5 | `platform_get_recording` → `social_get_connected_platforms` | `studioId`, connected accounts, X `characterLimit`. Never pass `includePreviewUrl` |

## Output
- Writes: `workspace/episodes/SLUG/assets/social-pack-clip-NN.md` per clip, optional base frames at `assets/thumb-clip-NN.png`, and one `publish-log.md` row per pack
- Shows: each pack in full, the counts, open flags, and which platforms were skipped and why (no connected account, clip too long, wrong aspect ratio)

Each pack file:

```
# clip-NN: SHORT-TITLE    edit CLIP_EDIT_ID@CLIP_REVISION    built_from CANONICAL_EDIT_ID@REVISION
Source lines: "VERBATIM LINE" (SPEAKER ROLE, M:SS on the clip)
Restrictions applied: RESTRICTIONS OR none
Flags: confirm with customer (NUMBER) · X limit unverified
Hook: HOOK

## LinkedIn   linkedinData.description   NNN / 3000   (first 140: HOOK FITS)
POST TEXT

## X   xData.text   NNN / CHARACTER_LIMIT   (URLs counted at 23)
POST TEXT

Thumbnail: TEXT (4 words max) · frame M:SS · COMPOSITION · YouTube: set in Studio
```

## Rules & quality bar
- **Clearance runs before drafting.** A non-zero `clearance-check` exit stops the pack, and restrictions reach every field
- **Quote only what the clip says,** verbatim, with its timestamp. No invented customer lines, numbers or outcomes
- **Caps come from the live guidelines,** and the X budget comes from the account's `characterLimit`
- **The LinkedIn hook lives in the first ~140 characters,** after any sponsor disclosure
- **Sponsored copy discloses on the first line,** in plain words
- **This skill never publishes.** `distribution-and-scheduling` is the only path to a live post
- **Nothing is approved until `content-quality-gates` passes it** and the user signs off

## Related skills
- Requires: `clip-selection` for built, logged clips, and `riverside-skills` for the scripts
- Feeds: `content-quality-gates`, then `distribution-and-scheduling`
- Pairs with: `transcript-to-written` when a clip also needs a long-form companion piece
- See also: `docs/riverside-mcp.md`
