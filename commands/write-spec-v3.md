Write a dispatchable specification for a code change. Takes a rough task description and produces a spec an agent can execute.

**This is write-spec v3.** It is identical to v2 except the concrete FM-2 example has been removed from step 3 to enable uncontaminated evaluation on the task that motivated that example. The alternative-diagnosis mechanism is unchanged.

## Core principle

A good spec is **locked down where it matters and silent where it doesn't**. It names the exact files, line numbers, and existing patterns. It enumerates every site the change touches. It says what is NOT allowed to change. And it leaves everything else alone.

The failure mode v2 is built around is **anchored reasoning**: inheriting the framing of the rough task without checking whether the fix actually lives where the task suggests. v1 let this through. v2 does not.

## Steps

### 1. Read the rough task literally

Extract the stated symptom, the suggested cause (if any), and the user's proposed fix site (if any). Write these down verbatim.

**Do not start exploring yet.** You need the rough task's framing pinned so you can challenge it in step 3.

### 2. Explore the codebase

Use Glob, Grep, and Read to find:

- The functions or types named in the rough task (if any)
- The files that would change under the rough task's framing
- Related code: callers, callees, tests, documentation
- Configuration driven by the affected code

Map the call graph outward from the rough task's anchor — both upward (who calls this?) and downward (what does this call?) — to **depth 2 minimum**. Not doing this is the #1 source of wrong-layer specs.

### 3. Alternative-diagnosis pass (mandatory)

**Before writing any spec, state in one sentence**: where does the fix live and why there?

Then state **at least one plausible alternative fix site**, and argue in one paragraph why it's worse than the chosen site. Compare concretely: what does the alternative require, what does it protect against that the chosen site doesn't, why is your choice better.

**If you cannot articulate an alternative**, that is a signal you have inherited the rough task's anchor without questioning it. Stop writing the spec. Return to step 2 and actively search for a second plausible fix site. Do not proceed until you have an alternative to compare against.

This step exists because invariant-guided reasoning at the wrong layer still produces confident, well-structured, wrong specs. The alternative-diagnosis pass is the mechanical counter to motivated reasoning.

The example that originally appeared here has been removed to enable uncontaminated evaluation.

### 4. Check scope and decompose if needed

Measure the blast radius:

- Count files to modify (`wc -l` each)
- Count functions that change
- Count tests that need updating
- Estimate total lines changed

**Decompose when any of**:
- >5 files touched
- >300 lines of production code changing
- >20 tests need updating
- The change has a safe "wrap then refactor" split (introduce new abstraction without changing behavior, then migrate callers in a second task)

Decomposition pattern for refactors:
- **Task 1**: Introduce the new abstraction alongside the old code. All tests pass unchanged. Zero behavior change. Proves the abstraction works.
- **Task 2**: Migrate callers to the new abstraction. Update tests. Delete old code. Apply full invariant/enumeration treatment here.

Each subtask gets its own spec. Subtask dependencies are explicit (task 2 blocked by task 1).

If the task can't be decomposed, warn the user. Large monolithic tasks have a high failure rate.

### 5. Consult tier-2 codebase invariants

Tier-2 invariants (the ones in `INVARIANTS.md` if the repo has one) describe couplings that exist regardless of what you're changing. They are declared as **abstractions** with matchers, not as concrete pairs.

If the repo has an `INVARIANTS.md` with abstraction-layer entries (per VY-241), run the DAG walker against the files you plan to modify and list every active abstract invariant. Reference them by name in the spec.

If the repo has no `INVARIANTS.md`, ask yourself the enumeration-based questions anyway:

- Is there a language manifest paired with a container? (`BuildToolchainDeclarationParity` — bumping Go/Python/Node in go.mod/pyproject.toml/package.json requires a matching Dockerfile bump. Missing this was the v0.3.20 incident.)
- Is there an interface with multiple implementations? (`BackendParity` — every change to one impl needs the same change to the others, or an explicit justification why not.)
- Is there a struct serialized across a boundary? (`ProducerConsumerFieldParity` — every field added on the producer side needs a matching field on every typed consumer, or documentation that the consumer is intentionally shell-only.)
- Is there a field whose value flows through a multi-stage pipeline? (`FieldPropagation` — trace from producer → serialization → transport → deserialization → consumer. Do NOT stop at the JSON boundary; see step 7.)

### 6. Identify ticket-specific invariants (tier-3)

For this change specifically, what cross-component contracts could a naive refactor break silently?

Ask:

