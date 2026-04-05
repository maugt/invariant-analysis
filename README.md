# Invariant Analysis for Agent-Generated Code

Lightweight cross-component bug detection for LLM agent PRs. Uses named invariants and cheap verification passes (~$0.10-0.20 per PR) to catch bugs that unit tests miss.

## The problem

LLM agents write code that passes tests but violates unstated contracts between components. A function produces output that another function consumes — but neither side tests the contract between them. The agent doesn't know the contract exists because it was never stated.

Examples from production:
- Agent-runner creates a PR but doesn't report the URL → controller skips the entire review pipeline
- Validator checks section headers exist → parser checks section content exists → a ticket with empty sections passes validation but produces empty parsed fields
- 10 config fields added over time never got env overrides because no invariant required it

## The approach

Three checkpoints, three prompt templates:

### 1. Pre-dispatch: invariant discovery ($0.10)
Before the agent runs, a cheap analysis pass reads the ticket spec and target code, then identifies cross-component invariants the spec implies but doesn't state.

**Template**: `prompts/invariant-discover.md`

### 2. Spec authoring: ENFORCE/VERIFY/CHECK
Invariants go in the ticket spec with explicit verbs:
- `ENFORCE` — agent must add defensive code (guard clauses, validation)
- `VERIFY` — agent must add tests
- `CHECK` — both

**Format**: `SPEC-DSL.md`

### 3. Post-completion: invariant verification ($0.10)
After the agent completes, a cheap verification pass reads the PR diff and checks each stated invariant. Reports SATISFIED or VIOLATED with counterexamples.

**Template**: `prompts/invariant-verify.md`

## Experimental results

Tested across 4 A/B experiments comparing natural language specs vs DSL specs, with Z-style invariant analysis at each stage.

### Bug detection rates

| Stage | Bugs found | Examples |
|-------|-----------|----------|
| Pre-dispatch | 6 | Missing env overrides, empty section validation gap, cross-component contracts |
| Post-completion | 5 | TotalCoverage violations, NoFalsePositive logic errors, CostNonNegative not enforced |
| Cross-component | 4 | Runner↔Controller result format, Validator↔Parser consistency |

### Cost per verification

| Operation | Model | Cost | Bugs found per run |
|-----------|-------|------|--------------------|
| Invariant discovery | haiku | $0.11 | 7-14 invariants identified |
| Invariant verification | haiku | $0.11 | 1-2 violations caught |
| Code agent (for comparison) | sonnet | $0.50-5.00 | N/A |

### DSL vs natural language

| Metric | Natural language + invariants | DSL + invariants |
|--------|------------------------------|-----------------|
| Code size | Larger diffs (30-40% more) | Smaller, more focused |
| Invariant compliance | Better at ENFORCE (defensive code) | Better at types and separation |
| Bug rate | Conflation bugs (merging distinct checks) | Omission bugs (stating but not implementing) |

Key finding: **invariant discipline matters more than spec format**. Both formats benefit from explicit ENFORCE/VERIFY/CHECK, but the DSL's function separation prevents conflation bugs while natural language's prose encourages defensive coding.

## Files

```
invariant-analysis/
├── .claude-plugin/
│   └── plugin.json                    # Claude Code plugin manifest
├── README.md                          # This file
├── SPEC-DSL.md                        # Spec format with ENFORCE/VERIFY/CHECK
├── commands/                          # Claude Code slash commands
│   ├── invariant-discover.md          # /invariant-discover — find invariants in code
│   ├── invariant-verify.md            # /invariant-verify — check invariants against a PR
│   └── design-patterns.md             # /design-patterns — analyze code for patterns
├── prompts/                           # Raw prompt templates (for non-Claude-Code use)
│   ├── invariant-discover.md
│   ├── invariant-verify.md
│   └── invariant-library.md
├── examples/
│   └── vy-182-z-spec.md               # Z notation example
└── data/
    └── experiment-results.md           # A/B test data
```

## Usage

### With Claude Code agent tasks

```yaml
# Pre-dispatch analysis
spec:
  taskType: review
  model: haiku
  maxBudgetUSD: "0.30"
  prompt: |
    Read prompts/invariant-discover.md.
    Analyze ticket VY-XXX against the code at <files>.
    Identify invariants and open decisions.

# Post-completion verification
spec:
  taskType: review
  model: haiku
  maxBudgetUSD: "0.20"
  prompt: |
    Read prompts/invariant-verify.md and prompts/invariant-library.md.
    Verify PR #NNN against these invariants:
    ENFORCE InvariantName: ...
    VERIFY InvariantName: ...
```

### With any LLM

The prompt templates work with any model that can read code diffs. Replace `gh pr diff` with your diff source.

## The invariant library

Common patterns that apply across codebases:

| Invariant | What it checks |
|-----------|---------------|
| `BackwardCompat` | Wire format doesn't change |
| `TotalCoverage` | Every item in set A has a match in set B |
| `NilSafety` | Nil inputs don't panic |
| `BehaviorPreservation` | Refactoring doesn't change output |
| `RoundTrip` | Encode→decode preserves data |
| `NoFalsePositive` | Valid inputs are never rejected |
| `ParseValidateConsistency` | Parser and validator agree |
| `IdempotentOperation` | Running twice = running once |
| `TypePreservation` | Type information isn't lost in conversion |

See `prompts/invariant-library.md` for full definitions with check methods.

## Installation

### As a Claude Code plugin

```bash
/plugin install invariant-analysis@maugt
```

Or add the marketplace to your settings and install:

```bash
/plugin marketplace add maugt/invariant-analysis
/plugin install invariant-analysis
```

### Manual install

Copy the commands into your Claude Code user commands:

```bash
cp commands/*.md ~/.claude/commands/
```

Or into a project's commands directory:

```bash
mkdir -p .claude/commands
cp commands/*.md .claude/commands/
```

### Usage

- `/invariant-discover src/foo.go src/bar.go` — discover cross-component invariants
- `/invariant-verify 42` — verify invariants against PR #42
- `/design-patterns src/` — analyze code for design patterns
