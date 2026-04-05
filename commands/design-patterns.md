---
name: design-patterns
description: |
  This skill should be used when the user asks to "analyze design patterns",
  "what patterns does this code use", "review patterns", "suggest patterns",
  or wants to identify, critique, or improve design patterns in code.
user-invocable: true
allowed-tools: Bash, Read, Glob, Grep, WebFetch
activation:
  keywords:
    - design patterns
    - analyze patterns
    - pattern review
    - what pattern
    - suggest pattern
    - refactor pattern
    - anti-pattern
  priority: 60
---

# Design Patterns Analyzer

Analyze code for design patterns — identify what's being used, suggest improvements, and flag anti-patterns.

## Reference

Use https://refactoring.guru/design-patterns as the authoritative reference. When you need details on a specific pattern, fetch the page:

```
https://refactoring.guru/design-patterns/{pattern-name}
```

For example: `https://refactoring.guru/design-patterns/observer`

## How to Analyze

When the user asks to analyze code for design patterns:

### Step 1: Read the code

Read the files the user points to, or if they say "this repo" / "this directory", use Glob and Grep to find the key source files (entry points, interfaces, base classes, factories, handlers).

### Step 2: Identify patterns in use

For each pattern you identify:
- **Name** the pattern
- **Where** — file and line range
- **How** — brief explanation of how the code implements it
- **Quality** — is it a clean implementation or a partial/distorted version?

### Step 3: Identify anti-patterns or missed opportunities

Look for:
- **God objects** — classes doing too much
- **Shotgun surgery** — one change requires edits across many files
- **Feature envy** — methods that use another class's data more than their own
- **Primitive obsession** — using primitives where value objects would be clearer
- **Missing abstractions** — switch statements or type checks that could be polymorphism

### Step 4: Suggest improvements

For each suggestion:
- Name the pattern that would help
- Explain **why** it fits (don't force patterns where they don't belong)
- Sketch the change at a high level (don't rewrite the code unless asked)
- Link to the refactoring.guru page for reference

### Step 5: Present results

Format as:

```
## Patterns Identified

| Pattern | Location | Quality | Notes |
|---------|----------|---------|-------|
| Observer | src/events.ts:15-80 | Clean | Event emitter with typed listeners |
| Singleton | src/config.ts:5 | Anti-pattern | Global mutable state, consider DI |

## Suggestions

1. **Strategy pattern** for `src/processor.ts:45-90`
   The switch statement on `type` could be replaced with strategy objects.
   Ref: https://refactoring.guru/design-patterns/strategy

2. **Facade** for `src/api/` directory
   The 8 individual API modules could benefit from a unified facade.
   Ref: https://refactoring.guru/design-patterns/facade
```

If you need more detail on any pattern to give a good analysis, fetch the refactoring.guru page for that pattern to refresh your understanding of its structure, applicability, and trade-offs.
