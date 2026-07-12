# UPSTREAM SYNC

Tracks this fork's relationship to `https://github.com/MyFriendBen/benefits-api`.
Update every time you rebase or merge from upstream. This repo moves fast —
don't let this file go stale, or you'll lose track of what's actually diverged.

## Current state

- **Pinned base commit:** `9ffb23facf75a3c0613085b485b1187075710c04`
- **Fork date:** 2026-07-11
- **Last rebase date:** _(update every time)_
- **Last rebase commit range:** _(e.g. "rebased through upstream commit abc123")_

## Rebase log

Append an entry every time you sync with upstream. Mechanical merges with no
real conflicts don't need much detail — conflicts that required an actual
decision do.

```
### YYYY-MM-DD
- Rebased onto upstream commit: <sha>
- Conflicts: none / describe
- Decisions required by a conflict: cross-reference DECISIONS.md if any
```

## Known upstream activity to watch for

- MyFriendBen's own `/add-pe-program` and `program-researcher` tooling may
  add new state/program calculators using the *old* architecture while this
  fork is in progress — expect to periodically reconcile newly-added
  upstream calculators against the patterns in `PATTERNS.md` rather than
  assuming the upstream program set is static.
- RFC 0003 itself may get updated, commented on, or superseded upstream —
  check `rfc/rfcs/0003-program-calc-refactor/discussion.md` periodically for
  any change from its empty-template state; if the RFC's design changes,
  that supersedes decisions here and needs a new `DECISIONS.md` entry, not a
  silent edit to old ones.

## Before requesting review / opening a PR upstream

- [ ] Confirm the fork is rebased onto a recent upstream commit, not stale
- [ ] Confirm no unrelated changes swept in via `git add .`/`-A` — MyFriendBen's
  own convention (per their `AGENTS.md`/skill files) is to stage specific
  files deliberately, to avoid an auto-formatter sweeping unrelated changes
  into a commit
