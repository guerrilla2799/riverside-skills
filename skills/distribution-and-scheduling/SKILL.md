---
name: distribution-and-scheduling
description: >-
  Use for "schedule these posts", "posting calendar", "publish this clip", "reschedule", "cancel that post", "how did it do". The only skill that calls social_upload_create, one explicit yes per post. Copy is social-asset-pack.
---

# Distribution and Scheduling

The publish calendar and the only path to a live post. Builds the schedule across YouTube, YouTube Shorts, TikTok, Instagram, Facebook, LinkedIn and X from approved packs, shows every post in full, and calls `social_upload_create` only after an explicit yes for that post. Also reschedules, edits and cancels posts that have not gone out, and reads their numbers afterward. "Nothing goes live until I sign off" is the default for every run, whether or not the user says it.

## When to use
- Approved clips and packs are ready to go out
- Someone wants a posting plan before anything is scheduled
- A scheduled post needs a new time, new copy, or cancelling
- Someone asks how a post did
- Not for writing or fixing copy (`social-asset-pack`) or long-form text (`transcript-to-written`)

## Inputs
- `publish-log.md` rows in state `approved` for each clip and its pack. The clip's `editId` is its row's `reference`
- `assets/social-pack-clip-NN.md` from `social-asset-pack`
- From the user: their timezone, the slots or cadence they want, and the privacy for each YouTube post
- Script paths are relative to this skill's base directory. The workspace is `./workspace` unless `RIVERSIDE_WORKSPACE` is set

## Workflow

The repo's `.claude/settings.json` puts `social_upload_create`, `social_update_upload`, `social_cancel_upload` and `social_disconnect_social_account` behind a Claude Code `ask` rule, so Claude Code prompts before each call. It applies only when Claude Code runs from inside the repo folder; if the skills were copied into `~/.claude/skills/`, the `riverside-skills` router can add it to user settings. Keep permission prompts on. This skill does not rely on the rule: the in-chat yes in step 9 comes first, every time.

### Schedule

