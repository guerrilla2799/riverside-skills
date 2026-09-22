# workspace/

Your working files live here and are gitignored. One folder per recording, whether it's a sales
call, a customer interview, a podcast episode or a webinar:

```
workspace/episodes/SLUG/
├── canonical.md      which cut is the source of truth: recording, edit, revision
├── clearance.md      who approved what, when, and where the written proof is
├── publish-log.md    every asset built from it, where it went, and which cut it came from
└── assets/           drafts: clip shortlists, show notes, captions, case studies
```

Create one with `skills/riverside-skills/scripts/new-recording SLUG`.

A few skills also write outside the per-recording folders:

```
workspace/research/call-mining-DATE/      research-call-mining: themes, quotes, the homepage comparison
workspace/transcripts/                    transcripts you exported from Gong, Zoom or a dialer
workspace/enablement/objection-library.md sales-enablement-clips: objection, clips, source, clearance
workspace/skill-candidates.md             skill-capture-loop: tasks done by hand, and how often
```

`canonical.md` is what lets `stale-check` tell you what went out of date when a cut changes.
`clearance.md` is what `clearance-check` reads before anything leaves the building. See
[docs/canonical-cut.md](../docs/canonical-cut.md).
