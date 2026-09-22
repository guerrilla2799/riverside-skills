# One canonical cut per recording

The least exciting idea in this repo. Every other skill leans on it.

## The failure it prevents

The wrong version of an episode goes out. It's on the podcast host, the YouTube upload is
scheduled, three clips are queued, and the show notes quote timestamps from a cut that no longer
exists. Nobody wrote down which cut was the real one, so fixing it means opening every platform
and guessing what came from where. A twenty-minute fix turns into a lost evening.

The same failure hits a case study built from the wrong interview take, or a clip library that
still points at a sales call someone re-edited.

## Three moves

1. **One file is the source of truth, named and dated.** `canonical.md` records the recording,
   the Riverside edit, and the edit's revision number. Riverside revisions every edit, so
   `edit_id@revision` names one exact version.
2. **Every asset built from it gets logged, with where it went.** `publish-log.md` gets a row per
   clip, post, show note, export and host upload, stamped with the `edit_id@revision` it was built
   from.
3. **Change the source, and the log tells you what went stale.** `stale-check SLUG` compares every
   live row against the current canonical cut and lists the ones that don't match.

```
workspace/episodes/012-pricing-objections/
├── canonical.md       edit_id: 6a9f…  revision: 14
├── clearance.md       status: cleared  scope: public
├── publish-log.md     clip-01  6a9f…@14  LinkedIn  scheduled
│                      clip-02  6a9f…@12  X         published   ← stale
└── assets/
```

## What goes stale when the cut changes

Cuts move time. Anything that carries a timestamp, a duration or a file is suspect.

| Asset | Depends on | Re-derive by |
|---|---|---|
| Chapters | Playable timeline | Re-reading the aligned transcript on the new edit |
| Show notes | Quotes and timestamps | Re-checking each timestamp against the new edit |
| Clips | Source ranges | `editing_compare_revisions` shows what moved; re-cut the ones that shifted |
| Export | The whole edit | A new render |
| Host episode | The export | Replacing the audio, **keeping the same GUID** so apps don't list it twice |
| Scheduled posts | A clip | Editing the text in place; for new video, cancel and re-create |
| Published posts | A clip | Removing it on the platform itself. The MCP cannot unpublish |

## Rules that keep it working

- **Declare the canonical cut before any other work.** The pipeline skills refuse to build assets
  until `canonical.md` has an edit and a revision.
- **Branch, don't overwrite.** Try an alternative with `editing_clone_edit`. The canonical edit
  only moves on purpose, with a row in its History table saying why.
- **Supersede, never delete.** When an asset is replaced, mark the old row `superseded` and log
  the new one. The log is the only record of what was public and when.
- **Done means `stale-check` exits 0.** Not when the last upload finishes, and not when it feels
  finished.

`podcast-recut-and-republish` runs this end to end for the night it goes wrong.