1. **Check staleness.** Run `../riverside-skills/scripts/stale-check SLUG`. Exit 2: STOP, `canonical.md` or `publish-log.md` is missing or no canonical cut is declared. Exit 1: any asset in this batch that is listed as STALE is dropped, and the user is told it was built from an older cut (route to `podcast-recut-and-republish`). If every asset in the batch is stale, STOP.
2. **Check approval.** Every clip and pack in the batch needs a `publish-log.md` row in state `approved`. A `draft` row: STOP for that asset and route to `content-quality-gates`. A pack with an open `confirm with customer` flag is not approved, whatever its row says.
3. **Find the studio.** `platform_list_studios`; ask which one if there are several. `platform_get_studio` gives its production, which `social_list_uploads` needs.
4. **Read the accounts.** `social_get_connected_platforms`. Note each account's human-readable name, `platformAccountId`, YouTube `channelId`, X `characterLimit`, TikTok `privacyLevelOptions`, `postingAvailable` and `maxVideoPostDurationSec`. An empty `accounts` list does not always mean nothing is connected; ask before concluding it. Accounts are connected in the Riverside dashboard, which is what Riverside's help center describes. `social_connect_social_account` exists and returns a sign-in link; use it only when the user asks. A platform with no account comes off the calendar, and the user is told.
5. **Read the rules.** `social_get_publishing_guidelines`. Check each clip against each platform's duration, size and aspect-ratio limits, and each field against its cap. When this was written, Facebook took Reels only (vertical), Shorts and TikTok took vertical or square, and LinkedIn posted to the member's personal profile, with company pages unsupported. Re-read the guidelines every run. A clip that does not fit a platform comes off that platform; it is not trimmed here.
6. **Draft the calendar.** Ask for the timezone. Write each slot as local date and time, IANA zone, and the UTC offset for that date (daylight saving moves it), plus the exact `scheduledAt` string in ISO 8601 with that offset. Slots come from the user, or from their own past posts (`social_list_uploads` with a past `from` and `to` and `includeViews`), never from generic best-time claims. Items the MCP cannot post, such as a text-only LinkedIn post, a newsletter or a blog post, go on the calendar marked `manual`. Write `assets/schedule.md` and show it. No publish tool has been called yet.
7. **Run the Content ID check.** Before any YouTube or Shorts post, call `editing_get_export_publish_data` on the clip's edit. A non-null `youtubeCode` means a likely copyright match: STOP for that post, show the code, and keep it off YouTube.
8. **Show each final summary.** Per post, numbered: platform; account or channel name, never an internal ID; the exact title and post text; privacy or visibility; and immediate, or the scheduled date, time and timezone. YouTube `privacyStatus` is asked for and shown, never defaulted. TikTok `privacyLevel` is one of that account's `privacyLevelOptions`. LinkedIn `visibility` is `PUBLIC` or `CONNECTIONS`, stated. Sponsored posts keep the disclosure on the first line and set TikTok `isBrandedContent: true`. A fresh clip edit has no export, so show the `composeSettings` defaults (1080p; watermark, transcription, gain normalization and background-noise removal all off) and ask whether to change any. TikTok rejects watermarked video. State plainly that an immediate publish cannot be undone through the MCP, and that once a post is live nothing here can edit, unpublish or delete it.
9. **Get an explicit yes per post.** The user answers by number ("yes 1, 3, 4"). Silence, approval of the calendar, or a yes given to an earlier version is not a yes. Any change to text, account, privacy or time re-opens step 8 for that post. No yes: STOP for that post.
10. **Create the post.** `social_upload_create` with `studioId`, `platformAccountId`, `platform`, `clipId` set to the clip's `editId` (edit IDs are accepted unchanged), the platform's data object from the pack, `composeSettings` as confirmed, and `scheduledAt`. Leaving out `scheduledAt` publishes immediately, so it is left out only when the user chose immediate in step 9. Shorts is `platform: YouTube` with `youtubePlatform: YOUTUBE_SHORTS`. Post the asset the user named and no other: if its ID is rejected, STOP and report, and never substitute a close match. On errors:
    - `VALIDATION_ERROR`: fix the named field and retry; if it says the clip is not exported, retry with the confirmed `composeSettings`
    - `SERVICE_UNAVAILABLE`, `INTERNAL_SERVER_ERROR`, a timeout or a network error: the outcome is unknown. Do not retry. Check `social_list_uploads` for the window first, or the post can go out twice
    - `INVALID_TOKEN`, `UNAUTHORIZED` or an account-level `FORBIDDEN`: the user reconnects the account in Riverside's web UI
    - `CONFLICT`: do not repeat the call; report it. `RATE_LIMIT_EXCEEDED`: tell the user to try again in a few minutes
11. **Check the outcome.** `social_get_upload_status` once, shortly after. Only `COMPLETED` is live, whatever `externalId` shows. `SCHEDULED`: stop, report `scheduledAt`, and check again at or after that time or when the user asks. `PENDING`: still rendering or uploading, so check again later. A render takes minutes and a scheduled post can be days out, so no polling loop. `FAILED`: relay `reason` as written and act on `reasonCode`. A `PLATFORM_CONNECTION_EXPIRED` failure with a `scheduledAt` can recover once the account is reconnected.
12. **Log it.** One `publish-log.md` row per post: date, `clip-NN → PLATFORM`, `built_from` copied from the clip's row, destination (platform and account name), reference = `uploadId`, state `scheduled` on `SCHEDULED` and `published` only on `COMPLETED`. A failed post keeps state `approved` with its `uploadId`, so it can be fixed with `social_update_upload`.

### Change or cancel