- What does component A assume about component B's output that isn't enforced by the type system?
- Could a refactor break this assumption without failing a test?
- Is there a property that must hold on every path, not just the happy one?

Name these using short composable names (`BehaviorPreservation(oldFunc, newFunc)`, `DataExposure(credentialSource, persistedResource)`) and list them in the spec's Invariants section.

**Do not redefine tier-1 universal invariants** (PathSymmetry, NilSafety, AnnotationConsistency). They're checked on every PR. Only list tier-3 invariants unique to this change.

### 7. Enumerate exhaustively (the backbone of the spec)

The bug is always in case N when only N-1 are handled. Enumerations force exhaustive coverage.

For every modified function, enumerate:

- **Exit paths × resources**: how many ways can this function return? What resources (processes, files, locks, connections, timers) does it acquire? Every exit path must handle every resource.
- **Enum values × switch cases**: does the function switch on a type/enum? List every value. What happens when a new value is added later?
- **Callers × new behavior**: is the function signature or contract changing? List every caller. Do they all handle the new behavior?
- **Config fields × coverage**: are new fields added to a struct? List every place that struct is read.
- **Error types × handling**: what errors can this function produce? Is each one returned, logged, or silently swallowed?
- **State mutations × failure paths**: if step A succeeds and step B fails, is the state left consistent?
- **Consumers** (new in v2, lesson from PR #155): if you add a field to a serialized struct, trace **every** consumer of that struct, not just the producer. Go-side struct, shell script, dashboard, metrics exporter, tests — each is a potential silent drop site. Stopping the enumeration at the JSON boundary is the failure mode.

### 8. Select design patterns that enforce constraints structurally

For each invariant or enumeration, ask: **is there a pattern that makes this hold by construction?** If yes, prescribe the pattern — the agent should implement the pattern, not a one-off fix.

Common mappings:

| Constraint | Pattern | Why |
|---|---|---|
| PathSymmetry (cleanup on all paths) | `defer` or extracted `cleanup()` helper | Enforced by construction — can't forget a path |
| AtomicGuard (side effect + annotation) | Extract `guardedEffect()` with retry-on-conflict | One call site, atomicity guaranteed |
| TotalCoverage (all fields handled) | Reflection-based test | Catches drift automatically |
| BackendParity (all impls agree) | Shared contract test suite | Divergence caught at test time |
| FieldPropagation (producer → consumer) | Shared type / code-gen'd mirror | Can't drift — same definition compiles both sides |
| BehaviorPreservation (refactor parity) | Table-driven test with before/after | Regression caught mechanically |

Check if the codebase already has an instance of the pattern. If yes, reference it ("follow the pattern in `step_review_dispatch.go`"). If no, introduce it explicitly in the spec.

### 9. Write the spec

Use this skeleton. Fill every section. Empty sections are a smell.

```markdown
## Problem

One paragraph. Symptom + why it matters + where the fix lives. Anchor this on your step-3 conclusion, not the rough task's framing.

## Alternative-diagnosis summary

One sentence: the alternative fix site you considered in step 3, and why you rejected it. This is scratchpad, not shipped spec content, but include it inline so the reviewer can see you questioned the anchor.

## Files to modify

- `path/to/file.ext:line` — what changes and why. List every file with its specific change site (line or function name).

## Context files

- `path/to/related.ext:line` — why the agent should read this. Include all callers, callees, and adjacent code the agent needs to understand the change in context.

## Acceptance criteria

- [ ] Testable criterion 1 (verifiable by running tests or reading code)
- [ ] Testable criterion 2
- [ ] ...

## Build/test

Exact commands the agent should run. From the correct working directory. Not "run the tests" — the literal command.

## Invariants (tier-3 only)

- Named invariant instantiations from step 6. Do NOT list tier-1 universal invariants.

## Enumerations (from step 7)

For each enumeration type that applies:

**Exit paths in `functionName`**:
- Path 1: ...
- Path 2: ...
- Path N: ...

**Callers of `changedFunction`**:
- `caller1.go:L` — current behavior, needs update to X
- `caller2.go:L` — current behavior, unchanged

**Consumers of `modifiedStruct`**:
- Producer: `path/to/producer.go` — already handles
- JSON boundary: `path/to/serializer.go` — already handles
- Consumer (Go): `path/to/consumer.go` — needs update
- Consumer (shell): `scripts/parse.sh` — already handles via jq
- Consumer (dashboard): none — field is intentionally shell-only

... etc

## Suggested approach

Concrete pseudocode for the mechanical steps, NOT hand-waved. If the fix is "walk the MRO", write:

```python
for cls in obj.__mro__:
    marks.extend(cls.__dict__.get("pytestmark", []))
```

not "walk the MRO and collect marks". This matters for cheap models (lesson from the hailort floor finding: 1.5B coder models can pattern-match concepts but not translate them into loops).

## Tests

Specific test names + what they verify. For each invariant, at least one test that would fail if the invariant is violated.

When a postcondition can be expressed as a formula, prescribe it as the test implementation:

Bad:  `assert result == [8, 9, 9, 0, 0, 1]`  (hand-computed, error-prone)
Good: `assert listToNat(result) == listToNat(l1) + listToNat(l2)`  (algebraic, correct by construction)

Write a helper function that verifies the postcondition, then use it in every test case.

## DO NOT change

Specific files, functions, or behaviors that must not be touched. Not "don't break anything" — name the specific things.

## Configuration

Values that must come from config, not hardcoded. If none, say "None".

## Escalation conditions

Concrete "stop and ask" triggers the agent should honor:
- If the change requires touching >N files beyond the listed ones
- If a named function has moved or been renamed since the spec was written
- If a prerequisite assumption is not true (list the assumption)
```

### 10. Semantic self-check (before shipping the spec)

**Classify every concrete claim in the spec** as one of:

- **Verified**: I read the exact source that produces this value, or I ran code that exercises it
- **Derived**: I am inferring it from adjacent code and the chain is short (one or two reads)
- **Guessed**: I am extrapolating from general ecosystem knowledge

Any **Guessed** claim must either:
- Be verified by reading/running the producing code (promote to Verified), or
- Be rewritten as a hedged claim that cannot be locally wrong

This is the VY-211 light check. It catches wrong-literal-claim failures (FM-1) before they ship.

## Rules (hard constraints)

- **File paths must exist**. Verify with Glob before listing. If a file doesn't exist and needs to be created, use `## Files to create/modify`.
- **Acceptance criteria must be testable**. "Works correctly" is not testable. "Returns error for nil input" is.
- **Build/test must be exact commands**. Not "run the tests" — the actual command.
- **DO NOT change must be specific**. Name the specific files, functions, or behaviors.
- **Invariants are instantiated, not defined**. Use `PathSymmetry(funcName, resourceName)`, don't redefine PathSymmetry.
- **Enumerations force exhaustive coverage**. If the spec lists N cases, the agent can't silently skip one.
- **Postconditions are test implementations, not acceptance criteria**. If a postcondition is a formula, prescribe it as the test.
- **Suggested Approach includes pseudocode**, not summaries, for mechanical steps.

## Anti-patterns (warn the user if the rough task has any)

- **Mostly prose, no code structure** → needs more concrete specification
- **"Explore" or "investigate" language** → this is research, not a code task
- **No file paths** → agent will burn tokens exploring
- **Touches `.github/workflows/`** → agents can't push workflow files (GitHub PAT lacks workflow scope)
- **Single task touching 10+ files** → decompose into smaller tasks
- **"Make it configurable" without specifying where config lives** → specify the config layer
- **Suggests a fix site without explaining why** → force the alternative-diagnosis pass harder

## Output

Present the spec as a single markdown block the user can paste into a Linear ticket. Do not create files or tickets — just output the spec.

---

## What changed from v1 (for the maintainer, not the agent)

1. **Mandatory alternative-diagnosis pass** (step 3). v1 had no counter to FM-2 wrong-layer framing. Batch-02 matplotlib-20826 showed all three arms inherit the rough task's anchor; this step forces articulation of alternatives.
2. **Semantic self-check** (step 10). v1 had no FM-1 guard. Classify claims as Verified / Derived / Guessed.
3. **Consumer enumeration** (step 7). v1 enumerated producers. PR #155 showed producer-side enumeration stops too early — the real invariant is "trace every consumer". Lesson from CodeRabbit catching a Go-side consumer gap that invariant analysis missed.
4. **Tier-2 DAG query** (step 5). v1 referenced tier-2 invariants in the abstract. v2 queries them via the DAG walker (VY-241) and lists active abstractions before writing the spec.
5. **Pseudocode in Suggested Approach** (step 9, Suggested approach). v1 allowed hand-waved descriptions. The hailort floor finding showed 1.5B coder models can't translate "walk the MRO" into a loop. v2 requires literal pseudocode for mechanical steps.
6. **Self-contained**. v1 referenced universal invariant libraries and other skills. v2 is standalone — an agent loading v2 needs no other files to produce a good spec.
