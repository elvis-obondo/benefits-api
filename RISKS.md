# RISKS

Living document. Append newly discovered risks at the bottom of the relevant
phase section — do not just fix a problem silently and move on, log it here
so a future session (or a future contributor) doesn't rediscover it the hard
way.

## Cross-cutting risks

- **Fork drift.** `benefits-api` moves fast (~1,951 AI-assisted commits
  observed in a ~14-month sampled window during planning). A long-lived fork
  will conflict. Budget rebasing every 1-2 weeks, not once at the end. Track
  in `UPSTREAM_SYNC.md`.
- **Silence from upstream maintainers.** The RFC's own `discussion.md` was an
  empty template — 9 months with no recorded engagement as of when this
  project started. Could mean no objections, could mean no bandwidth. Don't
  wait indefinitely for a response before proceeding with Phase 0-4 work; do
  set an explicit walk-away checkpoint (Phase 9).
- **PolicyEngine credential scope creep.** Hosted API credentials require
  emailing hello@policyengine.org — don't request them prematurely, don't ask
  MyFriendBen to share theirs before you have working code to show. See D-003.

## Phase 0

- **Resolved:** Postgres dependency confirmed required. `benefits/settings.py`
  hardcodes `ENGINE: django.db.backends.postgresql` with no sqlite/throwaway
  fallback, and CI (`pr-validation.yaml`) runs a real `postgres:15` service
  container. `pytest.ini` uses `--reuse-db --nomigrations`. Local dev needs a
  persistent live Postgres server, not just a test-only DB spun up on demand.
- **CLAUDE.md's ruff/mypy claim was stale — corrected.** The actual upstream
  toolchain is `black` only (`Makefile`'s `format` target, `pyproject.toml`'s
  `[tool.black]`, `format.yaml` in CI). No `ruff` or `mypy` config or
  dependency exists anywhere in the repo. Fixed in `CLAUDE.md`'s inherited-
  rules section.
- **Local dev Python version deviates from the pin (3.13 instead of 3.10).**
  `.python-version` and CI's `setup-python-django/action.yml` both pin 3.10,
  but Homebrew's `python@3.10` bottle is broken on this machine (`ensurepip`
  fails — bottled binary expects a newer `libexpat` symbol than the system's
  `/usr/lib/libexpat.1.dylib` provides), and neither `pyenv` nor another 3.10
  install path was readily available. Decision: rebuild the venv on system
  Python 3.13 rather than chase the Homebrew/libexpat issue further. This
  surfaced real consequences, not just a version-number mismatch — several
  exactly-pinned packages in `requirements.txt` have no Python 3.13 wheel and
  fail to build from source:
  - `numpy==1.23.3` and `kiwisolver==1.4.4` were **skipped entirely** for
    local dev — nothing in the app's own code imports either directly; they
    read as stale transitive cruft from an old `pip freeze`, not real deps.
  - `pillow==10.3.0`, `grpcio==1.58.0`, `grpcio-status==1.48.2`,
    `google-cloud-translate==3.24.0`, `googleapis-common-protos==1.72.0`,
    `google-api-core==2.29.0`, `psycopg2==2.9.3`/`psycopg2-binary==2.9.3` were
    installed **unpinned (latest compatible)** instead of the exact pin —
    these back real functionality (`integrations/clients/google_translate.py`,
    the Postgres driver itself), so skipping wasn't an option. Local dev now
    runs newer versions of these than CI/production. Revisit if this ever
    produces a behavior mismatch; a proper `pyenv`-built 3.10 or a Docker dev
    container would remove the need for this workaround entirely.
- `Makefile` only defines a `format` target, not `test` — `ROADMAP.md`'s
  Phase 0 item 3 referenced `make test`, but it doesn't exist. Tests run
  directly via `pytest` (confirmed working) or `python manage.py test`.
- PolicyEngine's self-hosted Docker image (`ghcr.io/policyengine/policyengine-household-api:current`)
  reports `unhealthy` for its first ~2-3 minutes while it loads model data —
  `docker inspect --format '{{.State.Health.Status}}'` and hitting
  `/liveness_check` will both fail during that window. Not a real failure;
  just don't conclude the container is broken from early health-check noise.
