# SESSION LOG

Append-only. Newest entry at the top. One entry per Claude Code session.
At the start of a new session, read only the most recent entry (plus
`ROADMAP.md`) — don't require re-reading the whole history to resume.

Template for each entry:

```
## YYYY-MM-DD — Session N

**Phase worked on:** (e.g. Phase 1)

**Completed:**
-

**Blocked / open:**
-

**Decisions made this session:** (cross-reference DECISIONS.md entry IDs)
-

**New risks discovered:** (cross-reference RISKS.md)
-

**Next concrete action:**
(the literal first thing the next session should do — not a vague pointer)
```

---

## 2026-07-12 — Session 2

**Phase worked on:** Phase 0

**Completed:**
- All 6 Phase 0 checklist items verified complete (see ROADMAP.md diff above for detail)
- Corrected CLAUDE.md's stale claim that code style is enforced via ruff/mypy — actual upstream toolchain is black only
- Rebuilt local venv on Python 3.13 after Homebrew's python@3.10 bottle proved broken on this machine (libexpat symbol mismatch in ensurepip); worked around several requirements.txt pins with no cp313 wheels (numpy/kiwisolver skipped as unused cruft; pillow/psycopg2/psycopg2-binary/grpcio/grpcio-status/google-cloud-translate/googleapis-common-protos/google-api-core installed unpinned) — full detail in RISKS.md
- Confirmed via settings.py + CI config that live Postgres is required, no throwaway/sqlite test DB path exists
- Verified PolicyEngine's self-hosted Docker image runs and is healthy locally
- Drafted the RFC intent-to-build comment for PR #5 on MyFriendBen/rfc; created new PENDING_COMMS.md to track it (and future drafts blocked on access), referenced from CLAUDE.md

**Blocked / open:**
- PENDING_COMMS.md's RFC comment still needs to be actually posted once collaborator access to MyFriendBen/rfc is granted
- Local .env + native Postgres + Redis setup (needed to actually run pytest/manage.py test) intentionally deferred — not scoped into Phase 0

**Decisions made this session:**
- CLAUDE.md's ruff/mypy claim corrected to black-only
- Local dev stays on Python 3.13 rather than chasing the broken Homebrew 3.10 bottle further; per-package pin overrides documented in RISKS.md
- RFC feedback goes as a plain comment on PR #5, not a new PR (no org membership to open PRs on MyFriendBen/rfc)
- "Drafted and ready in PENDING_COMMS.md" accepted as satisfying item 5 given the access blocker is outside the human's control

**New risks discovered:**
- See RISKS.md Phase 0 section in full: Python 3.10/3.13 mismatch + per-package workarounds, Makefile missing a `test` target, PolicyEngine Docker cold-start health-check noise, open .env/Postgres/Redis gap

**Next concrete action:**
Decide whether to start Phase 1 (JSON schema design) next session, or first close the .env/native-Postgres/Redis gap so pytest can actually run end-to-end.

---

## (seed entry — replace with your first real session)

**Phase worked on:** Phase 0 not yet started

**Completed:**
- Planning conversation completed; project scaffolding docs (`CLAUDE.md`,
  `ROADMAP.md`, `DECISIONS.md`, `PATTERNS.md`, `SCHEMA.md`, `RISKS.md`,
  `SESSION_LOG.md`, `UPSTREAM_SYNC.md`) created and ready to paste into workspace.

**Blocked / open:**
- None yet — project has not started Phase 0.

**Decisions made this session:**
- D-000 (pilot on SNAP/TANF, not Medicaid)
- D-001 (explicit factory registration, inherited from RFC)
- D-002 (JSON for data/config, Python for logic)
- D-003 (Docker self-host until Phase 4, no premature credential requests)
- D-004 (TX Medicaid gets a genuine fifth pattern, not forced into the RFC's four)

**New risks discovered:**
- See `RISKS.md` in full — seeded from planning discussion, not yet from
  actual implementation work.

**Next concrete action:**
Fork `benefits-api`, pin the base commit, record it in `UPSTREAM_SYNC.md`,
and begin Phase 0's checklist in `ROADMAP.md`.
