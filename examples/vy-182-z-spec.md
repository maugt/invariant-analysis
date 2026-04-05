# Example: Z Analysis of VY-182 (pr_url bug)

The bug: agent-runner creates PRs but doesn't include the URL in the result
ConfigMap. The controller reads the ConfigMap and skips the review pipeline
because pr_url is empty.

## Z notation

```z
┌─ RunnerState ───────────────────────────┐
│ pr_created  : 𝔹                          │
│ pr_url      : PR_URL ∪ {∅}              │
├──────────────────────────────────────────│
│ pr_created ⇒ pr_url ≠ ∅                 │
│ ¬pr_created ⇒ pr_url = ∅               │
└──────────────────────────────────────────┘

┌─ ReviewPipelineInvariant ───────────────┐
│ r : RunnerState                          │
│ cm : ResultConfigMap                     │
├──────────────────────────────────────────│
│ r.pr_created ⇒ cm.data.pr_url ≠ ∅      │
└──────────────────────────────────────────┘
```

## What Z found

The invariant `pr_created ⇒ cm.data.pr_url ≠ ∅` spans two components:
- The runner (which creates PRs and writes the ConfigMap)
- The controller (which reads the ConfigMap and dispatches reviews)

Unit tests on either side pass independently. The bug is in the contract
between them — the runner doesn't fulfill the invariant that the controller
assumes.

## DSL equivalent

```
ENFORCE ReviewPipelineContract:
  runner.pr_created => result_configmap.pr_url != ""
  (* if the runner creates a PR, the result must contain its URL *)
  spans: agent-runner/runner.go → controller/step_running.go
```

## Key insight

The Z notation forced us to state the cross-component property explicitly.
The bug wasn't in either component — it was in the unstated assumption
between them. No amount of unit testing catches this; you need the
invariant stated and checked.
