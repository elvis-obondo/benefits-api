# CLAUDE.md — Project Behavior Rules

This file governs *how* Claude Code should work on this project across sessions.
It does not contain project content — for that, read the files it points to below.

This project is a from-scratch implementation of `benefits-api`'s dormant
**RFC 0003: Program Calculator Architecture Refactor**, forked from
https://github.com/MyFriendBen/benefits-api, with the goal of eventually
proposing it back upstream. The owner of this project is a human who wants to
stay actively involved in every design decision — this file exists to make
sure that happens by default, not by accident.

## Read these first, every session

1. `SESSION_LOG.md` — most recent entry only, to resume context
2. `ROADMAP.md` — current phase, status, and what's next
3. `DECISIONS.md` — do not repropose anything already decided here without new information

Only read `PATTERNS.md` and `SCHEMA.md` when the task actually touches calculator
logic or the JSON schema — they're reference material, not session-start reading.

Before drafting any new external comment/PR/issue text, check `PENDING_COMMS.md` —
it may already have a drafted version, or a reason a previous draft is stale.

## Inherited rules (from the fork)

This repo inherits MyFriendBen's own `CLAUDE.md` and `AGENTS.md` conventions
(code style via `black`, commit conventions, the "three-question gate"
before editing partner-facing contract tests, changelog fragment format).
Where their rules and this file conflict, **this file wins** for anything
related to project ownership and design-decision process. Their rules win for
code style, testing conventions, and anything specific to how `benefits-api`
itself is built.

## Hard stop-and-ask checkpoints

Claude must stop and get explicit approval before:

- Moving from one Phase to the next in `ROADMAP.md` (do not silently start
  Phase 2 work while "finishing up" Phase 1)
- Changing the JSON schema (`SCHEMA.md`) — propose the change, wait for
  confirmation, then implement
- Making any design decision not already recorded in `DECISIONS.md` — propose
  2-3 options with tradeoffs, do not pick one and proceed
- Requesting or using PolicyEngine hosted API credentials (`client_id`/`client_secret`)
  for anything beyond local Docker self-hosting — see `RISKS.md` for why
- Rebasing onto a new upstream commit — log it in `UPSTREAM_SYNC.md` first
- Declaring a phase "done" — the human confirms parallel-verification results
  before a phase is marked complete in `ROADMAP.md`

## Working agreement

- Prefer small, reviewable commits/PRs over large ones — mirrors the phase
  structure in `ROADMAP.md`, and matches the plan to eventually PR this
  upstream in digestible pieces.
- When something in the existing `benefits-api` code doesn't match what
  `PATTERNS.md` describes, stop and flag it rather than assuming the doc is
  right — the codebase moves fast and the docs may be stale.
- Follow MyFriendBen's own git-hook convention of stripping AI co-author
  lines from commits before they'd land upstream, even in this fork, so the
  history looks native if this is ever proposed as a PR.
- At the end of every session, append an entry to `SESSION_LOG.md` (what was
  done, what's blocked, what decisions are pending, the next concrete action)
  and append any newly discovered risk to `RISKS.md`.

## Project goal, one sentence

Prove that RFC 0003's composition-based architecture (data/config JSON +
shared calculator classes + factory pattern) makes adding a new state's
PolicyEngine-backed program a JSON-only change for simple programs, starting
with SNAP/TANF and working up to Medicaid's harder patterns — with enough
working code to make reviving the RFC upstream a credible ask.
