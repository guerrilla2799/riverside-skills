---
name: research-call-mining
description: >-
  Mine discovery and win/loss calls, "voice of customer", "what do prospects say". Quotes prospects verbatim from platform_get_transcript, counts themes by call, sets them beside homepage claims. Internal only. Case studies: customer-interview-engine.
---

# Research Call Mining

Discovery and win/loss calls hold the words buyers use for their own problem. This skill pulls
those words out of Riverside, verbatim and timestamped, counts how many calls say the same thing,
and sets the result beside what the homepage claims. The output is internal evidence for
messaging decisions. Nothing in it leaves the company without a clearance step.

## When to use
- Before a positioning, messaging or homepage rewrite
- Someone asks what prospects actually say, or why deals were won or lost
- A headline test needs a candidate that came from buyers rather than from the team
- A new team's first skill: it is internal, so there is nothing to clear before starting

## Inputs
- **Calls in Riverside.** Found in step 1. Calls stored in Gong, Zoom, Chorus or a dialer are not
  visible to the MCP. Either upload the recordings in the Riverside app, which makes them
  searchable recordings, or export the transcripts as text files into `workspace/transcripts/`
  and this skill reads them from there. Do not use `media_create_media_upload` for this: it puts
  files in the editor's Your Media library, which is not searched or transcribed as a recording