1. **Find the post.** The `uploadId` from `publish-log.md`, or `social_list_uploads`. With no dates it shows only the next 30 days, so pass a past `from` and `to` for older posts.
2. **Read it.** `social_get_upload`. Read `editable`, `mediaEditable` and `cancellable` separately; they do not move together.
3. **Reschedule or edit.** If `editable` is false, STOP: the post is out or mid-publish, and changes happen on the platform itself. Otherwise run `stale-check SLUG` as in Schedule step 1, show the current values and the exact change, and get an explicit yes. Then `social_update_upload` with only the changed fields, inside the data object for the post's own platform. A new `scheduledAt` must be in the future. `publishNow` cannot be undone and cannot be combined with `scheduledAt`. X `children` and LinkedIn `comments` replace the whole list: send back everything `social_get_upload` returned plus the change, or the rest is deleted. `social_update_upload` cannot swap the video; a new cut means cancel, then create again from step 1 of Schedule. Show the post it returns and update the log row.
4. **Cancel.** If `cancellable` is false, STOP: tell the user the post has published or is publishing, and removal happens on the platform. If true, show the platform, account name, caption or title, and scheduled time, and get an explicit yes. `social_cancel_upload` is irreversible and removes the post from Riverside's Content Planner. Its own response is the confirmation: afterwards `social_get_upload` returns the same error an unknown ID does, so do not read it back. Set the row to `retired`.

### Results

`social_get_upload_analytics` with `uploadId` and `studioId`. Always quote `asOf`: the numbers are collected on a schedule, so they lag. `hasData: false` means nothing is collected yet; say so instead of reporting zeros. A null metric is not zero. `frozen: true` means final; X stops updating 5 days after publishing. A `daily` series is already per day, so report it as given. `MISSING_ANALYTICS_SCOPE` means the account needs reconnecting in Riverside before any numbers can exist. Write the read to `assets/analytics-YYYY-MM-DD.md`.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| Schedule 3 | `platform_list_studios` · `platform_get_studio` | `studioId`, and the production for `social_list_uploads` |
| Schedule 4 | `social_get_connected_platforms` | Account names and IDs, `channelId`, `characterLimit`, `privacyLevelOptions` |
| Schedule 4 | `social_connect_social_account` | Only on request; accounts are normally connected in the Riverside dashboard |
| Schedule 5 | `social_get_publishing_guidelines` | Caps, video limits, error recovery |
| Schedule 7 | `editing_get_export_publish_data` | Non-null `youtubeCode` stops the YouTube post |
| Schedule 10 | `social_upload_create` | Behind the `ask` rule. No `scheduledAt` means immediate. YouTube `privacyStatus` explicit |
| Schedule 11 | `social_get_upload_status` | Only `COMPLETED` is live |
| Schedule 6, Change 1 | `social_list_uploads` | Past windows need explicit `from` and `to` |
| Change 2 | `social_get_upload` | `editable`, `mediaEditable`, `cancellable` |
| Change 3 | `social_update_upload` | Behind the `ask` rule. Changed fields only |
| Change 4 | `social_cancel_upload` | Behind the `ask` rule. Irreversible |
| Results | `social_get_upload_analytics` | Quote `asOf`; X freezes after 5 days |

## Output
- Writes: `workspace/episodes/SLUG/assets/schedule.md` (every slot with local time, zone, offset, `scheduledAt`, platform, account name, asset, privacy, and `manual` rows), one `publish-log.md` row per post, `assets/analytics-YYYY-MM-DD.md` for results
- Shows: the calendar, each numbered final summary, then per post the `uploadId` and its status (`SCHEDULED` with time, `COMPLETED`, or `FAILED` with Riverside's reason)

## Rules & quality bar
- **The only skill that calls `social_upload_create`.** Other skills hand off here
- **An explicit yes per post,** after a final summary showing platform, account name, exact text, privacy and timing with timezone
- **Scheduled by default.** Immediate only when the user chose it, after being told it cannot be undone here
- **YouTube privacy is always explicit,** and a non-null `youtubeCode` stops a YouTube post
- **Only `COMPLETED` counts as live.** `social_upload_create` returning is not proof
- **Never retry an ambiguous failure** before checking whether the post already exists
- **Stale or unapproved assets never go on the calendar**

## Related skills
- Requires: `social-asset-pack` for approved copy; `clip-selection` for the clips; `content-quality-gates` for the pass
- Pairs with: `podcast-episode-pipeline` for full episodes; `podcast-recut-and-republish` when a cut changes after scheduling
- See also: `docs/riverside-mcp.md` for the approval gate and the publish tool table
