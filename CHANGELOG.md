# Changelog

## 1.1.0 – 2026-10-04

- New skill `webinar-archive-mining`: searches old webinars for the segments that answer an angle
  you supply, then routes clips and one blog post per webinar. It passes `createdAfterDate` on
  every search, because `search_recording_transcripts_exact` defaults to the last 120 days and
  would otherwise skip the entire archive it exists to read
- `webinar-production` now also produces the social copy for its clips (`social-asset-pack`) and a
  blog post off the canonical edit (`transcript-to-written`)
- `customer-interview-engine` reordered: the case study, quotes and clip shortlist are drafted
  first and the skill's last output is the approval email carrying that exact material. Clearance
  is now enforced at the publish boundary instead of before drafting, so nothing reaches a
  platform without the saved written reply
- Fixed a chain break the reorder above introduced. `customer-interview-engine` had lost its
  pre-draft gate, but `clip-selection` and `social-asset-pack` still stopped on a clearance
  check, and a fresh recording folder starts at `status: pending`, which fails the internal
  check as well as the public one. Drafting a case study therefore could not reach the clip or
  copy steps at all. `customer-interview-engine` now opens the folder at `internal-only` after
  the interview, which is true to what the interviewee agreed to, and the written reply upgrades
  it to `cleared` and `public`. `clip-selection` and `social-asset-pack` branch on whether the
  work is leaving the company now rather than eventually. No script changed:
  `clearance-check` already supported this
- `customer-interview-engine` routes the case-study video's YouTube title and description to
  `social-asset-pack`, which already wrote `youtubeData.description` but was only ever handed
  clips
- 17 skills

## 1.0.0 – 2026-09-23

First release.

- 16 skills: a router, three for pipeline assets from existing recordings, five for the podcast,
  four for repurposing, one for webinars, and two for quality
- Every Riverside tool named in a skill was checked against the live MCP on 2026-09-21
  ([docs/riverside-mcp.md](docs/riverside-mcp.md))
- Approval gate: a Claude Code `ask` rule on every tool that posts, changes a post, renders an
  export, or edits the brand kit. Tested holding against an explicit allow
- Four scripts that decide instead of the model: `new-recording`, `clearance-check`,
  `stale-check`, `skill-landed`. Each exit path tested under the Bash 3.2 that ships with macOS
- `workspace/` and every audio, video and caption file type are gitignored
