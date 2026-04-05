# Experiment 5: Invariant Analysis vs CodeRabbit

## Setup

Three PRs from the same session, reviewed by three systems simultaneously:
- **Invariant analysis**: Manual ENFORCE/VERIFY/CHECK verification against system invariants
- **CodeRabbit**: Automated AI code review (free tier)
- **Haiku review**: Automated haiku reviewer bot ($0.03-0.04 each)

## PRs

| PR | Ticket | Description | Complexity |
|---|---|---|---|
| #121 | VY-192 | Add tests for malformed invariant lines | Test-only, small |
| #122 | VY-194 | Idle timeout for agent-runner | New feature, medium |
| #123 | VY-191 | Post-completion invariant verify step | New pipeline step, medium |

## Findings comparison

| Finding | Invariant Analysis | CodeRabbit | Haiku Review |
|---|---|---|---|
| **#121**: Clean PR | 3/3 satisfied | Clean | Approved |
| **#122**: Idle timeout retries waste | **VIOLATED** — counterexample: tail -f → 3x retry → 30min waste | "Nitpick" suggestion | Not flagged |
| **#122**: Test exec sleep bug | Not caught | Not caught | Not caught |
| **#123**: ParseDescription on wrapped prompts | **VIOLATED** — counterexample: continuation task skips verification | **Not found** | Not flagged |

## Analysis

**Invariant analysis found a real bug CodeRabbit missed** (PR #123). The wrapped prompt issue is a cross-component bug spanning `step_continuable.go` (wraps prompts) and `step_invariant_verify.go` (parses prompts). Neither CodeRabbit nor the haiku reviewer caught it because it requires understanding the data flow across multiple files.

**Same finding, different severity framing** (PR #122). Both invariant analysis and CodeRabbit identified the idle timeout retry issue. Invariant analysis called it a VIOLATION with a concrete counterexample (tail -f → 3 retries → 30min). CodeRabbit called it a "nitpick". The invariant framing with a counterexample makes the bug harder to dismiss.

**Platform-dependent bugs remain hard** (PR #122). The `exec sleep` test bug was caught by neither system — it's a timing/platform issue that only manifests on slow hardware (Pi ARM in CI). This class of bug requires CI as the safety net.

## Cost

| System | Cost per PR | Bugs found |
|---|---|---|
| Invariant analysis (manual) | ~15 min human time | 2 (1 unique) |
| CodeRabbit (free tier) | $0 | 1 (shared with invariant) |
| Haiku review | $0.03-0.04 | 0 unique |

## Key takeaways

1. **Invariant analysis catches cross-component bugs at boundaries** — the class of bug where component A's output doesn't meet component B's assumptions. CodeRabbit and haiku review both miss these.
2. **CodeRabbit is better at single-file suggestions** — style, naming, potential improvements within one file.
3. **The counterexample discipline matters** — saying VIOLATED with a concrete input that breaks the property is more actionable than a suggestion.
4. **They're complementary, not competing** — invariant analysis for boundaries, CodeRabbit for single-file quality, haiku for quick approval.
5. **The cost difference will shrink** — when invariant analysis is automated in the controller (VY-190/191), it'll be ~$0.10-0.20 per PR vs $0 for CodeRabbit. The question is whether the incremental bugs justify the cost.
