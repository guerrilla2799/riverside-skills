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
