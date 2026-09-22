# The Riverside MCP: what it does, what it doesn't, and what bites

Every skill in this repo names real Riverside tools. This page is the reference they point at.
It was written against the live MCP on 2026-09-21, and against Riverside's own
[MCP page](https://riverside.com/mcp) and
[setup article](https://support.riverside.com/hc/en-us/articles/37803607978141-Connect-to-Riverside-MCP)
on the same day. If a tool behaves differently from what is written here, trust the tool and
open an issue.

---

## Before anything else

- You need a Grow, Webinar, or Business plan. The MCP is included at no extra cost on those plans.
- Your role must be account owner, admin, director, or editor.
- Connect in Claude Code:

  ```bash
  claude mcp add --transport http --scope user riverside https://mcp.riverside.com/mcp
  ```

  Then run `/mcp` inside Claude Code and sign in. `--scope user` makes Riverside available in
  every folder. Leave it off and it only works in the folder where you ran the command.
- Using the MCP doesn't spend your Riverside AI credits.

### Tool names depend on how you connected

The tools below are listed by their base name, such as `social_upload_create`. Claude Code sees
each one with a prefix:

| How you connected | Full tool name |
|---|---|
| `claude mcp add ... riverside ...` | `mcp__riverside__social_upload_create` |
| The Riverside connector on claude.ai | `mcp__claude_ai_Riverside__social_upload_create` |

The approval gate in `.claude/settings.json` lists both forms. If you named the server
something other than `riverside`, edit those entries to match or the gate won't fire.

---

## How Riverside is organized

```
production            one per account, usually
└── studio            a show, a team, a space
    └── project       one recording session and everything made from it
        ├── recording the raw take (or an upload). Has a transcript.
        └── edit      an editable timeline built from a recording. Has revisions.
```

Most tools need IDs from the level above. The cheap way down is `platform_get_project`, which
returns a project's recordings and edits in one call.

Every write tool takes an `expectedRevision` and rejects a stale one, because edits are
revisioned. So `edit_id@revision` identifies one
exact version of a cut, which is what this repo's `canonical.md` records and what
`publish-log.md` stamps on every asset.

Edits never overwrite the recording. `editing_create_edit_from_recording` and
`editing_clone_edit` make a new timeline. The original stays intact.

---

## Tool families

### Find things

| Tool | What it does | Watch for |
|---|---|---|
| `platform_list_productions` · `platform_get_production` | Top of the tree | Start here if you have no IDs |
| `platform_list_studios` · `platform_get_studio` | Studios and their production | `studioId` is required by every social and brand tool |
| `platform_list_projects` · `platform_get_project` | Projects, with recordings and edits | `get_project` is the efficient fan-out |
| `platform_list_recordings` · `platform_get_recording` | Recordings, newest first | `includePreviewUrl` returns a studio-wide, non-revocable share token. Never request it unless the owner asked for a share link |
| `platform_list_edits` · `platform_get_edit` | Edits | Results can include soft-deleted edits. Filter `deleted: true` out |
| `platform_get_transcript` | The transcript as prose, one `Speaker (m:s.mmm)` stamp per sentence | Minutes and seconds are not zero-padded: `1:5.000` is 1m05s. If a recording exposes no offset for a speaker, that speaker is timed from their own track start, so their times can't be compared with anyone else's |
| `search_riverside` | Fuzzy search over projects, clips and recording transcripts | 10 results a page. Pass explicit `null` for unused parameters. For spoken moments, filter with `{"entity": "TAKE", "field": null, "value": null}` |
| `search_recording_transcripts_exact` | Exact-phrase search across recordings | **Defaults to the last 120 days.** Pass `createdAfterDate` to reach further back |

### Edit

| Tool | What it does |
|---|---|
| `editing_create_edit_from_recording` | New editable timeline from a recording. Can return `FAILED_PRECONDITION` on some accounts. Retrying will not help, so use an existing edit from `platform_list_edits` |
| `editing_clone_edit` | Copy an edit into a new one. Use it to branch a version instead of editing the canonical cut in place |
| `editing_read_aligned_transcript` | Transcript on the edit's playable timeline. `detail: "words"` gives word-level timing and the IDs the cut tools need |
| `editing_resolve_transcript_selection` → `editing_cut_time_ranges` | Turn a quote into exact cut points, then cut. Only run the payload when `readyToApply` is true. Only `intent: "remove"` works today: `keep` and `move` are rejected, and so are `reorder_timeline` and `create_edit_from_segments`. To make a clip, clone the edit and cut away the head and the tail |
| `editing_remove_fillers` · `editing_remove_pauses` · `editing_apply_smart_mutes` | Audio cleanup. `editing_restore_audio_cleanup` undoes any of the three |
| `editing_set_magic_audio` | Riverside's audio enhancement, per track |
| `editing_apply_smart_layout` | Speaker-driven layouts: Smart, FullScreen, PictureInPicture, SplitScreen, Grid |
| `editing_get_captions_presets` → `editing_set_captions` | Captions. Pick an existing preset rather than inventing a style |
| `editing_get_brand` · `editing_apply_brand` · `editing_set_brand` | Brand kit. `set_brand` changes the studio's kit for everyone, so it is behind the approval gate |
| `editing_add_lower_third` · `editing_add_text_overlay` | Name and title cards, text |
| `editing_insert_media_as_scene` | A full-screen scene that lengthens the timeline, such as a closing card |
| `editing_insert_overlay` · `editing_insert_audio` | Layered visuals, music, sound |
| `editing_get_stock_media` · `editing_insert_stock_media` · `editing_get_stock_music` | Pexels stock and the free music library |
| `editing_batch` · `editing_validate_edit_plan` · `editing_apply_verified_edit_plan` | Many operations at once, including chapters. Read `editing_get_editing_guide` first |
| `editing_compare_revisions` | What changed between two revisions. This is how a re-cut gets summarized |
| `editing_get_revision` | Cheap check that a revision you hold is still current |
| `editing_get_export_publish_data` | Content ID flags before a YouTube publish. A non-null `youtubeCode` means a likely copyright match |

### Export

| Tool | What it does | Watch for |
|---|---|---|
| `exports_create_export` | Renders an edit to video (up to 2160p) or audio (`MP3`, `WAV`) | Calling it twice renders twice. Pass `rstlRevisionId` to render one exact revision, the canonical one. Behind the approval gate |
| `exports_get_export` | Status of a render | Returns an S3 key, **not a download link.** The file lands in your Riverside exports folder, and you download it from there |

### Upload your own files

`media_create_media_upload` → upload the bytes with `curl -T` → `media_finalize_media_upload`
→ poll `media_get_media`. Files up to 500 MB: MP4, WEBM, MPEG, WAV, MP3, OGG, JPG, PNG.

These land in the editor's **Your Media** library, ready to place on a timeline. Use them for
closing cards, intro bumpers, B-roll and logos. Going by the tool's own documentation, this isn't
how you get a call transcribed. To bring in a Zoom or Gong recording as a searchable
recording, upload it in the Riverside app.

### Publish

| Tool | What it does | Watch for |
|---|---|---|
| `social_get_connected_platforms` | Accounts on a studio, with IDs and per-account limits | An empty `accounts` list does not always mean nothing is connected |
| `social_get_publishing_guidelines` | Per-platform caps, video limits, error recovery | Read it before composing any post |
| `social_upload_create` | Publishes or schedules a **video** to YouTube, YouTube Shorts, TikTok, Instagram, Facebook, LinkedIn or X | It needs a `clipId` (a clip or an edit), so text-only posts are out. LinkedIn posts go to the member's personal profile, and company Pages aren't supported yet. Facebook takes Reels only: vertical, 3 to 90 seconds. TikTok takes vertical or square, and rejects watermarked video. Shorts is `platform: YouTube` with `youtubePlatform: YOUTUBE_SHORTS`, capped at 180 seconds, 9:16 or 1:1. Omit `scheduledAt` and it publishes **immediately**. YouTube needs an explicit `privacyStatus`. An unexported edit publishes with `composeSettings`, which renders it first |
| `social_get_upload_status` | The only proof a post went out | Live means `COMPLETED`. Check once shortly after publishing, stop at `SCHEDULED`, and check again when the user asks. Riverside's own guidance: no polling loop |
| `social_list_uploads` · `social_get_upload` | Find and read posts | With no dates, `list_uploads` shows the next 30 days only |
| `social_update_upload` | Reschedule or edit a post that has not published | `publishNow` cannot be undone. It cannot swap the video, even when `social_get_upload` reports `mediaEditable`: through the MCP, a new cut means cancel and re-create |
| `social_cancel_upload` | Stop a post before publishing starts | Irreversible, and it only works while `cancellable` is true |
| `social_get_upload_analytics` | Views, likes, comments, shares | Collected on a schedule, so always quote `asOf`. X stops updating after 5 days |
| `social_connect_social_account` · `social_disconnect_social_account` | Connect or remove an account | Disconnecting one Facebook Page disconnects every Page under that login and cancels their scheduled posts |

`social_upload_create` takes no revision. It posts the edit as it stands at the moment of the call, so check `editing_get_revision` against `canonical.md` right before publishing.

The approval gate exists because nothing on this MCP can edit, unpublish or delete a post once
it's published.

---

## What the MCP doesn't cover

The skills route around each of these, and the last column names the skill that handles it.

| Not covered | What to do instead | Skill that handles it |
|---|---|---|
| No download link for a rendered file | Download from your Riverside exports folder | `podcast-host-and-rss` |
| Your podcast host (Transistor, Buzzsprout, Captivate, or Riverside's own hosting) | Upload through the host, by hand or through its API | `podcast-host-and-rss` |
| Recordings that live in Gong, Zoom, Chorus or a dialer | Upload them in the Riverside app, or export transcripts into `workspace/` | `research-call-mining` |
| Scheduling a recording session | Book it in Riverside | `podcast-guest-ops` |
| Files that never went into Riverside | Edit locally (ffmpeg), or upload them to Your Media first | `podcast-episode-pipeline` |
| Text-only posts | Post natively on the platform. The MCP publishes video | `distribution-and-scheduling` |
| LinkedIn company Pages | Post natively. The MCP posts to personal profiles only | `distribution-and-scheduling` |
| Billing and account settings | Not exposed, by design | n/a |

---

## The approval gate

`.claude/settings.json` puts these tools behind an `ask` rule, so Claude Code stops and asks you
before each call no matter what the skill says:

- `social_upload_create`, `social_update_upload`, `social_cancel_upload`,
  `social_disconnect_social_account`: anything that posts, changes a post, or removes one
- `editing_set_brand`: changes the brand kit for the whole studio
- `exports_create_export`: starts a render

Tested 2026-09-21: an `ask` rule held even when the same tool was explicitly allowed. It wasn't
tested with permission prompts turned off, so keep prompts on when you publish.

The file only applies when you run Claude Code from inside this folder. If you copied the skills
into `~/.claude/skills/`, ask the `riverside-skills` router to add the gate to your user settings,
or copy the `ask` list into `~/.claude/settings.json` yourself.
