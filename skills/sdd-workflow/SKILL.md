---
name: sdd-workflow
description: Spec-Driven Development workflow. Understand existing code, write the spec, generate test cases from spec scenarios, then drive implementation through TDD. Composes code-explorer → planner → tdd-guide → code-reviewer with two human gates. Use when building features, fixing ambiguous bugs, or refactoring with behavioral changes.
origin: community
tags:
  - workflow
  - methodology
  - sdd
  - tdd
  - spec
version: 2.0.0
---

# Spec-Driven Development (SDD) Workflow

A gated, spec-first methodology. The specification is the source of truth — tests derive from spec scenarios, code is judged by spec alignment, and changes always start with spec updates.

> **SDD = ANALYZE → PROPOSE → TESTGEN → APPLY → VERIFY → ARCHIVE → RETRO**
>
> TDD puts tests before code. SDD puts the spec before tests. Each layer constrains the next. Retro feeds lessons back into the process.

## When to Activate

- Building a new feature whose behavior is not yet fully specified
- Fixing a bug where the correct behavior is ambiguous
- Refactoring code that changes observable behavior
- Designing a new API, protocol, or data model
- Any task where "what should this do?" is a non-trivial question
- The user says "spec", "spec-driven", or asks for a spec before code

## When NOT to Activate

- Typo fixes, formatting, or purely cosmetic changes
- Trivial one-liners where the behavior is self-evident
- Tasks already covered by a narrower skill (e.g., `tdd-workflow` for well-specified bugs)

## The SDD Pipeline

```
REQUEST
  │
  ▼
[0.Classify] ── size the task
  │
  ▼
[1.Analyze] ─── understand existing code
  │
  ▼
[2.Propose] ─── write the spec (delta)
  │
  ▼
⛔ GATE 1: Spec Approval ⛔
  │
  ▼
[3.TestGen] ─── spec scenarios → executable test cases
  │
  ▼
[4.Apply] ───── TDD red-green-refactor
  │
  ▼
[5.Verify] ──── spec alignment + test case verification
  │
  ▼
⛔ GATE 2: Archive Approval ⛔
  │
  ▼
[6.Archive] ─── preserve knowledge
  │
  ▼
[7.Retro] ───── periodic review → improve process
```

Everything between the two gates flows without stopping. The gates are mandatory pauses for human approval.

## Phase -1 — Classify Size (Right-Sizing)

Ceremony scales to spec complexity. Score on three signals, take the **highest** tier any signal reaches:

| Tier | Spec Scope | Behavioral Surface | Phases |
|------|-----------|-------------------|--------|
| **trivial** | 1-2 sentences; the correct behavior is obvious | 1 file, 1 function | 4 → 5 → 6 |
| **small** | 2-5 paragraphs; one component or endpoint | 1-3 files, no new contract | (1 light) → 2 → 4 → 5 → 6 |
| **standard** | Structured spec with sections; new module or API | 2-8 files, maybe a new internal contract | 1 → 2 → 3 → 4 → 5 → 6 → 7 |
| **large** | Full design doc; cross-cutting or new public API | many files, new external dep or public contract | 1 → 2 → 3 → (Scaffold) → 4 → 5 → 6 → 7 |

State the tier in one line. The user can override.

> **trivial** skips Analyze, Propose, TestGen, and Retro — just TDD, Verify, Archive.
> **small** skips TestGen — the spec scenarios ARE the test cases.
> **standard** and **large** run the full pipeline.

## The Phases

Each phase delegates to existing ECC agents — this skill composes, not reimplements.

### Phase 1 — Analyze (Understand Existing Code)

Before writing any spec, understand what already exists. A spec written without understanding the codebase is the primary cause of SDD failure.

**Delegate to**: `code-explorer` to trace existing behavior, call chains, and data models.

**What to understand**:
- **Call chain**: entry point → core path → external dependencies
- **Data model**: entities involved, fields, relationships, state enums
- **Impact surface**: what files would a change here affect, and why

**Output**: A structured analysis note covering the three dimensions above. Mark uncertain findings as "to-confirm". Read-only — no code changes in this phase.

### Phase 2 — Propose (Write the Spec)

The spec is a **contract**, not a design document. It says WHAT, not HOW. For new features, write a delta spec — describe only what changes, not the entire system state.

