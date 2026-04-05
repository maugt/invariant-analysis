# Agent Spec Benchmark Suite

Controlled test cases from production PRs with known outcomes. Each entry captures the pre-change commit, the spec, known bugs in the output, and what each review method found.

Use these to A/B test spec formats: replay from the pre-change commit with different specs and compare outcomes.

## How to run a benchmark case

1. Clone the repo at the specified base commit
2. Apply the spec as the agent prompt
3. Run the agent (or simulate with Claude Code)
4. Compare output against known bugs and expected diff
5. Score: bugs introduced, bugs avoided, cost, token count

## Cases

### Case 1: VY-194 Idle Timeout (PR #122)

**Base commit**: `136a6ea` (just before VY-194 merged)
**Files changed**: `agent-sandbox/agent-runner/runner/runner.go`, `runner_test.go`
**Complexity**: Medium (new feature, modifying existing select loop)

**Known bugs in original output**:
1. `cmd.Wait()` missing after `Kill()` on timeout and cancel paths (PathSymmetry)
2. Config validation: `AGENT_IDLE_TIMEOUT_MINUTES=0` silently uses 10m fallback
3. Flaky test: `sleep 0.04` with 80ms timeout, insufficient margin on slow hardware
4. `sleep 300` without `exec` in test — orphan process holds pipe open

**Review detection matrix**:
| Bug | Haiku (old) | Haiku (adversarial) | CodeRabbit | Invariant | Exhaustive | SelfSpec |
|-----|------------|-------------------|-----------|-----------|------------|---------|
| cmd.Wait() | - | - | - | - | - | Found |
| Config validation | - | Found | Found | - | - | Partial |
| Flaky test | - | Found | Found | - | - | - |
| exec sleep | - | - | - | - | - | - |

### Case 2: VY-173 Checkpoint/Resume (PR #117)

**Base commit**: `e7bb256` (just before VY-173 merged)
**Files changed**: `runner.go`, `runner_test.go`, `step_continuable.go`, `step_continuable_test.go`
**Complexity**: Medium (enhance existing WIP notes + new continuation prompt)

**Known bugs in original output**:
1. `## Still needed` → `## Remaining` rename breaks old WIP branches (BehaviorPreservation)
2. No cross-component test for section names between runner and controller (RoundTrip)

**Additional bugs found by exhaustive verify**:
3. Unchecked `git add -A` error in `commitAndPushWIP` — staging fails silently
4. Child RetryCount status update error logged but not returned

**Review detection matrix**:
| Bug | Haiku (old) | Invariant | Exhaustive |
|-----|------------|-----------|------------|
| Section rename | - | Found | - |
| Cross-component test | - | Found | - |
| Unchecked git add | - | - | Found |
| RetryCount not persisted | - | - | Found |

### Case 3: VY-198 Linear Comments (PR #126)

**Base commit**: `7078011` (just before VY-198 merged)
**Files changed**: `linear_comment_hook.go`, `linear_comment_hook_test.go`, `source.go`, `main.go`
**Complexity**: Medium (new OnCompleteHook, main.go restructure)

**Known bugs in original output**:
1. `annoLinearCommentPosted` defined locally, not in centralized constants (AnnotationConsistency)
2. Phase name exposure: `WaitingForReview` shown as raw enum in Linear comments
3. Race condition: PostComment succeeds → annotation Update fails → duplicate on next reconcile (AtomicGuard)

**Review detection matrix**:
| Bug | Haiku (old) | CodeRabbit | Invariant |
|-----|------------|-----------|-----------|
| Annotation location | - | - | Found |
| Phase name | - | Found | - |
| Race condition | - | Found | - |

### Case 4: VY-190 Invariant Discovery Step (PR #120)

**Base commit**: `2a80f96` (just before VY-190 merged)
**Files changed**: `step_invariant_discovery.go`, `step_invariant_discovery_test.go`, `agenttask_controller.go`, `sandbox_template.go`
**Complexity**: Medium (new pipeline step)

**Known bugs in original output**:
1. `extractFilesToModify` reimplements parsing that exists in `linear/parse.go` (ParseValidateConsistency)
2. No per-task annotation for opt-in (added manually)

**Review detection matrix**:
| Bug | Haiku (old) | Invariant |
|-----|------------|-----------|
| Duplicate parser | - | Found |
| Missing per-task opt-in | - | Found (design review) |

### Case 5: VY-199 Config Dedup (PR #130)

**Base commit**: `599e871` (just before VY-199 merged)
**Files changed**: `sandbox_template.go`, `sandbox_template_test.go`, `values.yaml`, `INVARIANTS.md`, 4 test files
**Complexity**: Large (remove DefaultSandboxConfig, update all tests)

**Known bugs in original output**:
1. Test only spot-checks 20 of 62 config fields (TotalCoverage — fixed with reflection)
2. testConfig() sets all booleans true → breaks tests expecting disabled features

**Review detection matrix**:
| Bug | Haiku (adversarial) | CodeRabbit | Invariant |
|-----|-------------------|-----------|-----------|
| Incomplete test coverage | Found | - | Found |
| testConfig boolean breakage | - | - | - (caught by CI) |

### Case 6: VY-200 cmd.Wait() Fix (PR #131 vs #132)

**Base commit**: `b65b48c` (just before experiment)
**Files changed**: `runner.go`, `runner_test.go`
**Complexity**: Small (2-line fix on 2 paths)

**A/B tested**: Goal spec vs Transformation spec

| Metric | Goal Spec (#131) | Transform Spec (#132) |
|---|---|---|
| Core fix | Correct | Correct |
| Cost | $0.40 | $0.34 |
| Test quality | Racy (PID file race) | Clean |
| Review result | CHANGES_REQUESTED | APPROVED |

## Scoring rubric

For each benchmark case × spec format:

| Metric | How to measure |
|---|---|
| **Bugs introduced** | Count bugs in the agent's output (from the known bugs list) |
| **Bugs avoided** | Count bugs from the list that the agent didn't introduce |
| **Cost** | Dollar cost from result ConfigMap |
| **Tokens** | Input + output tokens (when VY-201 is implemented) |
| **Test quality** | Does the adversarial review approve? Any flaky tests? |
| **Diff precision** | Lines changed vs minimum required diff |

## Spec formats to compare

1. **Goal spec**: Problem + acceptance criteria + files to modify
2. **Transformation spec**: Current state matrix + exact delta + consistency check
3. **Enumeration spec**: Goal + forced enumeration of all cases before coding
4. **Combined**: Transformation + enumeration + cross-component invariants
