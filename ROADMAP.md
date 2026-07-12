# ROADMAP

Status legend: `[ ]` not started · `[~]` in progress · `[!]` blocked · `[x]` done

Update the status and "Current state" note at the end of every session. Do not
mark a phase `[x]` without the human's explicit confirmation.

---

## Phase 0 — Groundwork
**Status:** [x]
**Current state:** All 6 checklist items complete and verified (not just written).
Toolchain runs on a local venv pinned to Python 3.13, not the .python-version's
3.10, due to a broken Homebrew python@3.10 bottle on this machine — several
requirements.txt pins needed per-package workarounds to install; full detail
in RISKS.md. Postgres confirmed required (no throwaway/sqlite path exists).
PolicyEngine Docker container verified healthy locally. Item 5's RFC comment
is drafted in PENDING_COMMS.md but not yet posted — blocked on collaborator
access to MyFriendBen/rfc, accepted as done for phase-completion purposes
given that blocker is external. Local .env/native-Postgres/Redis setup
needed to actually *run* pytest is intentionally deferred, not part of this
phase's scope.

- [x] Fork `benefits-api`, pin base commit (record it in `UPSTREAM_SYNC.md`)
- [x] Read `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` from the upstream repo in full
- [x] Set up local CI toolchain (`black`, `pytest`, `make format`/`make test`) — see `RISKS.md` Phase 0 for the ruff/mypy correction and Python 3.13 dependency workarounds
- [x] Verify whether `python manage.py test` needs a live Postgres connection
  or spins up its own throwaway test DB — this determines local setup scope
  (confirmed: needs live Postgres, see `RISKS.md` Phase 0)
- [x] Open a comment/draft PR on `rfc` repo's `0003-program-calc-refactor`
  stating intent to build a working prototype; ask if the design still holds
  (drafted in `PENDING_COMMS.md`, not yet postable — no collaborator access;
  human confirmed draft-ready counts as done given this external blocker)
- [x] Spin up PolicyEngine via Docker (`docker run --rm -p 8080:8080
  ghcr.io/policyengine/policyengine-household-api:current`) — no hosted
  credentials needed for this phase (confirmed healthy at localhost:8080;
  cold start takes ~2-3 min while it loads model data, see `RISKS.md`)

**Do not** request hosted PolicyEngine API credentials yet. See `RISKS.md`.

---

## Phase 1 — JSON schema design (data + config layers only)
**Status:** [~]
**Current state:** Items 1-3 done and validated. Data-layer schema (item 1)
validated against real WIC examples (CO, NC) — WA WIC ruled out of scope
(D-000), no WIC state has a real citation (CO/NC uncited/guessed, MA/IL's
numbers may just be uncredited copies of CO's — flagged for MyFriendBen
separately, see `RISKS.md`). Config-layer schema (item 2) validated against
real SNAP examples (federal base, CO, TX) — `base_inputs` is an
`extends`+`additional_inputs` reference rather than a flat list, to avoid
repeating SNAP's 22-entry federal dependency list 6 times (D-006). Also
surfaced: a `pe_period_month` field the first draft missed (SNAP-only,
added and documented), and a likely latent upstream bug where MA WIC never
sends its state-code dependency to PolicyEngine (`RISKS.md`, not fixed).
Item 3's both halves (CO/TX SNAP config; CO/NC WIC data) are covered by
items 1-2's worked examples. Items 4-5 not started.

- [x] Write JSON Schema for the data layer (benefit amount tables) —
  validate against WIC, not SNAP (SNAP has no value table, a bad test case)
  (`schemas/data_layer.schema.json`; `source` is a structured
  `{status, citation}` object per `DECISIONS.md` D-005, not free text —
  see `RISKS.md` Phase 1 for why)
- [x] Write JSON Schema for the config layer — `pe_name`, `pe_entity`,
  `state_dependency`, `base_inputs` (dependency names as strings)
  (`schemas/config_layer.schema.json`; `base_inputs` is an
  `extends`+`additional_inputs` reference, not a flat list, per
  `DECISIONS.md` D-006 — avoids repeating SNAP's 22-entry federal
  dependency list 6 times over)
- [x] Hand-transcribe 2-3 existing examples into the schema to sanity check
  it (CO/TX SNAP config; CO/NC WIC data — corrected from "CO/WA", WA WIC is
  a custom non-PolicyEngine program, out of scope per D-000; see `RISKS.md`
  Phase 1) before building anything that reads it
- [ ] Register schemas at a stable, fetchable location (mirror how
  `test_case_schema.json` is published)
- [ ] Decide and document in `DECISIONS.md`: how mixin-style shared logic
  (e.g. `IlMedicaidFplIncomeCheckMixin`) is represented — this is an RFC 0003
  open question the original discussion never answered

**Gate before Phase 2:** schema must NOT contain conditional/branching fields.
Data and wiring only. See `PATTERNS.md` for why.

---

