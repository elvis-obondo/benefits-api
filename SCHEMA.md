# SCHEMA

Human-readable companion to the actual JSON Schema files (keep those in
`schemas/` in the repo proper — this file explains them, doesn't replace them).

**Status: not yet designed.** This file is a skeleton to fill in during
Phase 1. Update the "Fields" tables and examples as the schema solidifies —
do not let this drift out of sync with the actual `.schema.json` files.

## Scope discipline

This schema covers **data and config only** — Pattern 1 (pure pass-through)
and the data/config portions of Patterns 2-5. It does NOT encode control
flow. If a task seems to need a conditional field in this schema to express
program logic, stop — that belongs in a calculator's Python, not here. See
`PATTERNS.md` and D-002 in `DECISIONS.md`.

## Config layer schema (Phase 1)

Purpose: wires a state's program to a PolicyEngine variable.

| Field | Type | Notes |
|---|---|---|
| `program` | string | |
| `state` | string | Use `"federal"` as the sentinel for a base config other configs `extend` |
| `pe_name` | string | The PolicyEngine variable name, e.g. `snap`, `co_tanf` |
| `pe_entity` | enum | `spm_unit` \| `tax_unit` \| `person` \| `household` \| `family` \| `marital_unit` — see `PATTERNS.md` for base-class mapping. No literal `pe_entity` attribute exists in the current Python code; entity type is implied by which `PolicyEngine*Calculator` base class a program subclasses, and never varies by state for the same program. This field makes that choice explicit so Phase 3's factory doesn't have to re-derive it from inheritance. |
| `state_dependency` | string \| null | Name of the state-code dependency class, e.g. `CoStateCodeDependency` — **must resolve to a real class**; validate in CI. `null` for federal-only configs. |
| `extends` | string \| null | The `program` name of the base config this one extends (e.g. `"snap"`), or `null` if fully self-contained. See "How `base_inputs` avoids duplication" below — this replaces a flat `base_inputs` list. |
| `additional_inputs` | array of strings | Dependency class names beyond `extends`'s base (or the complete list, if `extends` is `null`). Does not include `state_dependency` — tracked separately so it isn't duplicated between the two fields. |
| `pe_period_month` | string \| null | Default `null` (use `pe_period` as-is, no sub-month suffix). Only federal SNAP sets this today, to `"01"` — see `federal/pe/spm.py`'s `Snap.pe_period_month`/`pe_output_period`. Confirmed via a full `programs/` sweep that no state overrides it and no other program defines it; stated explicitly in every example rather than assumed. |

### How `base_inputs` avoids duplication

SNAP's federal dependency list is 22 entries
(`federal/pe/spm.py:4-23`), and all 6 states' SNAP subclasses do exactly
`[*Snap.pe_inputs, <State>CodeDependency]` — repeating that list verbatim in
every state's JSON config would multiply the existing dependency-name-as-
string footgun (`RISKS.md` Phase 1) by 6. Instead, `extends` names the
`program` of a base config, and `additional_inputs` holds only the delta —
mirroring Python's own spread pattern. See `DECISIONS.md` D-006 for the
full reasoning and rejected alternatives.

**Depth constraint (enforced outside this schema, not by it):** a config
whose `extends` is non-null must point to a config whose own `extends` is
`null` — one level only, no chains, no cycles. JSON Schema draft-07 can't
express this — the referenced config lives in a *different file*, and
schema validation only ever evaluates one document at a time. This is a
runtime/CI check, same category as "every `state_dependency` must resolve
to a real class" above. No real example currently needs more than one
level (every case found is state-extends-federal), so this is a
forward-looking guardrail, not a fix for an observed violation.

**Worked example — SNAP, federal base:**

```json
{
  "program": "snap",
  "state": "federal",
  "pe_name": "snap",
  "pe_entity": "spm_unit",
  "state_dependency": null,
  "extends": null,
  "additional_inputs": ["SnapUnearnedIncomeDependency", "SnapEarnedIncomeDependency", "... 20 more federal entries ..."],
  "pe_period_month": "01"
}
```

**Worked example — SNAP, Colorado and Texas.** Both extend the federal base
above; the only per-state addition is the state-code dependency, already
captured by `state_dependency`, so `additional_inputs` is empty for both:

```json
{
  "program": "snap",
  "state": "co",
  "pe_name": "snap",
  "pe_entity": "spm_unit",
  "state_dependency": "CoStateCodeDependency",
  "extends": "snap",
  "additional_inputs": [],
  "pe_period_month": "01"
}
```

(TX's config is identical except `"state": "tx"` and
`"state_dependency": "TxStateCodeDependency"` — see
`schemas/examples/tx_snap_config.json`. That near-total similarity is
honest: `CoSnap`/`TxSnap` really are that thin in Python.)

## Data layer schema (Phase 1)

Purpose: benefit amount tables — the part PolicyEngine does NOT compute for
in-kind programs (Medicaid, WIC). Not needed at all for pure cash pass-through
programs like SNAP/TANF.

| Field | Type | Notes |
|---|---|---|
| `program` | string | |
| `state` | string | |
| `category_amounts` | object | Maps a category key (e.g. `ADULT`, `DISABLED`) to a monthly dollar amount. No fixed key enum — Medicaid's 11-key taxonomy and WIC's 6-key one both validate against the same schema. |
| `source` | object: `{status: "cited"\|"unsourced"\|"guessed", citation: string\|null}` | **Required, not optional** — see `DECISIONS.md` D-005. Structured, not free text, specifically because a free-text `source` string is trivially satisfiable by writing anything into it — a real citation and a confident-sounding fabrication look the same to the schema. `status` is machine-checkable; the schema itself enforces `citation` is non-null (and non-empty) iff `status` is `"cited"`. |

**Worked example — WIC, Colorado.** Real values transcribed from
`co/pe/member.py`'s `CoWic.wic_categories`. `status` is `"unsourced"`, not
`"guessed"` — no rationale of any kind was found in the code or across the
file's full git history (see `RISKS.md` Phase 1):

```json
{
  "program": "wic",
  "state": "co",
  "category_amounts": {
    "NONE": 0,
    "INFANT": 130,
    "CHILD": 79,
    "PREGNANT": 104,
    "POSTPARTUM": 88,
    "BREASTFEEDING": 121
  },
  "source": {
    "status": "unsourced",
    "citation": null
  }
}
```

**Worked example — WIC, North Carolina.** Real values from
`nc/pe/member.py`'s `NcWic.wic_categories` — flat `60` across every category.
`status` is `"guessed"`, not `"unsourced"`: git history shows these replaced
differentiated real-looking values (`INFANT: 130, CHILD: 26, PREGNANT: 47,
POSTPARTUM: 47, BREASTFEEDING: 52`) via a commit titled "Updated NC WIC
estimated saving value" — a deliberate estimate, just never tied to a
citation. See `RISKS.md` Phase 1 for the full history.

```json
{
  "program": "wic",
  "state": "nc",
  "category_amounts": {
    "NONE": 0,
    "INFANT": 60,
    "CHILD": 60,
    "PREGNANT": 60,
    "POSTPARTUM": 60,
    "BREASTFEEDING": 60
  },
  "source": {
    "status": "guessed",
    "citation": null
  }
}
```

**TX WIC has no data-layer file at all.** `TxWic` (`tx/pe/member.py`) has no
`wic_categories` dict — `member_value` defers entirely to PolicyEngine's own
computed value. TX WIC is Pattern 1 (pure pass-through), not Pattern 2, same
as SNAP/TANF — see `PATTERNS.md`. The schema correctly doesn't force a
data-layer file to exist for every state a program touches.

**`$id` caveat.** The `$id` above (`https://myfriendben.org/schemas/data-layer-v1.json`)
is not actually served at that URL — same as the existing precedent,
`test_case_schema.json`'s own `$id`, which isn't served anywhere in this repo
either. It's a namespacing convention, not a live endpoint. Item 4
("register schemas at a stable, fetchable location") still needs to make
this real; not solved by this schema.

## Validation requirements (CI)

- Every `state_dependency` and every entry in `base_inputs` must resolve to
  an actual class in `programs/programs/policyengine/calculators/dependencies/`
  — fail the build if not, don't let this be a silent runtime error.
- `source` field is required on any data-layer file, no exceptions — this is
  a deliberate quality bar above what currently exists upstream.
- Schema itself should be fetchable from a stable URL, matching how
  `test_case_schema.json` is published, so `program-researcher` or
  `/add-pe-program` could eventually validate against it too.

## Explicitly out of scope for this schema (revisit only with a DECISIONS.md entry)

- Fallback-chain logic (MA Medicaid→CHIP) — Python, Phase 7b
- Conditional gates (CHP's insurance-status check) — Python, Phase 6
- Population-segmented, cross-member lookups (TX Medicaid) — Python, Phase 7c

## Known limitation, not yet addressed

`category_amounts` as a flat key→number map fits category-lookup-shaped
programs (Medicaid's 11 keys, WIC's 6) but not every benefit-amount shape.
The RFC's own NSLP example used FPL-*threshold tiers* instead
(`tier_1_fpl`/`tier_2_fpl`/`tier_1_amount`/`tier_2_amount`) — a fundamentally
different structure, not category-keyed at all. This schema has not been
tested against that shape and likely doesn't generalize to it as-is. Not
fixing this now — flagging it for whoever hits it next (plausibly Phase 5's
Tier 2 work or beyond), so it doesn't get rediscovered as a surprise
mid-migration.
