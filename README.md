# Invariant Analysis

Lightweight cross-component bug detection for code changes. Uses named invariants and cheap verification passes to catch bugs that unit tests miss.

## The problem

Code changes that pass tests can still violate unstated contracts between components. A function produces output that another function consumes — but neither side tests the contract between them. These cross-component bugs are invisible to unit tests because each component works correctly in isolation.

Examples:
- A validator checks that section headers exist, a parser checks that section content exists — a document with empty sections passes validation but produces empty parsed fields
- 10 config fields added over time never get environment variable overrides because no invariant requires coverage
- A producer changes its output format, but the consumer's expectations are only checked at runtime

## The approach

### 1. Invariant discovery

Before making changes, analyze the target code to identify cross-component invariants the task implies but doesn't state. Focus on boundaries between components — places where one part assumes something about another.

**Template**: `prompts/invariant-discover.md`

### 2. Spec authoring: ENFORCE/VERIFY/CHECK

State invariants explicitly with verbs that say what's expected:
- `ENFORCE` — add defensive code (guard clauses, validation)
- `VERIFY` — add tests
- `CHECK` — both

**Format**: `SPEC-DSL.md`

### 3. Invariant verification

After changes are complete, read the diff and check each stated invariant. Reports SATISFIED or VIOLATED with concrete counterexamples.

**Template**: `prompts/invariant-verify.md`

## Experimental results

Tested across 4 A/B experiments comparing natural language specs vs DSL specs with invariant analysis at each stage.

### Bug detection

| Stage | Bugs found | Examples |
|-------|-----------|----------|
| Pre-change discovery | 6 | Missing env overrides, empty section validation gap, cross-component contracts |
| Post-change verification | 5 | TotalCoverage violations, NoFalsePositive logic errors |
| Cross-component | 4 | Producer/consumer format mismatch, validator/parser consistency |

### DSL vs natural language

| Metric | Natural language + invariants | DSL + invariants |
|--------|------------------------------|-----------------|
| Change size | Larger diffs (30-40% more) | Smaller, more focused |
| Invariant compliance | Better at ENFORCE (defensive code) | Better at types and separation |
| Bug type | Conflation bugs (merging distinct checks) | Omission bugs (stating but not implementing) |

Key finding: **invariant discipline matters more than spec format**. Both formats benefit from explicit ENFORCE/VERIFY/CHECK.

## The invariant library

Named, reusable patterns that apply across codebases:

| Invariant | What it checks |
|-----------|---------------|
| `BackwardCompat` | Wire format doesn't change |
| `TotalCoverage` | Every item in set A has a match in set B |
| `NilSafety` | Nil inputs don't panic |
| `BehaviorPreservation` | Refactoring doesn't change output |
| `RoundTrip` | Encode then decode preserves data |
| `NoFalsePositive` | Valid inputs are never rejected |
| `ParseValidateConsistency` | Parser and validator agree |
| `IdempotentOperation` | Running twice = running once |
| `TypePreservation` | Type information isn't lost in conversion |

See `prompts/invariant-library.md` for full definitions with check methods.

## Installation

### As a Claude Code plugin

```bash
/plugin marketplace add maugt/invariant-analysis
/plugin install invariant-analysis@invariant-analysis
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

### With any LLM

The prompt templates in `prompts/` work with any model that can read code diffs. Use them directly or adapt them to your workflow.

## Usage

- `/invariant-discover src/foo.go src/bar.go` — discover cross-component invariants in code
- `/invariant-verify 42` — verify invariants against PR #42
- `/design-patterns src/` — analyze code for design patterns

## Files

```
.claude-plugin/
  plugin.json, marketplace.json     # Claude Code plugin manifests
commands/                           # Slash commands
  invariant-discover.md
  invariant-verify.md
  design-patterns.md
prompts/                            # Raw prompt templates
  invariant-discover.md
  invariant-verify.md
  invariant-library.md
examples/                           # Worked examples
data/                               # Experimental data
SPEC-DSL.md                         # Spec format documentation
```

## Related work

- **SelfSpec** — [Self-Spec: Model-Authored Specifications for Reliable LLM Code Generation](https://openreview.net/forum?id=6pr7BUGkLp). SelfSpec has the LLM design its own specification format, then generate code that matches it. Our `/exhaustive-verify` skill was inspired by SelfSpec's finding that forced exhaustive reasoning (tracing every exit path) catches bugs that targeted invariant checks miss — specifically resource cleanup asymmetries like missing `cmd.Wait()` after `Process.Kill()`. SelfSpec operates at the single-function level; our invariant analysis operates at cross-component boundaries. Together they cover the full spectrum.

- **Verina** — [Verina: Benchmarking Verifiable Code Generation](https://openreview.net/forum?id=0A4Uf88pog) ([github](https://github.com/sunblaze-ucb/verina)). Verina separates code, formal specification (preconditions/postconditions), and proof as distinct outputs. Our key takeaway: postconditions should be prescribed as test implementations, not acceptance criteria. When agents hand-compute expected test values, they get them wrong (observed in both goal-spec and transformation-spec variants). When they verify postconditions algebraically (`listToNat(result) == listToNat(l1) + listToNat(l2)`), tests are correct by construction. This is Verina's insight applied to informal specs.

- **Design by Contract** (Bertrand Meyer, Eiffel, 1986). Our invariant library (ENFORCE/VERIFY/CHECK) is design by contract for LLM agents in weakly-typed languages. In strongly-typed languages (Haskell, Rust), most of our intra-function invariants are enforced at compile time. Our unique value is architectural constraints spanning language/deployment boundaries that no type system reaches.

## License

Apache 2.0. See [LICENSE](LICENSE).
