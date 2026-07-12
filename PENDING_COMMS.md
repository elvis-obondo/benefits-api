# PENDING COMMS

Drafted external communications (PR descriptions, PR comments, issue text)
that are ready to post but blocked on access — e.g. not yet a collaborator on
a target repo. Append new entries at the top. When something actually gets
posted, update its status and add the live URL; don't delete the entry —
it's the record of what was actually said.

Template for each entry:

```
## YYYY-MM-DD — <target repo> — <short title>
**Status:** drafted, not yet posted / posted (see URL)
**Target:** <repo> — <PR/issue/comment, with link if it exists>
**Blocked by:** <why it can't be posted directly yet>

<the exact text to post>
```

---

## 2026-07-12 — MyFriendBen/rfc — Comment on PR #5 (RFC 0003 intent-to-build)

**Status:** drafted, not yet posted
**Target:** [MyFriendBen/rfc PR #5](https://github.com/MyFriendBen/rfc/pull/5) — plain comment
**Blocked by:** commenting/PR access on `MyFriendBen/rfc` requires org membership; not yet a collaborator

Hi — I'm working on a from-scratch prototype of this RFC in a fork of
`benefits-api`, aiming to validate the composition-based architecture
end-to-end before proposing it back upstream as a real PR sequence. Piloting
on SNAP/TANF first (pure pass-through in all 6 states, zero override-pattern
complexity) rather than Medicaid, specifically to avoid conflating "does
composition work" with "does it handle Medicaid's three different shapes" in
one pass. Plan is to work up through Tier 2 (EITC/CTC/ACA/SSI/MSP/Head Start)
and only then tackle Medicaid, split into its three real sub-shapes —
CO/IL/NC's clean category-dict pattern, MA's Medicaid→CHIP fallback chain,
and TX's population-segmented calculators, which don't fit cleanly into any
of the four override patterns above and may need a fifth pattern documented
for it.

Two questions before going further:
1. Does this design still hold as proposed, or has anything changed/been
   superseded since October?
2. On the two items already flagged as open in `discussion.md` (state-level
   mixin handling, and the double-PE-API-call concern during rollout) — any
   existing thinking, or are these genuinely still open? Plan is to answer
   both with working code from the prototype rather than guessing, but
   wanted to check before sinking time into it.

Will open small, phase-by-phase PRs against `benefits-api` as real code
exists, rather than one large drop.
