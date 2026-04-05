# Spec A/B Test Results: VY-162 + VY-182

## Experiment

Two tickets, two spec formats, comparing agent outcomes.

### VY-182 (completed): Z-spec surfaced a real bug

The Z notation for VY-182 (pr_url missing from result ConfigMap) identified a cross-component invariant that no test checked:

```
pr_created ⇒ cm.data.pr_url ≠ ∅
```

This invariant spans the agent-runner (which creates PRs) and the controller (which reads the ConfigMap). Unit tests on either side couldn't catch it. The natural language ticket described the symptom ("missing pr_url") but the formal spec identified *why* it was a bug — it violated an invariant between two components that was never stated explicitly.

**Takeaway**: Formal specs are valuable for surfacing cross-component invariants. The notation itself isn't the point — it's the discipline of stating "if X happens in component A, then Y must be true in component B."

### VY-162 (both failed to complete): Spec format didn't matter

Both the natural language spec and the Z-style spec for VY-162 (SandboxConfig refactor) produced agents that:
- Spent similar budgets ($5.66 vs $5.04)
- Used similar turn counts (106 vs 105)
- Hit the same bottleneck: iterative grep/sed/build cycles updating test file struct literals
- Both actually completed the work and passed tests, but ran out of budget before creating PRs

The spec format made no measurable difference because the bottleneck was mechanical — updating `SandboxConfig{InfraRetryLimit: 3}` → `SandboxConfig{Retry: RetryConfig{InfraRetryLimit: 3}}` across dozens of test cases, one file at a time, rebuilding after each batch.

**Takeaway**: When the bottleneck is mechanical find-replace, spec quality doesn't matter. The task needed to be decomposed, not better specified.

## Token burn analysis

Both VY-162 agents spent tokens on the same pattern:
1. ~20% defining structs, UnmarshalJSON, DefaultSandboxConfig (the design work)
2. ~10% updating step files (straightforward field path changes)
3. ~70% iteratively fixing test files (grep for old pattern → sed replace → build → find more failures → repeat)

The test file updates are the long tail. Each build cycle on a Pi takes ~8 seconds, and the agent needed 15-20 cycles to converge. The agent should have done all sed replacements in one pass, but it didn't know the full scope upfront — it discovered test failures incrementally.

## Optimization opportunities

### 1. Task decomposition
Split VY-162 into:
- **Part A**: Define sub-structs + UnmarshalJSON in sandbox_template.go. Update testConfig() helper. Don't touch step files.
- **Part B**: Mechanical find-replace across all step files and tests. Could be a different agent or even a script.

### 2. Upfront scope discovery
The agent should grep for all usages before starting edits, not discover them through build failures. The spec could include: "There are N callsites across these files — here's the grep output."

### 3. Batch sed operations
Instead of fix-one-file-rebuild, the agent should build a complete sed script for all replacements and apply it once. This is a prompt engineering fix — instruct the agent to plan all replacements before executing.

### 4. Cross-component invariants as spec sections
Add a `## Invariants` section to the ticket standard (VY-118). Not full Z — just English statements like:
- "If the runner creates a PR, the result ConfigMap must contain the PR URL"
- "All 45 config fields must appear in exactly one sub-struct (no orphans)"
- "JSON deserialization of the old flat format must produce identical values in the nested format"

These catch the class of bugs that Z caught without the notation overhead.

## Experiment 3: step_running.go result parsing extraction

### Setup
Same invariants in both specs (PhaseFromResult, NilSafety, CostNonNegative, StatusFieldMapping). Isolating whether DSL encoding vs natural language affects invariant compliance.

### Results