- **Still open:** actually running `pytest`/`manage.py test` needs a `.env`
  (`SECRET_KEY`, `DB_NAME`/`DB_USER`/`DB_PASS`, `REDIS_URL`, `FRONTEND_DOMAIN`)
  plus a native local Postgres and Redis install, per the setup wiki
  (`Gary-Community-Ventures/benefits-api` wiki, "Get Started"). Deliberately
  scoped out of "set up the toolchain" (item 3) — treated as its own step,
  since it involves further decisions (e.g. `GOOGLE_APPLICATION_CREDENTIALS`
  is a real shared secret, unrelated to this project's RFC 0003 scope, and
  the dummy values needed for local Postgres/Redis need their own call).

## Phase 1

- Dependency-name-as-string in JSON is a real footgun — if a dependency class
  gets renamed in the Python codebase, JSON referencing the old name breaks
  silently at runtime with no compile-time signal. Mandate CI validation that
  every referenced name resolves to a real class (see `SCHEMA.md`).
- Don't let the config schema quietly grow conditional/branching fields to
  handle a specific program's logic — that's scope creep into an ad hoc rules
  DSL, which is a much bigger project than this one. Push logic to Python.
- The RFC's own open question about state-level mixins (e.g.
  `IlMedicaidFplIncomeCheckMixin`) is unresolved — don't let Phase 4+ start
  before this has an explicit answer in `DECISIONS.md`, or the schema will
  need a disruptive redesign mid-migration.
- **The Medicaid citation gap (Phase 7's "Fix the citation gap" item) is
  confirmed to extend to WIC as well, today, not just Medicaid.** Checked
  CO, IL, NC, MA, TX's `wic_categories`, including full git history via
  `git log --follow -p` (not just current code) for every state before
  labeling any of them:
  - **CO** (`co/pe/member.py`): real dollar values, zero citation anywhere
    in the file's rename history back to its oldest tracked commit
    (`c6c55d58`, "move calculators to state folders"). `status: "unsourced"`.
  - **NC** (`nc/pe/member.py`): currently flat `60` across every category,
    but this was NOT always the case and NOT a case of "nobody ever thought
    about it" — commit `895a4561` ("Updated NC WIC estimated saving value,"
    2025-05-01) deliberately replaced differentiated real-looking values
    (`INFANT: 130, CHILD: 26, PREGNANT: 47, POSTPARTUM: 47,
    BREASTFEEDING: 52`) with the current flat `60`. No citation or linked
    spec/PR body was found (the commit touched only that one file). This is
    a deliberate estimate, just uncited — `status: "guessed"`, distinct
    severity from CO's.
  - **TX** (`tx/pe/member.py`): has no `wic_categories` dict at all —
    `member_value` defers entirely to PolicyEngine's computed value. TX WIC
    is Pattern 1 (pure pass-through), not Pattern 2 — see `PATTERNS.md`. No
    data-layer file needed for TX WIC, same as SNAP/TANF.
  - MA and IL are a distinct, more serious finding — see the next entry,
    don't bucket them with CO/NC's "uncited/estimated" severity.
- **Separate, more serious finding: MA's and IL's live WIC dollar values may
  just be Colorado's numbers, not their own states' actual WIC package
  costs.** This is a real-world accuracy concern in the *existing upstream
  code*, independent of anything this refactor does — not merely "missing a
  citation."
  - **MA** (`ma/pe/member.py`, `MaWic`): the code comment
    `# NOTE: guesses based off Colorado` has been present since the
    class was first introduced (commit `341c0692`, "ma config") — an
    explicit admission, confirmed via full git history, not just current
    code.
    `status: "guessed"`.
  - **IL** (`il/pe/member.py`, `IlWic`): values are byte-for-byte identical
    to CO's (`130/79/104/88/121`) from the very first commit that introduced
    the class (`52c717af`, "IL: Add WIC Program") — no comment, no PR body
    beyond the title, nothing distinguishing it from an uncredited copy of
    CO's own uncited numbers. `status: "unsourced"` (no admission either
    way — worse in one sense than MA, which at least documents the guess).
  - **Worth raising to MyFriendBen directly, separately from the
    architecture RFC/PR conversation** — this is a good-faith accuracy
    finding about production benefit-value data that stands regardless of
    whether this refactor ever lands. Not yet drafted in
    `PENDING_COMMS.md` — same access blocker applies (see Phase 0).
- **`source` field design.** See `DECISIONS.md` D-005 — `source` is a
  structured `{status, citation}` object, not a free-text string, precisely
  because the findings above showed a free string is trivially satisfiable
  by anything, including an eventually-wrong-looking citation
  indistinguishable from a real one.
