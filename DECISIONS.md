# DECISIONS

ADR-style log. Append new entries at the top. Never silently contradict an
entry here — if new information suggests revisiting one, propose it
explicitly and add a new entry noting the supersession rather than editing
the old one.

---

## D-000: Pilot program is SNAP/TANF, not Medicaid
**Date:** 2026-07-11
**Context:** RFC 0003 uses Medicaid as its worked example, and Medicaid is
the highest-value program to fix. But Medicaid is not one pattern — it's at
least three (CO/IL/NC's clean category-dict pattern, MA's Medicaid→CHIP
fallback chain, and TX's four population-segmented calculators with no
category table at all, using PolicyEngine's raw output directly). Piloting
on Medicaid tests the new architecture and all override patterns
simultaneously, making failures hard to diagnose.
**Decision:** Pilot on SNAP/TANF first (pure pass-through in all 6 states,
zero exceptions, zero override-pattern complexity). Expand through Tier 2
(EITC/CTC/ACA/SSI/MSP/Head Start) before touching any override pattern.
Medicaid comes last, split into three sub-tiers matching its three real shapes.
**Alternatives considered:** Medicaid-first (rejected — conflates two hard
problems); starting with a custom (non-PolicyEngine) program (rejected — out
of scope, this project targets the PolicyEngine-delegated architecture specifically).

---

## D-001: Auto-discovery factory registration rejected (inherited from RFC 0003)
**Context:** RFC 0003's own "Alternatives Considered" section already
evaluated and rejected auto-discovering calculator configs via directory
scanning / Python introspection, in favor of explicit registration.
**Decision:** Follow the RFC's existing reasoning — explicit registration in
`CalculatorFactory`, one line per state/program. Do not re-litigate this
without new information; the original reasoning (debugging clarity at ~60+
calculators, migration control during phased rollout, no accidental
registration of draft configs) still applies.
**Source:** `rfc/rfcs/0003-program-calc-refactor/README.md`, "Alternatives Considered"

---

## D-002: JSON, not `.py`, for the data and config layers — but NOT for calculator logic
**Context:** The original RFC 0003 code samples use `.py` files for data and
config classes. Project owner wants JSON instead, for diffability,
schema-validation, and (critically) so a future drift-monitoring agent can
read config without parsing Python.
**Decision:** Data layer (benefit amount tables) and config layer (`pe_name`,
entity type, state dependency, base inputs) move to JSON, schema-validated.
Calculator logic (the override patterns — fallback chains, conditional
gates, population segmentation) stays in Python. Do not attempt to encode
control flow in JSON; that risks inventing an ad hoc rules DSL, which is a
much bigger and riskier undertaking than this project's scope.
**Open question, not yet decided:** whether JSON is read live at request time
or imported into DB rows at deploy time (see Phase 3 in `ROADMAP.md`) —
depends on how `import_program_config` currently works; investigate before deciding.

---

## D-003: PolicyEngine credentials — Docker self-host until Phase 4
**Context:** Hosted PolicyEngine API access requires emailing
hello@policyengine.org and waiting for OAuth `client_id`/`client_secret` to
be issued — no self-serve signup. MyFriendBen's own `.env` credentials came
from this same process.
**Decision:** Use the self-hosted Docker image
(`ghcr.io/policyengine/policyengine-household-api:current`) for all local
development through Phase 3. Do not request hosted credentials — from
PolicyEngine or from MyFriendBen — until Phase 4 genuinely requires them, if
it does at all (the Docker image "accepts the same household payload and
returns the same response shape" per PolicyEngine's own docs, so it may
suffice through Phase 4 too).
**Source:** https://policyengine.org/us/api

---

## D-004: TX Medicaid gets a fifth pattern, not forced into the RFC's four
**Context:** RFC 0003 documents four override patterns (Eligibility + Fixed
Amount, Fallback Chain, PE Value + Adjustment, Conditional Override). None of
these describe TX Medicaid's actual shape: four separate calculators
(children/pregnant/parents-caretakers/emergency), each with custom
member-level gating (age cutoffs, insurance-status checks, a cross-member
lookup checking whether a *different* household member — a child —
independently qualifies), with no category-value table at all.
**Decision:** Treat this as a genuine fifth pattern to design in Phase 7c,
not a special case to awkwardly fit into an existing pattern. Acceptable
fallback if this proves too complex to generalize: document TX Medicaid as
staying on the old architecture, rather than forcing a bad abstraction.
**Status:** Design not yet started. Revisit this entry once Phase 7c begins.

---

## D-005: `source` field in the data-layer schema is a structured status+citation object, not a free string
**Date:** 2026-07-12
**Context:** While drafting the data-layer JSON Schema (Phase 1, item 1) and
transcribing real WIC benefit-amount examples, every existing state's
`wic_categories`/`medicaid_categories` table turned out to have zero real
citation (CO, IL, NC — see `RISKS.md` Phase 1) or an explicit admission of
guessing (MA's `# NOTE: guesses based off Colorado`). `SCHEMA.md` already
called `source` out as required, "not optional," specifically to force a
quality bar the old code never had. But a plain required string is
trivially satisfiable by writing anything into it — a real citation and a
confident-sounding fabrication are indistinguishable to the schema itself.
**Decision:** `source` is a structured object, not a free string:
`{"status": "cited" | "unsourced" | "guessed", "citation": string | null}`.
`status` is required and machine-checkable; a future CI lint can require
`citation` to be non-null (and shaped like a real URL/document reference)
whenever `status` is `"cited"`, and flag any file where `status` is missing
or `citation` doesn't match. This makes the cited/uncited distinction part
of the schema's enforcement surface instead of an honor-system convention.
**Alternatives considered:** (a) plain required free-text string — rejected,
this is the exact problem being solved, an honest `"UNSOURCED — ..."` string
is indistinguishable from a fabricated-looking one to the schema; (b) hybrid
free-text `source` note plus a separate `citation_url: string | null` field —
rejected as a smaller change from the original sketch, but a null
`citation_url` is just as easily fabricated as a missing `status`, and it
doesn't capture the real three-way distinction this project's own WIC
research surfaced (a real citation vs. an admitted guess vs. a flat, never
sourced number) — `status` needing an explicit `"guessed"` value earns its
complexity given MA's own code comment already uses that exact word.
**Source (if applicable):** n/a — internal schema-design decision, not
sourced from RFC 0003 or upstream.

---

## D-006: `base_inputs` uses an `extends`+`additional_inputs` reference, not a flat duplicated list
**Date:** 2026-07-12
**Context:** Config-layer schema research confirmed the duplication problem
`ROADMAP.md`'s item 2 flagged is real: SNAP's federal base is 22
dependencies (`federal/pe/spm.py:4-23`), and all 6 states' `pe/spm.py`
subclasses do exactly `[*Snap.pe_inputs, <State>CodeDependency]` — same
22-entry federal list, one state-specific addition, zero deviation. A flat,
fully-duplicated `base_inputs: string[]` per state would repeat that
22-entry list 6 times for SNAP alone, and again for every other multi-state
program — `RISKS.md`'s Phase 1 section already flags dependency-name-as-
string as a rename footgun; flat duplication multiplies that blast radius
by the number of states sharing a program.
**Decision:** `base_inputs` is represented as `extends` (the `program` name
of a base config, or `null`) plus `additional_inputs` (the delta beyond
that base, or the complete list if `extends` is `null`) — mirroring
Python's own `[*Snap.pe_inputs, ...]` spread pattern. This is data
composition (the same category as the `$ref` sub-shape reuse
`test_case_schema.json` already does), not control flow, so it doesn't
cross D-002's line against branching/conditional logic in JSON. Chains are
capped at one level (`extends` must point to a config whose own `extends`
is `null`) — see `SCHEMA.md` for why this can't be a JSON Schema constraint
itself. The resolution mechanism (when/how `extends` actually gets merged
at runtime) is deferred to Phase 3's already-planned "JSON-loading layer"
decision.
**Alternatives considered:** (a) flat, fully self-contained list per state,
no composition — rejected: real duplication cost demonstrated above
(22 entries × 6 states for SNAP alone), and the existing dependency-name-
as-string footgun gets 6x worse, not smaller; (b) keep the schema flat now
and punt deduplication entirely to Phase 3 (e.g. a build step that expands
a shorthand into flat files) — rejected as deciding the JSON *shape* now is
cheap and Phase 3 already owns *when* `extends` gets resolved, so there's
no reason to also defer *what the file looks like*. Both alternatives were
presented via `AskUserQuestion` before this decision was made; `extends`
was chosen specifically to avoid the duplication cost neither alternative
avoids as well.
**Source (if applicable):** n/a — internal schema-design decision.

---

## D-007 template — copy this for new decisions

```
## D-XXX: <short title>
**Date:**
**Context:**
**Decision:**
**Alternatives considered:**
**Source (if applicable):**
```