**Delegate to**: `planner` for structural thinking about scope and boundaries. `architect` for cross-cutting specs that touch multiple domains.

**Spec structure** (for standard and large tiers):

```markdown
## Behavior Specification: [Feature / Fix]

### 1. Input / Output Contract
- **Inputs**: Types, ranges, required vs optional, default values
- **Outputs**: Success shape, error shape, side effects
- **State changes**: What state is read, what state is written

### 2. Success Scenarios (happy path)
- GIVEN [precondition], WHEN [action], THEN [observable outcome]
- Number each scenario: S1, S2, ...

### 3. Error Scenarios
- GIVEN [precondition], WHEN [invalid action], THEN [error outcome]
- Number each: E1, E2, ...

### 4. Edge Cases & Boundaries
- Empty, null, zero, max, min, concurrent, timeout
- Boundary between "works" and "fails"

### 5. Non-Functional Constraints
- Performance: latency budget, throughput
- Security: who can call this, what must be checked
- Compatibility: what must not break
```

**Delta Spec principle**: For incremental changes, write only the spec for what's changing — not the entire module. The delta spec references existing behavior from Phase 1's analysis.

**Spec quality self-check** (before Gate 1):
- [ ] Every scenario is testable (GIVEN/WHEN/THEN is concrete)
- [ ] Error scenarios cover every failure mode mentioned
- [ ] Edge cases include emptiness, extremes, and concurrency
- [ ] Non-functional constraints have measurable thresholds
- [ ] The spec does NOT prescribe implementation (no "use a hashmap", "create a class X")
- [ ] Reused entities/services from Phase 1 analysis are explicitly referenced

### Scaffold Step (Large tier only, between Phase 2 and Phase 3)

Stand up the first end-to-end skeleton — one scenario wired through all layers — just enough to validate that the spec is buildable.

**Delegate to**: `tdd-guide` for the scaffold's tests.

**Gate**: If the scaffold reveals spec gaps, return to Phase 2 before continuing.

### Phase 3 — TestGen (Spec Scenarios → Executable Test Cases)

This is the bridge between behavioral spec and executable code. Each GIVEN-WHEN-THEN scenario is decomposed into concrete test cases across four dimensions:

| Dimension | What it covers | Example |
|-----------|---------------|---------|
| **Happy Path** | Normal input → expected output | `findByUserId_returnsThreeOrders()` |
| **Boundary** | Empty, extremes, pagination edges, zero, max | `findByUserId_emptyResult_forUnknownUser()` |
| **Permission / Exception** | Auth, validation, not-found, illegal input | `searchByOtherUser_returns403_forNormalUser()` |
| **Data Consistency** | Sort order, field completeness, related data integrity | `findByUserId_sortedByCreatedAtDesc()` |

**Rules**:
- Every spec scenario produces at least 1 Happy Path + 1 Boundary or Exception case
- Do not generate test cases for behavior not in the spec (no overflow)
- Each test case must be named with the pattern: `methodName_whenCondition_thenExpectedResult`

**Output format** — a test case inventory:

```markdown
## Test Cases: [change-id]

### Scenario S1: [description] (Happy Path)
| # | Test Method | Arrange | Act | Assert |
|---|------------|---------|-----|--------|
| 1 | searchOrders_byUserId_returnsMatchingOrders | userId=1, DB has 3 orders | GET /api/v1/orders/search?userId=1 | 200, total=3, desc order |

### Scenario E1: [description] (Permission)
| # | Test Method | Arrange | Act | Assert |
|---|------------|---------|-----|--------|
| 2 | searchOrders_crossUser_returns403 | userA logged in, querying userId=userB | GET /api/v1/orders/search?userId=userB | 403, ERROR_CODE |

### Boundary / Consistency
| # | Test Method | Arrange | Act | Assert |
|---|------------|---------|-----|--------|
| 3 | searchOrders_pageExceedsTotal_returnsEmpty | DB has 3 orders | GET ...?page=10&size=20 | 200, total=3, content=[] |
```

