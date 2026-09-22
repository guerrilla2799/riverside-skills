---
name: skill-capture-loop
description: >-
  Use for "turn this into a skill", "I keep doing this by hand", "the skill got it wrong". Builds a skill from a narrated Riverside recording via platform_get_transcript and proves each fix with skill-landed. Asset review: content-quality-gates.
---

# Skill Capture Loop

Turns repeated work into skills, and proves every correction to a skill made it into the file. Two loops. The capture loop: the third time a task gets done by hand, it becomes a skill, built from what you did and what you said while doing it. The correction loop: when a skill gets something wrong, the fix goes into the skill file, gets committed, and a script confirms it landed. A correction that lives only in the chat is gone next session.

## When to use
- The third time the same task gets done by hand
- At the start of a session whose task should come out as a skill
- The task lives in your head or on screens Claude cannot see, so recording a walkthrough beats explaining it
- A skill got something wrong and you corrected it in the session

## Inputs
- **Tally:** `workspace/skill-candidates.md`, one line per task done by hand. Created on first use
- **For capture from a recording:** a Riverside recording of you doing the task once, screen share on, narrating. Record it in the Riverside app, because the MCP cannot start or schedule a recording
- **Knowledge files** the new skill should read: voice guide, ICP doc, brand rules, approved claims. Their paths, never their contents
- **A skills folder under git,** such as the project's `.claude/skills/` or `~/.claude/skills/`. `skill-landed` exits 3 on a SKILL.md outside a git repo, and then nothing proves a fix survives
- The script is called by its path from this skill's base directory: `../riverside-skills/scripts/skill-landed`

## Workflow

### 1. Count to three
Each time a task gets done by hand, add the date to its line in the tally:

```
weekly win-loss summary | 2026-08-04, 2026-08-18, 2026-09-01
```

On the third date, ask: "This is the third time. Want it as a skill?" Not sooner. One run shows a procedure. Three show which steps actually repeat and which were one-offs. If the user says no, write `declined` on the line and stop asking.

### 2. Declare the intent before the task starts
The declaration comes first, before any work: "We are doing TASK by hand this session, and the session ends with a skill for it." From then on, keep a capture list as you go: every step, every input and where it came from, every decision with the reason given, every correction the user made. Reconstructing afterwards keeps the steps and loses the decisions, and the decisions are the part worth encoding.

At the end, the draft skill accounts for every item on the capture list. Each item is either written in or struck with a stated reason. Until then the draft is not done.

### 3. Or record it once in Riverside
When explaining the task would take longer than doing it, do it once on a Riverside recording with screen share on, narrating as you go.
- **Say everything out loud.** Only speech is transcribed, never the screen. Say the file names, field names, thresholds, what you check and why. A click you do not narrate does not exist for the skill
- **Say roles, not names:** "the customer's CFO". Anything named on the recording lands in the transcript
- Find the recording with `platform_list_recordings`, newest first. Read it with `platform_get_transcript`, passing the recording id as `sessionId` for a single-take recording
- Extract, each with its transcript timestamp in your working notes: the steps in order, every decision ("if", "unless", "I check X first"), every reason ("because", "otherwise"), every input named, and every place you said it breaks
- Where the transcript skips a step, list the gaps and ask. Never fill one with a guess

### 4. Write the skill
**Point at knowledge files, never carry copies.** The skill says "Read PATH before drafting." It never pastes the voice guide, the ICP or the brand rules into its body. A pasted copy goes stale the day the original changes, and nothing tells you which skills still hold the old version. Check before saving: take two distinctive sentences from each knowledge file and run `grep -F "SENTENCE" SKILL.md`. Any hit is a copy. Replace it with the path.

**Scrub names.** List every person and company named in the transcript or the session, and grep the draft for each. Replace every client, customer or colleague name with a role.

**Put each must-do in a workflow step with a STOP branch:** "Run X. If it fails, STOP: tell the user Y and do not continue." A rule that lives only in a Rules section gets skipped.

Frontmatter rules for any new skill:
- `name` equals the folder name. Lowercase and hyphens
- `description` is a folded block scalar, `description: >-`, so quotation marks and colons inside it cannot break the YAML
- The description opens with trigger words: the phrases a user actually types
- It names the literal operations the skill runs, such as `platform_get_transcript` or a script name, so a request that names the operation finds the skill
- It ends with a short boundary naming the sibling skill that owns the neighbouring job. This repo holds descriptions to 150–250 characters, which forces the trigger words to the front