- **Likely latent bug: MA WIC never sends its state-code dependency to
  PolicyEngine.** Found while researching the config-layer schema's
  `base_inputs`-duplication problem. Every other state+program combination
  checked (CO/IL/NC/TX WIC; CO/IL/KS/MA/NC/TX/WA SNAP) does
  `[*FederalProgram.pe_inputs, <State>CodeDependency]` — the federal list
  plus exactly one state-code addition. `MaWic` (`ma/pe/member.py`) is the
  one exception: it doesn't override `pe_inputs` at all, so it inherits the
  federal `Wic.pe_inputs` (4 entries) unchanged, with **no**
  `MaStateCodeDependency` added. Not fixed here — this is an upstream Python
  behavior question (oversight vs. intentional), not a schema-design
  concern. Worth confirming with MyFriendBen directly, same as the MA/IL
  WIC-value-copying finding above.
- **`extends`/`additional_inputs` composition (config-layer schema).** See
  `DECISIONS.md` D-006 — chosen over a flat, fully-duplicated `base_inputs`
  list specifically because SNAP's 22-entry federal dependency list would
  otherwise be repeated 6 times, multiplying the dependency-name-as-string
  footgun above by the number of states sharing a program. The `extends`
  depth is capped at one level (state→federal only); this is a runtime/CI
  check, not something JSON Schema draft-07 can express on its own (see
  `SCHEMA.md`).

## Phase 2

- "Are we calling the PE API twice?" — the RFC itself flagged this as an open
  concern (new client + old `Sim` both hitting PolicyEngine during parallel
  verification). Must be resolved with a passing test, not just an assumption,
  before Phase 4 begins.
- `PolicyEngineClient`'s method signatures are the single riskiest early
  design decision — every later phase depends on them. Don't finalize until
  Phase 4 has exercised them against a real calculator.

## Phase 3

- JSON caching strategy (load at process start vs. read per-request) has a
  correctness implication, not just a performance one — if cached at startup,
  a JSON edit won't take effect until redeploy. Confirm this matches how
  `import_program_config` currently expects config changes to be deployed
  before assuming either approach is fine.
- Backward compatibility isn't optional per the (admittedly template/placeholder)
  answer in the RFC's own discussion doc ("maintain deprecated APIs for 2
  release cycles"). Design the factory so old and new calculators genuinely
  coexist, not as a hard cutover.

## Phase 4

- **"Identical output" needs a precise definition before running the
  comparison**, not after seeing a mismatch and rationalizing it away.
  Floating-point/rounding differences, timezone-dependent period
  calculations, and differing missing-dependency handling could all produce
  near-but-not-exact matches. Decide the tolerance threshold first.
- Don't declare "SNAP is pure pass-through" proven from one state (CO) alone
  — check every state's actual `pe/spm.py` for hidden overrides (WA's BBCE
  net-income-test waiver is a known example) before assuming uniformity holds
  across all six.
- Test data privacy: confirm "real screener submissions" for comparison work
  means sanctioned test/staging data, not production PII, before running
  anything.

## Phase 6

- CHP is not a clean single-pattern example — it's Pattern 2 (fixed amount)
  AND Pattern 4 (conditional override) combined (`chp_eligible AND
  has_insurance_types(("none",))`). Don't let a "simple" first override
  migration quietly under-design the base class for composability.

## Phase 7

- **7c (TX population-segmented) is the highest-risk step in the whole
  project.** Budget real design time — could plausibly take as long as
  Phases 1-6 combined. If it doesn't generalize cleanly, the acceptable
  fallback is documenting TX Medicaid as staying on the old architecture,
  not forcing a bad abstraction to cover it.
- KFF (or equivalent) data access method is unconfirmed — API vs.
  scrape-only. If scrape-only, check terms of use before building against it.
- KFF's "Enrollment Group" categories won't map 1:1 onto the existing 11-key
  taxonomy automatically — this mapping needs to be defined once, explicitly,
  not inferred.

## Phase 8

- A single giant PR is very likely to stall, especially given the 9-month
  silence on the original RFC discussion. Split into a sequence of small,
  independently mergeable PRs mirroring the phase structure.
- Check for the "three-question gate" pattern (from `AGENTS.md`) on anything
  touching partner-facing test files before it becomes a surprise blocker.
