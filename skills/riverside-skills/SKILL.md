---
name: riverside-skills
description: >-
  Router and setup check for the Riverside skills. Use for "where do I start", "set up the
  riverside skills", "check my riverside connection", or a symptom instead of a task. Routes to
  one skill and runs no workflow itself.
---

# Riverside Skills

The front door. Checks the setup once, then reads what the user is trying to do and routes to
exactly one skill. It also owns the shared scripts and templates every other skill uses.

## When to use
- First session after install
- The user describes an outcome or a problem rather than naming a skill
- Two skills could plausibly apply
- Somebody wants a tour

## Inputs
- A connected Riverside MCP (Grow, Webinar or Business plan)
- Nothing else. This skill finds out what is missing

## Workflow

### First run: the setup check

Run all four. Report each as OK or MISSING, with the fix.

1. **MCP connected.** Call `platform_list_recordings` with `limit: 5` and show title and date for
   each. If no Riverside tool is available, STOP: the MCP is not connected or the sign-in did not
   finish. Tell the user to run
   `claude mcp add --transport http --scope user riverside https://mcp.riverside.com/mcp`, then
   `/mcp`, choose riverside, and complete the browser sign-in. Build nothing on an unverified
   connection.
2. **Approval gate present.** Read `.claude/settings.json` in the current folder and
   `~/.claude/settings.json`. The gate is present when `permissions.ask` lists
   `social_upload_create` under the prefix this session actually uses (`mcp__riverside__` or
   `mcp__claude_ai_Riverside__`). If it is missing, show the user the `ask` block from this repo's
   `.claude/settings.json` and offer to merge it into `~/.claude/settings.json`. Merge only on an
   explicit yes, preserve every existing key, and re-read the file afterwards to confirm it parses.
   If the user declines, say plainly that publish calls will then rely on the skills alone.
3. **Workspace.** Confirm `./workspace/episodes/` exists (or `RIVERSIDE_WORKSPACE` is set). If
   not, offer to create it.
4. **ffmpeg** (podcast users only). Run `command -v ffmpeg`. Missing is fine unless they plan to
   prep audio for a podcast host or grab thumbnail frames locally.

### Routing

Ask one thing: **what recording do you have, and what should it become?** Route on the answer.

| The user has | And wants | Route to |
|---|---|---|
| Discovery or win/loss calls | Messaging, positioning, headline tests | `research-call-mining` |
| A customer interview | A case study or testimonial | `customer-interview-engine` |
| Sales calls with objections | Rep training, clips for prospects | `sales-enablement-clips` |
| A podcast episode | The whole episode shipped | `podcast-episode-pipeline` |
| A live episode that is wrong | To fix it everywhere | `podcast-recut-and-republish` |
| An episode | Chapters, titles, show notes only | `podcast-show-notes-and-chapters` |
| A finished episode file | It on the podcast host | `podcast-host-and-rss` |
| A guest coming up | Booking, prep, release, their asset pack | `podcast-guest-ops` |
| Any recording | The moments worth clipping | `clip-selection` |
| Approved clips | Captions, titles, hooks per platform | `social-asset-pack` |
| Any recording | A newsletter, blog or LinkedIn text post | `transcript-to-written` |
| Approved assets | Scheduled or published | `distribution-and-scheduling` |
| A webinar | Run-of-show, then on-demand and clips | `webinar-production` |
| A draft of anything | A check before it goes out | `content-quality-gates` |
| A task done three times | A skill | `skill-capture-loop` |

If the user has no recordings in Riverside yet, the first move is getting some in: record, or
upload existing calls in the Riverside app. Then `research-call-mining`, because it pays off
fastest and needs no customer permission.

### Starting order for a new team

1. Setup check, above
2. `research-call-mining` on ten recent discovery calls. Internal only, so nothing to clear
3. `customer-interview-engine` on one customer who has already agreed in writing
4. `clip-selection` on that interview, then `social-asset-pack`, then `content-quality-gates`
5. `distribution-and-scheduling` for the first post
6. The podcast skills, if there is a podcast

## Shared scripts

Every skill calls these by path relative to its own folder, as `../riverside-skills/scripts/NAME`.
They decide. The model reports what they say.

| Script | Job | Exit codes |
|---|---|---|
| `new-recording SLUG` | Creates `workspace/episodes/SLUG/` from the templates | 0 created · 1 exists · 64 usage |
| `clearance-check SLUG internal\|public` | Reads `clearance.md` and decides | 0 cleared · 2 no file · 3 not cleared |
| `stale-check SLUG` | Lists logged assets built from an old cut | 0 none stale · 1 stale found · 2 no canonical |
| `skill-landed SKILL_DIR "PHRASE"` | Proves a correction reached a skill file and was committed | 0 landed · 1 missing · 2 uncommitted · 3 not in git |

Templates live in `templates/`: `canonical.md`, `publish-log.md`, `clearance.md`.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| Connection check | `platform_list_recordings` | `limit: 5` is enough |
| Finding IDs | `platform_list_productions` → `platform_list_studios` → `platform_get_project` | `get_project` returns recordings and edits in one call |

Full catalog and the gaps: `docs/riverside-mcp.md`.

## Output
- The setup report: four lines, OK or MISSING, each with its fix
- One route, with one sentence on why, and what comes after it

## Rules & quality bar
- **Verify before building.** No route until the connection check returns recordings
- **One skill at a time.** Loading several at once bloats context and produces mush
- **Route on the recording, not the request.** Somebody asking for "more video" usually has
  recorded calls that answer the question faster
- **Never edit user settings without a yes**, and never remove an existing key

## Related skills
- Every skill in this repo
- `docs/riverside-mcp.md` for the tool catalog, `docs/canonical-cut.md` for the method
