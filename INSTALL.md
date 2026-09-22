# Install

## Before you start

- **Claude Code**, installed and signed in.
- **A Riverside Grow, Webinar or Business plan.** That's where the MCP is included. Your
  Riverside role needs to be owner, admin, director or editor.
- **ffmpeg** (optional). Only the podcast skills use it, to prep the audio file for your host and
  to grab a frame for a thumbnail. On a Mac: `brew install ffmpeg`.

## 1. Connect the Riverside MCP

```bash
claude mcp add --transport http --scope user riverside https://mcp.riverside.com/mcp
```

Open Claude Code, run `/mcp`, choose **riverside**, and sign in to Riverside when the browser
opens. That sign-in is the authorization step, and it's the easiest one to skip without noticing.

`--scope user` makes Riverside available in every folder. Without it, the connection only works
in the folder where you ran the command.

To publish or schedule posts, connect your social accounts (YouTube, TikTok, Instagram,
Facebook, LinkedIn, X) in the Riverside dashboard first.

## 2. Install the skills

### Option A: work from the cloned folder (recommended)

```bash
git clone https://github.com/guerrilla2799/riverside-skills.git
cd riverside-skills
claude
```

Claude Code picks the skills up through `.claude/skills`, which links to `skills/`. The approval
gate in `.claude/settings.json` applies automatically, and everything you generate stays next to
the skills in `workspace/`.

On Windows, or anywhere symlinks don't survive the clone, run `cp -R skills .claude/skills` once
from inside the folder.

### Option B: copy the skills into your personal folder

```bash
git clone https://github.com/guerrilla2799/riverside-skills.git
cp -R riverside-skills/skills/* ~/.claude/skills/
```

Restart Claude Code. The skills are now available in every folder, and each project gets its own
`workspace/` wherever you run them.

**One extra step with this option.** The approval gate lives in the repo's
`.claude/settings.json`, which doesn't apply outside the cloned folder. Ask Claude:

```
Check my Riverside setup.
```

The router spots the missing gate and offers to add it to `~/.claude/settings.json`. It changes
nothing without a yes. To do it by hand, copy the `ask` list from this repo's
`.claude/settings.json` into the `permissions` block of `~/.claude/settings.json`.

## 3. Verify

Ask Claude:

```
List my five most recent Riverside recordings with their dates.
```

If your recordings come back, you're connected. If Claude says it doesn't have a Riverside tool,
run `/mcp` again, because the sign-in didn't finish.

Then run `Check my Riverside setup.` for the full check: connection, approval gate, workspace,
and ffmpeg.

## If you named the server something else

The approval gate matches tools by name, and the name includes the server:
`mcp__riverside__social_upload_create` if you ran the command above,
`mcp__claude_ai_Riverside__social_upload_create` if you connected through the Riverside
connector on claude.ai. Both are already in the list. If you chose a different server name, edit
the entries in `.claude/settings.json` to match, or the gate won't fire.

## Update

Option A: `git pull`. Option B: pull, then copy the skills again. Your `workspace/` is never
touched by either.

## Remove

Delete the cloned folder, or the skill folders you copied into `~/.claude/skills/`. To disconnect
Riverside, run `claude mcp remove riverside`. Nothing in your Riverside account changes.
