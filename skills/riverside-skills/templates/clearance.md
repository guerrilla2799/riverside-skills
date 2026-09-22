# Clearance

Who agreed to what, and where the proof lives. `clearance-check` reads this file before any
skill drafts an asset meant for outside the company, and refuses when it is incomplete.

status: pending
scope: internal
approved_by:
approved_on:
evidence:
restrictions:

## How to fill it in

- `status`: `pending`, `cleared`, `internal-only`, or `refused`
- `scope`: `internal` (team use only) or `public` (can leave the building)
- `approved_by`: name, role, company of the person who agreed. For a recording where you are
  the only speaker, write `self (sole speaker)`, and put `n/a – sole speaker` in `evidence`
  so the public check can pass
- `approved_on`: the date of the written approval, YYYY-MM-DD
- `evidence`: path or link to the written approval, such as an email saved into this folder or a
  signed release. A verbal yes on the call does not count
- `restrictions`: anything the approver ruled out (no revenue figures, no logo, first name only)

A public asset needs `status: cleared`, `scope: public`, and all of `approved_by`,
`approved_on`, and `evidence` filled in.
