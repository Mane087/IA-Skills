---
name: test-driven-development
description: >
  Practical guide for applying test-driven development to new development,
  bug fixes, and behavior changes.
  Trigger: Use when the task involves implementing a new behavior, fixing a bug,
  changing observable behavior, designing code from requirements, or deciding
  what test should be written first before production code. Also use when
  reviewing whether a change should follow a red-green-refactor workflow instead
  of coding first.
---

# Test-Driven Development (TDD)

## Overview

Use tests as a design and verification tool, not as ceremony.

The default TDD cycle is:

1. Write a test for the behavior you want.
2. Run it and confirm it fails for the expected reason.
3. Implement the smallest change that makes it pass.
4. Refactor while keeping tests green.

**Core principle:** tests should help define, protect, and clarify behavior.

TDD is the default approach for behavior-heavy changes, but it is not a ritual to apply blindly to every trivial edit.

---

## When to Use TDD

### Strong default

Use TDD for:

- New business logic
- Bug fixes
- Validation rules
- Condition-heavy flows
- Public API or service behavior
- Refactors where behavior must remain stable
- Anything with meaningful regression risk

### Often helpful, but not always required

Consider TDD for:

- UI behavior with non-trivial state changes
- Data transformation logic
- Integration points with important contracts
- Shared utilities

### Cases where strict TDD may be unnecessary

You may skip strict test-first when the change is clearly trivial and low risk, such as:

- Mechanical renames
- Formatting-only changes
- Documentation updates
- Generated code
- Very small wiring/configuration edits with no real logic
- Throwaway spikes or prototypes

If a change is small but still alters important behavior, test it.

**Decision rule:** do not ask whether the change is small; ask whether it changes behavior worth protecting.

---

## Recommended Cycle: Red → Green → Refactor

### RED — Write a failing test

Write one focused test that describes the desired behavior.

```typescript
test('rejects empty email', async () => {
  const result = await submitForm({ email: '' });
  expect(result.error).toBe('Email required');
});
```

Good test characteristics:

- Covers one behavior
- Has a precise name
- Makes the expected outcome obvious
- Tests observable behavior, not internal implementation details

### Verify RED

Run the test and confirm:

- It fails
- It fails for the expected reason
- The failure is caused by missing behavior, not a broken setup or typo

A passing test before implementation usually means one of three things:

- The behavior already exists
- The test is too weak
- The assertion is not checking the right thing

### GREEN — Implement the smallest useful change

Add the minimum code needed to satisfy the test.

```typescript
function submitForm(data: FormData) {
  if (!data.email?.trim()) {
    return { error: 'Email required' };
  }

  // existing flow
}
```

Avoid adding unrelated abstractions or speculative features.

### Verify GREEN

Run the relevant tests and confirm:

- The new test passes
- Existing related tests still pass
- No new warnings or unexpected failures appear

### REFACTOR — Clean up safely

Once green:

- Remove duplication
- Improve naming
- Extract helpers if they simplify the design
- Keep the same behavior

Refactoring is not the moment to expand scope.

---

## Practical Rules

### 1. Prefer behavior over implementation details

A good test describes what the system should do, not how the code happens to be structured today.

### 2. Keep tests small and explicit

If a test name contains multiple behaviors, it is often doing too much.

### 3. Use the smallest test that gives confidence

Not every change needs a large integration or end-to-end test.
Choose the cheapest test that still protects the behavior.

### 4. Do not overreact to the process

If you explored an idea before writing the test, that does not invalidate the work. What matters is whether the final implementation is grounded by meaningful tests before you consider the change complete.

### 5. Bugs should usually end with a regression test

If a bug was real enough to matter, it is usually worth preserving with a test that proves the fix.

---

## Test Selection Guidance

Use the most appropriate level:

### Unit tests
Best for:

- Pure logic
- Validation
- Calculations
- Small rule-based behavior

### Integration tests
Best for:

- Collaboration between components
- Database/service boundaries
- Framework wiring that actually matters

### End-to-end tests
Best for:

- Critical user flows
- Cross-layer confidence
- High-value paths that must not break

Do not force everything into unit tests if integration tests are more honest and simpler.

---

## Good Test Heuristics

| Quality | Better | Worse |
|---------|--------|-------|
| Focus | One behavior per test | Many behaviors packed together |
| Naming | Describes outcome | Vague labels like `test1` or `works` |
| Assertions | Observable result | Internal implementation details |
| Scope | Minimal but meaningful | Huge setup for tiny value |
| Confidence | Protects behavior | Only mirrors current code structure |

---

## Common Failure Modes

### “I will write tests after”

That can still produce useful tests, but it is not the same as TDD.
Tests written after implementation tend to validate what you built, not what the requirement actually demanded.

### “This is too small to test”

Maybe. But small code can still have high impact.
Judge by behavior and risk, not by line count.

### “Manual testing is enough”

Manual verification is helpful, but it is not durable. It cannot be re-run consistently on every future change.

### “This test is hard to write”

Sometimes that means the design is too coupled, the API is awkward, or the behavior is not clearly framed.
That signal is useful.

---

## Completion Checklist

Before considering a change complete, ask:

- Is the important behavior covered by at least one meaningful test?
- Did I verify the test can actually fail?
- Am I testing behavior instead of implementation trivia?
- Did I choose an appropriate test level?
- Are edge cases covered where they matter?
- Can someone else understand the requirement by reading the tests?

You do not need every change to satisfy every possible checkbox.
You do need enough evidence that the behavior is protected.

---

## When to Pause and Reassess

Pause if:

- Mock setup is larger than the behavior under test
- You are asserting mostly on implementation details
- The test is painful because the design is overly coupled
- You are adding tests only to satisfy policy, not to protect behavior
- The change is trivial enough that a test adds near-zero value

In those cases, step back and choose a better test level, simplify the design, or document why a formal test is not needed.

---

## Bottom Line

TDD is a strong default, especially for new projects and behavior-heavy code.
Use it to drive design, catch regressions early, and keep changes honest.

But do not confuse discipline with rigidity.

The goal is not to perform TDD theatrically.
The goal is to ship behavior you can explain, verify, and safely change.