Template:

```
---
name: SKILL-NAME
description: >-
  Use for "TRIGGER PHRASE", "SECOND PHRASE". Does WHAT, using OPERATION. Not for NEIGHBOURING JOB: SIBLING-SKILL.
---

# Skill Title
One paragraph: what it does and why it exists.

## When to use
## Inputs
Each input and where it comes from. Knowledge files by path: Read PATH.
## Workflow
1. STEP. If CHECK fails, STOP: tell the user WHAT and do not continue.
## Output
Files written, where, and what the user sees.
## Rules & quality bar
## Related skills
```

Commit the new skill. Then start a fresh session and type one of its trigger phrases. If the skill does not load, the description is wrong, not the body.

### 5. Close every correction with skill-landed
When a skill gets something wrong in a session:
1. Correct the output in the session
2. Update the skill file. The user says "update SKILL-NAME so this does not happen again", or you offer to. The fix goes into the workflow step where the failure happened
3. Pick PHRASE: 4 to 10 words copied from the new line exactly as written, since the match is exact and case-sensitive. Confirm it sits on an added line in `git -C SKILL_DIR diff -U0 -- SKILL.md`. A phrase that was already in the file proves nothing
4. Show the user the diff, then commit it
5. Run `../riverside-skills/scripts/skill-landed SKILL_DIR "PHRASE"`, with SKILL_DIR as an absolute path. The script resolves it from the current directory, not from this skill's folder

| Exit | Meaning | Do this |
|---|---|---|
| 0 | The phrase is in SKILL.md, committed, nothing pending | Report the `LANDED` line and the commit it printed. The loop is done |
| 1 | The phrase is not in SKILL.md. The fix did not land | STOP. Tell the user the correction exists only in this chat. Apply it to the file, commit, re-run |
| 2 | The phrase is there, but SKILL.md has uncommitted changes | STOP. Commit, re-run |
| 3 | SKILL.md is missing, not in a git repo, or never committed | STOP. Tell the user. With their okay, put the skills folder under git, commit, re-run |
| 64 | Bad arguments | Check the quoting around PHRASE, re-run |

The loop is not done until it exits 0. On any other exit, never tell the user the skill is fixed.

If the fix belongs in a knowledge file instead (the voice guide was wrong, not the skill), make it there. `skill-landed` reads only SKILL.md, so check the file directly: `grep -F "PHRASE" FILE` finds the phrase, and `git status --porcelain FILE` prints nothing.

## Riverside tools
| Step | Tool | Note |
|---|---|---|
| 3 | `platform_list_recordings` | Newest first. Never ask `platform_get_recording` for a preview URL: it is a studio-wide, non-revocable credential |
| 3 | `platform_get_transcript` | `sessionId` is the recording id for a single-take recording. One `Speaker (m:s.mmm)` stamp per sentence, not zero-padded: `1:5.000` is 1m05s |
| 3 | `search_recording_transcripts_exact` | Finds an older walkthrough by a phrase you remember saying. Last 120 days unless you pass `createdAfterDate` |

## Output
- A new skill at `SKILL_DIR/SKILL.md`, committed
- The tally at `workspace/skill-candidates.md`
- For a correction: the edit, its commit, and the `LANDED` line from `skill-landed` shown in the chat
- Prints the capture list against the draft, with each gap asked as a question

## Rules & quality bar
- **The third time, not the first.** Two runs cannot show which steps are stable
- **Declare before the task, not after.** Decisions are lost in reconstruction
- **The transcript and the session are the only sources.** No step, threshold or reason neither of them contains
- **Point at knowledge files, never copy them**
- **Done means exit 0.** "I updated the skill" is a claim. `skill-landed` is the proof
- **Fixes go into workflow steps,** with a STOP branch when they are gates
- **No client, customer or colleague names in a skill file**

## Related skills
- `content-quality-gates`: judges outbound assets. It does not review skill files
- `riverside-skills`: the router, and home of `skill-landed`
- `transcript-to-written`: when the walkthrough should become an article, not a skill