- **From the user:**
  - How many calls, or a date range. Default: the last 20
  - The homepage claims, as text. If the request still holds a placeholder such as
    [PASTE YOUR HOMEPAGE CLAIMS], ask for the claims before step 8. If only a URL is given, read
    the page, show the claims you extracted, and get a yes before comparing
  - Everyone on the user's side who speaks on calls (reps, SEs, founders), so prospect speakers
    can be told apart
  - Optional: known phrases to search for (a competitor, a pain phrase, the rep's opening question)
- **Workspace:** `./workspace`, or `RIVERSIDE_WORKSPACE` if set

## Workflow

1. **Find the calls.** Start with `platform_list_recordings` (newest first, `limit` up to 200,
   page with `offset`) and note title, date and id for each. Narrow with `search_riverside` using
   a TAKE filter, `[{"entity": "TAKE", "field": null, "value": null}]`, and
   `nextSearchAfter: null` on the first page. Use `search_recording_transcripts_exact` for known
   phrases, up to 20 per call, OR-combined. It covers only the last 120 days unless you pass
   `createdAfterDate`, so pass it whenever the range reaches further back. Search results carry
   short fragments and no timestamps. They locate calls. They are never quoted.
2. **Confirm the call set. STOP until the user confirms.** Label each candidate discovery,
   win/loss or unclear from its title and project. If any label is unclear, or the set is short
   of the number asked for, show the list (title, date, recording id, your label) and ask which
   calls are in. Do not continue on a guessed set. If a hit belongs to a multi-take recording,
   confirm the take: a `sessionId` can resolve to a sibling take.
3. **Pull and save each transcript.** `platform_get_transcript` with the recording's `sessionId`.
   Save each one to `workspace/research/call-mining-YYYY-MM-DD/transcripts/` as it arrives, one
   file per call. Exported transcripts in `workspace/transcripts/` are read in place.
4. **Sort the speakers. STOP on any unknown.** Mark each speaker label team side (on the user's
   list) or prospect side. A generic label ("Speaker 2") or a name on neither side goes to the
   user. No quote is attributed to a prospect until its speaker is confirmed prospect side.
5. **Extract every problem moment, prospect speakers only.** A problem moment is a prospect
   describing the pain, what it costs, what causes it, what they tried, why now, or (win/loss)
   why they chose or rejected a vendor. Not evidence: a rep's paraphrase ("so what I'm hearing
   is"), a prospect agreeing with the rep's framing ("yes, exactly", where the words are the
   rep's), a feature request with no problem attached. Copy the words from the saved transcript
   with no grammar fixes, and mark trims with "...". Stamp each moment with recording title, call
   date, timestamp and speaker role. Write timestamps as m:ss. The tool does not zero-pad, so
   `1:5.000` is 1:05.
6. **Verify every quote. Cut what fails.** Search the saved transcript for each quote literally
   (`grep -F`, one sentence at a time for longer quotes, each piece either side of an ellipsis).
   A quote not found verbatim is cut, never repaired. Report how many were cut. These are
   recording timestamps, good for finding the moment in Riverside. A clip made later takes its
   timing from an edit's playable timeline (`editing_read_aligned_transcript`), because cuts
   shift time.
7. **Group by theme and count calls.** Name each theme in the prospects' words, not the
   product's. For each theme: distinct calls it appears in, total moments, and the two or three
   clearest quotes. A call that raises a theme five times counts once. Sort by distinct calls.
   Frequency across calls beats intensity in one: one angry prospect is an anecdote. A theme heard
   on a single call goes in an Anecdotes list, not the theme table. For win/loss, split the count
   into won and lost where the outcome is known.
8. **Set it beside the homepage.** For each claim: the themes that support it with their call
   counts, the prospect wording for the same idea, and the gap. Then the themes no claim touches,
   by call count. Then the phrases prospects use that the site does not: check each against the
   pasted claims literally, case-insensitive, and list only phrases that do not appear there.
9. **Stamp and write.** Write `findings.md`. Its first line is always
   `INTERNAL ONLY. Prospect words from recorded calls. Not cleared for external use.`
10. **Gate any external use. STOP here if one is requested.** A quote in quote marks, or any
    words attributed to a person or company, going into a site, deck, ad or post is external. For
    a customer's story, route to `customer-interview-engine`. For anything else, run
    `../riverside-skills/scripts/new-recording SLUG` for that call if it has no folder, then
    `../riverside-skills/scripts/clearance-check SLUG public`. If it exits non-zero, STOP: show
    the script's stderr verbatim, keep the quote internal, and offer to draft the consent request.
    A headline written in the prospects' language, with no quote marks, name or attribution, is
    the team's own copy and needs no clearance. If the phrase is distinctive enough to identify
    who said it, treat it as a quote.
11. **End with one recommendation.** The phrase worth a headline test, verbatim, with the number
    of calls it came from, the claim it would test against, and the page it belongs on: the
    homepage hero for problem language heard early in calls, a comparison or alternatives page for
    switching language, the pricing page for cost and risk language. One phrase, one page, one
    reason.

## Riverside tools

| Step | Tool | Note |
|---|---|---|
| 1 Find | `platform_list_recordings` | Newest first. Page with `limit` and `offset` |
| 1 Find | `platform_get_project` | Recordings and edits in one call, when calls are grouped by project |
| 1 Find | `search_riverside` | TAKE filter for spoken moments. 10 results a page. Explicit `null` for unused parameters |
| 1 Find | `search_recording_transcripts_exact` | Last 120 days by default. Pass `createdAfterDate` to reach further back |
| 3 Pull | `platform_get_transcript` | Prose, one `Speaker (m:s.mmm)` stamp per sentence, measured from the start of the recording |
| Never | `platform_get_recording` with `includePreviewUrl` | Studio-wide, non-revocable share token. Not needed here |

## Output
- **Writes:** `workspace/research/call-mining-YYYY-MM-DD/findings.md`, with the saved transcripts
  in `transcripts/` beside it. `workspace/` is gitignored. Keep it that way: these are prospect
  calls
- **findings.md holds:** the INTERNAL ONLY line; a source table (title, date, recording id, call
  type); the theme table; the homepage comparison; phrases the site does not use; anecdotes;
  every verified moment grouped by theme; the recommendation
- **Prints:** calls read, moments found, quotes cut in verification, the top five themes with call
  counts, the biggest gap in one sentence, and the recommendation
- **No publish-log rows.** Nothing here is built from an edit or leaves the company. Every quote
  carries its recording id, so a later external use can be traced and cleared

## Rules & quality bar
- **Prospect words only.** Rep lines, rep paraphrases and one-word agreements are not evidence
- **Verbatim or cut.** A quote that fails the literal search in step 6 is removed, never reworded
- **Count calls, not mentions.** The theme table sorts by distinct calls
- **Never guess the call set or a speaker's side.** Steps 2 and 4 list what was found and ask
- **INTERNAL ONLY on line one,** every run
- **Quotes leave only through step 10.** The script decides clearance, not the model
- **Roles, not names,** for prospect speakers in `findings.md`, so it can circulate internally
  without exposing individuals

## Related skills
- Pairs with: `customer-interview-engine`, for a named customer's story cleared in writing
- Pairs with: `sales-enablement-clips`, which searches the same calls for objections
- Feeds: `content-quality-gates`, which judges the headline copy before it ships
- Setup and shared scripts: `riverside-skills`. Tool catalog and seams: `docs/riverside-mcp.md`
