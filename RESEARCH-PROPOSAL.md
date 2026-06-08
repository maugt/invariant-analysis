# Research Proposal: Invariant-Guided Specification for LLM Code Agents

## Research question

Does explicitly stating cross-component invariants in task specifications improve the correctness of LLM-generated code, and can automated verification of these invariants catch defects that conventional review processes miss?

## Why this matters

LLM code agents produce output that passes unit tests but violates unstated contracts between components. Current quality gates (tests, code review) check individual components, not the assumptions they make about each other. This is the class of defect agents uniquely introduce — they work on one file at a time without awareness of system-level contracts.

## Preliminary findings (caveat: small sample, single codebase)

From 4 A/B experiments on an agent orchestration controller:
- A $0.10 invariant verification pass caught defects that $0.11 code review missed
- 5 real bugs found across ~8 verification runs
- Cross-component invariants (spanning 2+ files) had the highest detection rate
- The ENFORCE/VERIFY/CHECK distinction changed agent behavior measurably

These are anecdotal. The proposed study tests whether they replicate at scale.

## Study design

### Setting
Production codebase at [company]. Multiple services, multiple languages, real engineering tasks. Tasks written by different engineers, not the researchers.

### Conditions
1. **Control**: Agent generates code → unit tests → LLM code review → merge
2. **Treatment A**: Same as control + human-written invariants + automated verification
3. **Treatment B**: Same as control + LLM-discovered invariants + automated verification

### Blinding
- Engineers file tasks normally (no awareness of study condition)
- Invariants are written/discovered BEFORE the agent runs (from ticket spec only)
- Verification runs automatically on the PR
- An independent evaluator classifies flagged violations as true/false positive
- Post-merge defects tracked for 30 days across all conditions

### Sample size
Target: 50 tasks per condition (150 total). Power analysis TBD based on expected effect size from preliminary data.

### Metrics

**Primary:**
- Defect detection rate: % of real defects caught by invariant verification that were missed by tests + code review
- Post-merge defect rate: defects reaching production within 30 days, by condition
- False positive rate: % of VIOLATED reports that are not actual defects

**Secondary:**
- Cost per defect found: verification cost / true positives
- Spec authoring overhead: time to write invariants vs standard ticket
- Invariant type effectiveness: which invariant patterns have highest detection rates
- Discovery agent accuracy: how do LLM-discovered invariants compare to human-written ones

### Analysis
- Fisher's exact test for defect detection rates between conditions
- Cost-effectiveness analysis (cost per defect prevented)
- Qualitative coding of defect types caught vs missed
- Taxonomy of effective invariant patterns

## Threats to validity

**Internal:**
- Selection bias: tasks may not be randomly assigned to conditions
- Author expertise: invariant quality depends on understanding the codebase
- Hawthorne effect: engineers may write better specs knowing the study is running

**External:**
- Single organization: results may not generalize to other codebases/cultures
- Codebase structure: microservices have clearer boundaries than monoliths
- Model dependency: tested only with Claude, may not replicate with other LLMs

**Construct:**
- "Defect" definition may vary between evaluators
- Post-merge defects may be caught by other means (monitoring, users) regardless

## Tooling (built as part of the study)

### Claude Code skills
- `/invariant-discover` — reads ticket spec + target code, identifies cross-component invariants
- `/invariant-verify` — reads PR diff, checks named invariants, reports SATISFIED/VIOLATED

### Invariant library
Named, reusable patterns: BackwardCompat, TotalCoverage, NilSafety, BehaviorPreservation, RoundTrip, NoFalsePositive, ParseValidateConsistency, IdempotentOperation, TypePreservation.

### Data collection
- All specs, invariants, generated code, verification reports, and evaluations stored
- Anonymized replication package published with the paper

## Timeline

| Month | Activity |
|-------|----------|
| 1 | Build skills, finalize study protocol, IRB if needed |
| 2-3 | Run study: 150 tasks across 3 conditions |
| 4 | 30-day post-merge defect tracking |
| 5 | Analysis and write-up |
| 6 | Submit to ICSE 2027 or FSE 2027 |

## Budget

- Agent runs: ~$1-3/task × 150 = $150-450
- Verification runs: ~$0.20/task × 150 = $30
- Independent evaluator time: ~20 hours
- Total: <$500 in API costs + evaluator time

## Target venues

- ICSE 2027 — Software Engineering in Practice track
- FSE 2027 — Industry track
- ASE 2027
- Empirical Software Engineering (journal, no deadline)

## Related work

- Meyer (1992) — Design by Contract / Eiffel
- Formal verification for AI code (Verified Lifting, etc.)
- Property-based testing (QuickCheck, Hypothesis)
- SWE-bench, HumanEval — agent capability benchmarks (we measure spec quality, not agent capability)
- LLM self-consistency / self-refinement — checks internal consistency, not external contracts
