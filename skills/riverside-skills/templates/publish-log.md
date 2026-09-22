# Publish log

Every asset built from this recording, where it went, and which version of the canonical cut
it came from. `built_from` is `edit_id@revision` copied from `canonical.md` at the moment the
asset was made. Leave `reference` blank until there is one (a Riverside uploadId, a host
episode id, a URL, a file path).

States: `draft` · `approved` · `scheduled` · `published` · `superseded` · `retired`

`stale-check` ignores rows marked `superseded` or `retired`. Mark a row `superseded` once its
replacement is logged, never delete it.

| date | asset | built_from | destination | reference | state |
|---|---|---|---|---|---|