**Delegate to**: manual decomposition (this skill's checklist). For standard/large tiers, `planner` can assist structuring the inventory.

**Gate**: The test case inventory is reviewed as part of Gate 1 (alongside the spec). Do not proceed to Phase 4 until both spec AND test cases are approved.

### Phase 4 — Apply (TDD Red-Green-Refactor)

Drive each test case through the TDD cycle. The test case inventory IS the implementation plan — work through it by risk priority, highest first.

```
For each test case in the inventory (ordered by risk):
  RED:   Write a test that fails because the behavior is not yet implemented
  GREEN: Write minimal code to make the test pass
  REFACTOR: Clean up while keeping tests green
```

**Delegate to**: `tdd-guide` agent (or `tdd-workflow` skill).

**Scenario matching**: During implementation, match domain patterns to specialized guidance:

| Domain Pattern | Key Constraints |
|---------------|-----------------|
| External HTTP call | Timeout 3s/5s, retry 3x exponential backoff, fallback |
| Message Queue (MQ) | Idempotent consumption (dedup key), retry → dead letter |
| Database query | Paginated, no N+1, indexed WHERE |
| File import/parse | Streaming read, batch insert (1000/batch), error row logging |
| Scheduled task | Distributed lock, paginated processing, alert on failure |
| Excel export | Streaming write, async for large datasets, sensitive field masking |

> The patterns above are reference examples — adapt to the project's actual tech stack and conventions.

**Git checkpoints** (per `tdd-workflow`):
- One commit per scenario group: `test: add reproducer for <group>` → `feat: implement <group>`
- Verify each checkpoint is on the current active branch

### Phase 5 — Verify (Spec Alignment Check)

Two-dimensional verification:

1. **Code quality** — standard review by `code-reviewer`
2. **Spec alignment** — trace every spec scenario through test cases to code

**Spec alignment checklist**:
- [ ] Every spec scenario (S1..Sn) has ≥1 test case in the inventory
- [ ] Every test case maps to a spec scenario (no untesteds)
- [ ] Every test case has a passing test in the codebase
- [ ] No code exists without a corresponding test case (no uncovered paths)
- [ ] All four TestGen dimensions are covered for each scenario (standard/large tiers)
- [ ] Coverage ≥ 80% per `rules/common/testing.md`

**Delegate to**: `code-reviewer` for code quality, `security-reviewer` if security triggers are touched.

### Phase 6 — Archive (Preserve Knowledge)

After Gate 2 approval, preserve what was learned. Knowledge that stays only in chat history evaporates after the session ends.

**What to capture**:
- **Session summary**: requirement, implementation decisions, problems encountered and solutions
- **What was reused**: existing entities, services, and patterns discovered in Phase 1
- **Delta trace**: which spec scenarios were implemented, which test cases verified them
- **Lessons learned**: what surprised you, what would you do differently

**Storage convention**: Write the summary as a dated note wherever the project keeps its development records. Link it to the spec document and the Phase 1 analysis so future sessions can trace the full chain.

### Phase 7 — Retro (Periodic Review)

Periodically (weekly, or after every 3-5 changes), review the accumulated session summaries to improve the process itself.

**What to analyze**:
- **Spec accuracy**: how often did the spec need amendment during implementation?
- **Rework rate**: what proportion of work was redone after review?
- **Pattern recurrence**: which mistakes repeat across sessions?

**Output**: A brief retro note listing the top 3 problems and concrete improvement actions — update a rule, sharpen a skill template, adjust the spec format. This is the evolution engine that makes each SDD cycle better than the last.

## The Two Gates

### GATE 1 — Spec + Test Case Approval (after Phase 2, or after Scaffold for large tier)

Present the spec AND the test case inventory to the user. Do NOT write any implementation code (other than scaffold for large tier) until the user approves both.

**Gate 1 checklist**:
- [ ] Spec covers all success paths the user expects
- [ ] Error handling strategy is agreed
- [ ] Edge cases and boundaries are explicit
- [ ] Non-functional constraints have measurable thresholds
- [ ] Test case inventory covers all four dimensions per scenario
- [ ] User has explicitly approved both spec and test cases (not just "looks good")

### GATE 2 — Archive Approval (after Phase 5)

Present the diff summary, test results, and spec alignment report. Do NOT proceed to archive until the user confirms.

**Gate 2 checklist**:
- [ ] All spec scenarios have passing tests
- [ ] Code review findings (CRITICAL/HIGH) are resolved
- [ ] Security review passed (if triggered)
- [ ] Coverage ≥ 80%
- [ ] Commits are conventional and scoped
- [ ] Session summary is ready for archiving

## Agent / Command Map

| Phase | Primary | Fallback / Escalation |
|-------|---------|----------------------|
| Analyze (Phase 1) | `code-explorer` | Trace call chains, data models, impact surface |
| Propose (Phase 2) | `planner` | `architect` for cross-cutting specs |
| Scaffold (large, Phase 2-3) | `tdd-guide` | `build-error-resolver` on build breaks |
| TestGen (Phase 3) | manual (this skill's checklist) | `planner` for structuring complex inventories |
| Apply / TDD (Phase 4) | `tdd-guide` (or `tdd-workflow` skill) | `build-error-resolver` on build breaks |
| Verify (Phase 5) | `code-reviewer` / `/code-review` | Language reviewer (`typescript-reviewer`, `python-reviewer`, …) |
| Security | `security-reviewer` | — |
| Archive (Phase 6) | manual (this skill's template) | `doc-updater` for updating project docs |
| Retro (Phase 7) | manual analysis | `planner` for structured retro format |

## Security-Review Trigger

Pull in `security-reviewer` when the spec or diff touches any of: authentication, authorization, user input, database queries, filesystem paths, external API calls, cryptography, or secrets. (Per `rules/common/security.md`.)

## Spec Change Protocol

When implementation reveals that the spec is wrong or incomplete:

```
1. STOP — don't silently diverge from the spec
2. Write down what you discovered (the gap)
3. Propose a spec amendment (one paragraph)
4. Get user approval for the amendment
5. UPDATE the spec document FIRST
6. UPDATE the test case inventory SECOND
7. THEN adjust implementation
```

**Rationale**: The spec is the contract. If the implementation drifts from the spec, both become unreliable. Spec-first, always.

## Traceability

Every line of code must be traceable to a test case, every test case to a spec scenario:

```
SPEC ──drives──→ TEST CASES ──constrain──→ TESTS ──verify──→ CODE
  ↑                                                           │
  │                                                           │
  └─────────────────── verified by ───────────────────────────┘
  │                                                           │
  └─────────── knowledge preserved in Archive ─────────────────┘
```

If code exists without a corresponding spec scenario, either:
- The spec is incomplete → add the scenario, regenerate test cases
- The code is unnecessary → remove it

If a spec scenario has no test cases, TestGen missed it — go back to Phase 3.

## Key Principles

1. **Understand before specifying** — writing a spec without understanding existing code is the primary cause of SDD failure
2. **Spec first, code second** — the spec is the primary artifact; code is its derivative
3. **Delta, not monolith** — write only the spec for what changes, not the entire system
4. **Scenarios drive test cases** — every spec scenario spawns test cases across four dimensions before any code is written
5. **No overflow, no omission** — test cases cover every spec scenario (no omission) and nothing beyond (no overflow)
6. **Human gates are mandatory** — spec approval and archive approval are hard stops, not suggestions
7. **Knowledge must survive the session** — every implementation session ends with a preserved summary; otherwise experience decays to zero over time
8. **Retro is the evolution engine** — periodic review of accumulated sessions is what makes each SDD cycle better than the last
9. **One change, one vertical slice** — a spec covers one coherent change, not an entire module
10. **Spec is a contract, not a design** — it says WHAT behavior, not HOW to implement; design decisions live in design notes, not in the spec

## Comparison: SDD vs TDD vs Ad-Hoc

| Dimension | Ad-Hoc | TDD | SDD |
|-----------|--------|-----|-----|
| What drives development | Intuition | Tests | Spec |
| When is "correct" defined | After code | Before code (as tests) | Before tests (as spec) |
| Test design | After implementation | During implementation | Before implementation (TestGen) |
| Review criteria | "Looks good" | Tests pass + coverage | Spec alignment + tests pass + coverage |
| Change protocol | Edit code directly | Edit tests, then code | Edit spec, update test cases, then code |
| Knowledge preservation | None | Test file is partial record | Archive session + retro review |
| Best for | Scripts, throwaways | Well-understood features | Ambiguous or high-stakes features |

## Anti-Patterns

### FAIL: Writing code before the spec is approved

```
User: "Add a login endpoint"
Claude: *writes 200 lines of auth code*
```

**Why it fails**: No agreement on what "login" means — email+password? OAuth? Session or JWT? Rate limiting? Lockout policy?

### PASS: Spec first, TestGen second, then code

```
User: "Add a login endpoint"
Claude: "Let me first analyze existing auth code, then write a behavior spec."
  [Phase 1: code-explorer finds existing User entity, AuthFilter, session config]
  [Phase 2: writes spec — email+password input, JWT output, 5-attempt lockout, rate limit]
  User: "Yes, but add OAuth as an alternative input"
Claude: [updates spec]
  [Phase 3 TestGen: decomposes into 7 test cases across Happy Path / Boundary / Permission]
  User: "Approved"
  [Phase 4: TDD implementation]
  [Phase 5: verifies all 7 test cases pass, coverage 92%]
  [Phase 6: archives session summary linking to spec and analysis]
```

### FAIL: Spec drift

Writing a spec, then ignoring it during implementation because "it's easier to just code it." The spec becomes dead documentation.

### FAIL: Over-specifying

Writing a 20-page spec for a 50-line utility function. Match the ceremony to the tier.

### FAIL: Skipping TestGen for standard/large tiers

Jumping from spec directly to TDD without decomposing scenarios into test cases. This leads to missed edge cases and incomplete coverage. TestGen is the quality gate BEFORE code.

## Quick-Start Example

```
User: "I need a function that merges two sorted lists"

Claude (SDD activated):
  TIER: trivial (1 function, 1 file, behavior is well-defined)

  ANALYZE (light): already two sorted arrays of comparable elements — no existing code to trace.

  SPEC:
  Input: two sorted arrays of comparable elements
  Output: one merged sorted array
  Edge cases: empty arrays, different lengths, duplicate values

  TESTGEN (light): spec scenarios ARE the test cases for trivial tier
    - Happy Path: [1,3,5] + [2,4,6] → [1,2,3,4,5,6]
    - Boundary: empty + [1,2] → [1,2]
    - Consistency: [1,1] + [2,2] → [1,1,2,2]

  → Gate 1: User confirms spec + test cases ✓

  APPLY (TDD):
    RED: write test → fail
    GREEN: implement merge → pass
    REFACTOR: clean up

  VERIFY: 3/3 tests pass, coverage 100%

  → Gate 2: Present diff, user confirms ✓

  ARCHIVE: session note — "merged two sorted lists, edge cases: empty arrays and duplicates"
  RETRO: skipped (trivial tier)

  COMMIT: feat: add sorted list merge function
```

## Verification (Before Gate 2)

- [ ] Size tier was stated and matched the work
- [ ] Phase 1 analysis identified existing entities/services/dependencies
- [ ] Every spec scenario has ≥1 test case in the inventory
- [ ] Test case inventory covers all four dimensions (standard/large tiers)
- [ ] Every test case maps to a spec scenario (no overflow)
- [ ] Every test case has a passing test in the codebase
- [ ] Gate 1 (spec + test case approval) was honored
- [ ] Gate 2 (archive approval) was honored
- [ ] `security-reviewer` ran iff a security trigger was touched
- [ ] Coverage ≥ 80% per `rules/common/testing.md`
- [ ] Commits follow the spec → test → feat pattern
- [ ] Spec document and test case inventory are committed alongside code
- [ ] Session summary is preserved (Phase 6)

## Model Tiering (Optional)

For users who can choose models per phase:

| Phase | Reasoning Priority | Why |
|-------|-------------------|-----|
| Analyze | Deep reasoning | Understanding complex code structures |
| Propose | Deep reasoning | High-quality spec as the source of truth |
| TestGen | Broad coverage | Exhaustive dimension scanning |
| Apply | Cost-efficient speed | High volume of code generation |
| Verify | Balanced | Understanding + pattern matching |
| Archive / Retro | Lightweight | Simple file operations and analysis |

Match the model to the phase's cognitive demand. Don't use the heaviest model for simple archiving.

## Related Skills

- `tdd-workflow` — test-driven development within each TestGen scenario group
- `orch-pipeline` — general-purpose orchestration for non-spec-driven work
- `security-review` — used in Phase 5 when security triggers fire