## Phase 2 — `PolicyEngineClient`
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Implement `get_member_value` / `get_spm_value` / `get_household_value`
  wrapping the existing `Sim` object — do not touch `Sim`/`engines.py`
- [ ] Unit test suite using `vcrpy` fixtures per RFC 0008's established convention
- [ ] Write and pass a test proving the client does NOT trigger a second live
  API call when wrapping an existing `Sim` instance (RFC 0003's own open question)

**Gate before Phase 3:** get real usage feedback from writing the Phase 4 SNAP
calculator before considering this interface final — revisit if needed.

---

## Phase 3 — Factory + calculator base
**Status:** [ ]
**Current state:** _(update here)_

- [ ] `CalculatorFactory` with explicit registration (not auto-discovery —
  see `DECISIONS.md` for why this was already rejected by the RFC)
- [ ] JSON-loading layer — decide read-at-request-time vs. import-to-DB-at-deploy-time
  (check how `import_program_config` currently handles this)
- [ ] Confirm old and new calculators can coexist and be called
  interchangeably by the existing orchestration layer

---

## Phase 4 — Pilot: SNAP, single state, full parallel verification
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Audit every state's existing `pe/spm.py` Snap subclass for hidden
  overrides before assuming purity holds (check WA's BBCE net-income-test
  waiver specifically — flagged in its `spec.md`)
- [ ] `SnapCalculator` (should be nearly empty — SNAP is a pure pass-through)
- [ ] CO SNAP config/data JSON (start here — original deployment, best-tested)
- [ ] Decide and document the output-comparison tolerance threshold BEFORE
  running parallel verification, not after seeing a mismatch
- [ ] Run new vs. old `CoSnap` side by side against test/staging data
  (never production PII) — 100% parity required before extending
- [ ] Extend to remaining 5 states' SNAP
- [ ] Extend to TANF (second confirmation of pure-pass-through pattern)

**Human confirms parity results before this phase is marked done.**

---

## Phase 5 — Tier 2: EITC, CTC, ACA, SSI, MSP, Head Start
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Migrate one `tax_unit` program (EITC) — stress-tests client beyond `spm_unit`
- [ ] Migrate one `person`-entity program (SSI or Head Start)
- [ ] Recheck each target program for hidden state-specific adjustments
  before assuming Tier-2 purity (don't repeat the SNAP oversight)

---

## Phase 6 — First override pattern: Eligibility + Fixed Amount
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Migrate CHP (CO), EveryDayEats (CO), or IL LIHEAP
- [ ] Note: CHP's real condition is "eligible AND has no insurance" — a
  conditional layered on a fixed amount, not a pure fixed-amount case. Design
  the base class to compose (condition fn + fixed amount), don't treat CHP as
  proof the simple case alone is sufficient.

---

## Phase 7 — Medicaid, in three sub-tiers
**Status:** [ ]
**Current state:** _(update here)_

### 7a — Base category-dict pattern (CO, IL, NC)
- [ ] Should be mechanical if Phases 1-6 are solid

### 7b — MA's fallback chain
- [ ] New logic: calculator must support "try variable A, fall back to
  variable B" (Medicaid → CHIP) — not yet built in any earlier phase

### 7c — TX's population-segmented shape (the fifth pattern, missing from the
original RFC entirely)
- [ ] Design a calculator shape defined as a *set* of (PE variable, member-level
  gating condition) pairs, with support for cross-member lookups (the
  caretaker-eligibility check against a child's Medicaid status)
- [ ] **Highest-risk step in the project.** Budget real design time. If it
  can't generalize cleanly, it is legitimate to leave TX Medicaid on the old
  architecture as a documented exception rather than force a bad abstraction.

### Also in this phase
- [ ] `KffMedicaidClient` (or equivalent) for the value-table data WA/KS
  already source from KFF/KHI — separate sub-task, own risk profile
  (scraping vs. API access, category-taxonomy mapping)
- [ ] Fix the citation gap: CO/IL/MA/NC currently have unsourced Medicaid
  value tables — cheap to fix while already touching every state's data file

---

## Phase 8 — Testing, docs, PR sequence
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Write the "migration guide for adding new states" (RFC calls for this,
  never wrote it)
- [ ] Full RFC 0008-compliant test coverage
- [ ] Split into a sequence of small, independently-reviewable PRs mirroring
  these phases — not one large PR

---

## Phase 9 — Land it, or don't, gracefully
**Status:** [ ]
**Current state:** _(update here)_

- [ ] Set an explicit walk-away checkpoint NOW (e.g., "if Phase 4's PR gets no
  engagement within a month of opening") — decide before you're emotionally
  invested in the outcome
- [ ] If accepted: help drive the RFC's own Contract phase (remove old
  calculator classes) and finish its test-coverage phase
- [ ] If stalled: the fork + a monitoring tool built on top of it (see
  `DECISIONS.md` re: the deferred "idea #1" periodic drift-check) is still a
  complete, demonstrable result independent of MyFriendBen's decision
