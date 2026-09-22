# Changelog

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
