# PATTERNS

Reference material only — read when a task touches calculator logic. Not
session-start reading.

## The five calculator patterns

RFC 0003 documented four; the fifth (population-segmented) was identified by
working through Texas's actual Medicaid implementation, which fits none of
the original four.

### 1. Pure pass-through
PolicyEngine determines both eligibility AND the dollar value. Calculator
does nothing but wire up inputs/outputs — no logic at all.

**Confirmed examples (verified in code, all 6 states, zero exceptions):**
SNAP, TANF.

**Also mostly pure (verify per-state before assuming):** EITC, CTC, ACA
subsidies, SSI (WA/TX), MSP (TX/IL), Head Start (TX/MA).

**Also confirmed: TX WIC.** Unlike the other four PolicyEngine-backed WIC
states, `TxWic` (`tx/pe/member.py`) has no `wic_categories` dict at
all — `member_value` returns PolicyEngine's computed value directly. Found
while building Phase 1's data-layer schema: WIC is not uniformly Pattern 2
across states, the same lesson Medicaid already taught (see below) applies
here too — check per-state, don't assume from one state's shape.

**Warning:** "Pure" was established as a *pattern*, not proven per state. WA's
SNAP has a BBCE-driven net-income-test waiver noted in its own `spec.md` —
check every state's existing `pe/spm.py` subclass for hidden overrides before
assuming a program is pure pass-through in a state you haven't checked yet.

### 2. Eligibility + Fixed Amount
PolicyEngine determines eligibility; calculator returns a hardcoded amount
from the data layer, sometimes gated by an additional condition.

**Examples:** CHP (CO) — but note CHP's actual logic is `if chp_eligible AND
has_insurance_types(("none",))`, i.e. eligibility + a *condition* + a fixed
amount, not just eligibility + fixed amount. EveryDayEats (CO). IL LIHEAP —
this is the RFC's own worked code example.

**Also: WIC in CO, IL, NC, MA.** PolicyEngine determines WIC eligibility;
each state's calculator returns a hardcoded amount from a `wic_categories`
dict keyed the same way Medicaid's `medicaid_categories` is (a 6-key
taxonomy: NONE/INFANT/CHILD/PREGNANT/POSTPARTUM/BREASTFEEDING, vs.
Medicaid's 11). Confirmed while building Phase 1's data-layer schema — see
`SCHEMA.md` for the worked CO/NC examples and `RISKS.md` Phase 1 for a
citation-quality finding independent of the pattern itself (CO/NC uncited or
estimated; MA/IL's numbers may just be uncredited copies of CO's).

### 3. Fallback Chain
Try one PE variable's category value; if it's zero/absent, fall through to a
second PE variable's category value.

**Example:** MA MassHealth. Its calculator merges `Medicaid.pe_outputs` AND
`Chip.pe_outputs` into one calculator, tries the Medicaid category value
first, and falls through to a separate `chip_categories` dict keyed on a
different PolicyEngine variable (`ChipCategory`) if Medicaid returns 0.
**This is not yet built in any phase before 7b — genuinely new logic.**

### 4. Conditional Override
Use the PE value only if some other condition holds.
**Example:** CHP again (see pattern 2 — it's actually patterns 2+4 combined).

### 5. Population-Segmented (not in the original RFC — identified via TX)
Program is defined as a *set* of (PE variable, member-level gating condition)
pairs rather than one calculator with a category lookup. No shared category
table. May require cross-member lookups (checking a *different* household
member's eligibility, not just the current member's).

**Example:** TX Medicaid. Four separate calculator classes —
`TxMedicaidForChildren`, `TxMedicaidForPregnantWomen`,
`TxMedicaidForParentsAndCaretakers`, `TxEmergencyMedicaid` — each with its own
age/insurance-status gate, and the caretaker one requires checking whether a
*child* in the household independently qualifies for Medicaid. None of them
use a `medicaid_categories` table; they pass through PolicyEngine's raw
output value directly (`self.get_member_variable(member.id)`).

**This is the highest-risk, least-designed pattern in the project. See
Phase 7c and D-004.**

---

## PolicyEngine entity → calculator base class mapping

(From MyFriendBen's own `/check-pe-support` skill documentation — reuse this
mapping, don't reinvent it.)

| PE entity | Base class |
|---|---|
| `spm_unit` | `PolicyEngineSpmCalulator` |
| `tax_unit` | `PolicyEngineTaxUnitCalulator` |
| `person` | `PolicyEngineMembersCalculator` |
| `household` / `family` / `marital_unit` | `PolicyEngineCalulator` (no dedicated subclass) |

Pattern-1 programs cover `spm_unit` (SNAP/TANF) and will need Tier-2
migrations to prove the client abstraction also works cleanly for `tax_unit`
(EITC/CTC) and `person` (SSI/Head Start) before assuming `PolicyEngineClient`
generalizes across all three.

---

## Medicaid's real uniformity, precisely

Two layers, different uniformity:

**Eligibility layer — genuinely uniform.** All 6 states inherit the same
federal `Medicaid` class, same two-pathway logic (ACA expansion at 138% FPL
for adults under 65; aged/disabled pathway at state-specific FPL thresholds),
same 11-key category taxonomy (NONE/ADULT/INFANT/YOUNG_CHILD/OLDER_CHILD/
PREGNANT/YOUNG_ADULT/PARENT/SSI_RECIPIENT/AGED/DISABLED). PolicyEngine
already sourced the state-specific FPL variation — no new state research
needed for eligibility.

**Value-assignment layer — NOT uniform in shape:**
- CO, IL, NC, MA's base case: Pattern 1's data-swap works (~4 of 6 states)
- MA: Pattern 3 (fallback chain) — see above
- TX: Pattern 5 (population-segmented) — see above

**Value-table sourcing (separate from PE eligibility entirely):**
WA cites KFF ("Medicaid Spending Per Full-Benefit Enrollee"); KS cites the
Kansas Health Institute; CO/MA/IL/NC have **no source citation at all** in
the existing code. KFF publishes this as one standardized, cross-state
dataset (not 50 bespoke state sources) — same shape of problem MyFriendBen
already solved once via `HudIncomeClient` replacing a manual per-state
Google Sheet. Building a `KffMedicaidClient` on that precedent is Phase 7's
recommended approach, not per-state manual research.
