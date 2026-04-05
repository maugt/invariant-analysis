Write a dispatchable specification for an agent task. Takes a description of what needs to be done as arguments.

Transform a rough task description into a structured, agent-ready specification. Use the codebase to fill in concrete file paths, existing patterns, and testable criteria.

## Steps

1. **Understand the task**: Read the arguments. If vague, ask one clarifying question — don't guess.

2. **Explore the codebase**: Find the files that need to change. Use Glob and Grep to locate:
   - Functions/types that will be modified
   - Test files that will need updating
   - Related code the agent should read for context
   - Configuration that must not be hardcoded

3. **Check scope**: If the task touches more than 5 files or requires changes across multiple modules, recommend decomposition. Large tasks burn tokens on incremental build failures. Suggest how to split.

4. **Identify invariants**: Look at boundaries between the changed code and its callers/consumers. For each boundary, ask:
   - What does this component assume about the other's output?
   - Could a refactor break this assumption silently?
   - Name the invariant using ENFORCE/VERIFY/CHECK:
     - ENFORCE: agent must add defensive code (guard clauses, validation)
     - VERIFY: agent must add tests
     - CHECK: both

5. **Write the spec** with these sections:

```markdown
## Files to modify
- `path/to/file.go` — what changes and why (use - or * list markers)

## Context files
- `path/to/related.go` — why the agent should read this

## Acceptance criteria
- [ ] Testable criterion 1
- [ ] Testable criterion 2
(Each must be verifiable by running tests or reading code — not aspirational)

## Build/test
```
exact commands to build and test
```

## Tests
- Specific test name or behavior to implement
- Edge cases to cover

## Invariants
ENFORCE InvariantName: property description
VERIFY InvariantName: property description
CHECK InvariantName: property description

## DO NOT change
- Files or behaviors that must not be touched
- Guardrails to prevent scope creep

## Configuration
- Values that must come from config, not hardcoded
```

## Rules

- **File paths must exist**. Verify with Glob before listing. If a file doesn't exist and needs to be created, use `## Files to create/modify`.
- **Acceptance criteria must be testable**. "Works correctly" is not testable. "Returns error for nil input" is.
- **Build/test must be exact commands**. Not "run the tests" — the actual `cd foo && go test ./...` command.
- **DO NOT change must be specific**. Not "don't break anything" — name the specific files, functions, or behaviors.
- **Invariants should span boundaries**. Single-function properties aren't worth stating. Cross-component contracts are.
- **Include the dangerous commands warning** if the task involves process execution, file watching, or streaming: agent must not use `tail -f`, `watch`, or other blocking commands.

## Anti-patterns to flag

If the task description has any of these, warn the user:
- **Mostly prose, no code structure** → needs more concrete specification
- **"Explore" or "investigate" language** → this is research, not a code task
- **No file paths** → agent will burn tokens exploring
- **Touches `.github/workflows/`** → agents can't push workflow files (GitHub PAT lacks workflow scope)
- **Single task touching 10+ files** → decompose into smaller tasks
- **"Make it configurable"** without specifying where config lives → specify the config layer

## Output

Present the spec as a single markdown block the user can paste into a Linear ticket. Do not create files or tickets — just output the spec.