| Metric | Natural (#108) | DSL (#109) |
|--------|---------------|------------|
| Cost | $0.68 | $0.87 |
| Diff lines | 341 | 302 |
| Test count | 27 | 25 |
| CostNonNegative | Enforced in code + tested | Not enforced, not tested |
| retriesRemaining type | `bool` (lossy) | `int` (preserves info) |
| oomKilled handling | Kept in signature | Marked `_` (correct) |

### Z invariant verification

**Natural**: All 4 invariants satisfied. CostNonNegative enforced defensively in `parseAgentResult()` with `if r.CostUSD < 0 { r.CostUSD = 0 }` and a dedicated test.

**DSL**: **CostNonNegative violated**. The invariant was stated in the spec (`INVARIANT CostNonNegative: forall r: AgentResult: r.CostUSD >= 0`) but the agent neither enforced it in code nor wrote a test for it. The DSL agent treated invariants as documentation/verification properties rather than implementation requirements.

**But the DSL agent made a better type decision**: `retriesRemaining int` preserves the actual retry count information, while the natural agent pre-computed to `bool` which loses it. The DSL's explicit `WHEN retriesRemaining > 0 => Continuable` case guided this — the `> 0` naturally suggests an int, while the natural language "retries remaining" is ambiguous between bool and int.

### Interpretation

Natural language agents are more likely to **enforce** invariants defensively in code (treating them as "make this true"). DSL agents are more likely to treat them as **verification properties** (something the system should satisfy, not something code should enforce). Both interpretations are valid — but without enforcement, the invariant is just a comment.

This suggests the DSL should distinguish between:
- `ENFORCE CostNonNegative: ...` (agent must add code to guarantee this)
- `VERIFY PhaseFromResult: ...` (agent must add tests to check this)

The distinction matters because it tells the agent whether the invariant is a defensive coding requirement or a test case.

## Overall conclusions

1. **Formal specs help with cross-component invariants** — the Z spec on VY-182 found the actual bug. Worth adopting as a lightweight `## Invariants` section.
2. **Formal specs don't help with mechanical tasks** — VY-162 burned the same tokens regardless of spec format. The bottleneck was task size, not spec quality.
3. **Task decomposition > spec quality** for large refactors. A well-specified 14-file task still fails if it exceeds the agent's budget.
4. **The 70% token burn on test updates** is the optimization target. Pre-computing the full replacement scope would cut this dramatically.
5. **DSL produces smaller diffs** — consistently 30-40% less code across experiments. The agent explores less when the spec is precise.
6. **Natural language agents enforce invariants more aggressively** — they treat invariants as defensive coding requirements and add both code guards and tests. DSL agents treat them as verification properties and sometimes skip enforcement.
7. **DSL agents make better type decisions** — the explicit case enumeration (`WHEN x > 0`) naturally guides the agent toward correct types (`int` not `bool`).
8. **The spec DSL needs ENFORCE vs VERIFY distinction** — telling the agent whether an invariant should be guaranteed by code or checked by tests.
9. **CLAUDE.md should be a spec compression codec** — shared invariant vocabulary, standard build commands, and conventions that every ticket can reference by name instead of repeating.

## Experiment 4: ValidateTicket (DSL v2 with ENFORCE/VERIFY/CHECK)

### Setup
First experiment using DSL v2 with the three invariant verbs. Adding ticket validation to the Linear poller. Z pre-analysis found the real bug: poller dispatches any `claude-agent` issue without checking description quality.

### Z pre-analysis bugs found
- The poller dispatches unvalidated tickets (anyone can bypass the dispatch skill by labelling directly)
- Cross-component invariant: poller and dispatch skill should enforce the same section requirements

### Results

| Metric | Natural (#110) | DSL v2 (#111) |
|--------|---------------|---------------|
| Cost | $0.95 | $0.92 |
| Test cases | 5 | 8 + HasSection tests |
| Separate file | No (inline in source.go) | Yes (validate.go) |
| NoFalsePositive | **VIOLATED** | ✅ Satisfied |

### Z post-analysis — bug found in natural language version

The natural agent combined the acceptance criteria header check and checklist check into a single AND: `if !hasACHeader || !hasChecklist`. This rejects tickets with `## Acceptance criteria\nMust handle errors` — valid tickets without `- [ ]` checkboxes.

The DSL agent got this right because the spec explicitly separated `HasAcceptanceCriteria` as its own function with its own WHEN cases, and `ValidateTicket` only calls `hasSection()` for the header check.

### DSL v2 ENFORCE/VERIFY/CHECK evaluation

The three verbs worked as intended:
- `ENFORCE StructureComplete` → both agents added validation code ✅
- `ENFORCE NoFalsePositive` → DSL agent satisfied it, natural agent violated it
- `VERIFY AcceptanceCriteriaTestable` → DSL agent wrote separate table-driven tests ✅
- `CHECK PollerIntegration` → both agents added FetchNewTasks integration + test updates ✅

The DSL agent's separate `validate.go` file suggests that explicit function specs (`FUNCTION ValidateTicket`, `FUNCTION HasAcceptanceCriteria`, `FUNCTION hasSection`) guide the agent toward better code organization — each function gets its own definition, so the agent naturally separates concerns.

### Key insight

**The DSL's explicit function separation prevented the bug.** By specifying `HasAcceptanceCriteria` and `hasSection` as distinct functions with distinct cases, the DSL forced the agent to implement them independently. The natural language agent merged them into one check and got the logic wrong.

This is the same pattern as the VY-182 Z analysis: **stating things separately that are separate prevents conflation**. Z does this with schemas, the DSL does it with FUNCTION blocks, and natural language encourages merging because prose is harder to keep decomposed.

## Z analysis: where it finds bugs

Z-style invariant analysis found real bugs at every stage of the lifecycle. This is the key finding of the experiments.

### Three checkpoints, three classes of bugs

**1. Pre-dispatch (analyzing the spec before the agent runs):**
The analyst reads the ticket spec and the target code, then states cross-component invariants the spec implies but doesn't state explicitly.

| Ticket | Invariant found | Bug |
|--------|----------------|-----|
| VY-182 | `pr_created ⇒ pr_url ≠ ∅` | Runner creates PR but doesn't report URL — controller skips entire review pipeline |
| VY-174 | `TotalCoverage: forall f in fields: override exists` | 10 config fields added in recent tickets have no env override |
| ValidateTicket | `ParseValidateConsistency` | ValidateTicket checks headers but not content — empty sections pass validation |

**2. Post-completion (analyzing generated code against invariants):**
The analyst reads the PR diff and checks each stated invariant.

| Experiment | Invariant | Bug |
|-----------|-----------|-----|
| VY-174 A/B | `TotalCoverage` | Both agents missed `[]string`/`map[string]string` — mechanism can't handle complex types |
| Result parsing | `CostNonNegative` | DSL agent stated the invariant but didn't enforce it in code |
| ValidateTicket | `NoFalsePositive` | Natural agent's AND logic rejects tickets with acceptance criteria header but no checkboxes |

**3. Cross-component (spanning boundaries between systems):**
The analyst identifies contracts between components that are implied but never stated.

| Boundary | Invariant | Status |
|----------|-----------|--------|
| Runner → Controller | Result JSON must contain pr_url when PR created | Was violated (VY-182) |
| Poller → Dispatch skill | Both must enforce same section requirements | Currently violated (poller doesn't validate) |
| ValidateTicket → ParseDescription | `RoundTrip`: if validation passes, parsing must extract content | Potentially violated (headers without content) |

### Pattern: invariants are most valuable at boundaries

Unit tests cover individual functions. Integration tests cover happy paths. **Invariants cover the contracts between components** — the assumptions one component makes about another's output. These are exactly the bugs that agents introduce silently, because agents work on one component at a time without awareness of the contract.

### At scale: invariant libraries

The insight for scaling is that invariants are **named, reusable contracts**. Common patterns emerge:

```
BackwardCompat(oldAPI, newAPI)     — wire format doesn't change
TotalCoverage(fields, overrides)   — every field has a corresponding entry
NilSafety(function)                — nil inputs don't panic
BehaviorPreservation(old, new)     — same outputs for same inputs
RoundTrip(serialize, deserialize)  — roundtrip preserves data
NoFalsePositive(validator, input)  — valid inputs are never rejected
```

These become a vocabulary in CLAUDE.md that any ticket can reference by name. An analysis agent knows what each one means and can check it mechanically against the code.

### The workflow

```
1. Human files rough ticket
2. Spec agent drafts sections + identifies decisions (VY-119)
3. Z analysis agent states invariants (pre-dispatch)
   └─ finds bugs in the spec before any code is written
4. Human reviews invariants, answers decisions
5. Code agent executes
6. Z analysis agent verifies invariants against PR diff (post-completion)
   └─ finds bugs in the code that tests don't catch
7. Haiku reviews code quality
8. Merge queue merges (risk-gated)
```

Steps 3 and 6 are new — cheap haiku runs (~$0.10 each) that catch cross-component bugs before and after execution. VY-186 tracks implementation.
