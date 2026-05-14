---
name: test-driven-development-antipatterns
description: >
  Guide for spotting and avoiding common testing and TDD anti-patterns, especially
  around mock misuse, over-isolation, implementation-coupled assertions, and weak
  integration coverage.
  Trigger: Use when reviewing or writing tests for new development, bug fixes, or
  behavior changes, especially if the task involves mocks, test doubles, isolation
  strategy, integration coverage decisions, cleanup logic added for tests, or
  diagnosing brittle tests that pass without proving real behavior.
---

# Testing Anti-Patterns

Use this reference when writing tests, adding mocks, or deciding how much isolation a test really needs.

## Overview

Tests should verify meaningful behavior.
Mocks, stubs, and fakes are tools to control dependencies, not the main thing being proven.

**Core principle:** test what the system does, not what the test setup pretends to do.

These anti-patterns are especially common when teams try to adopt TDD mechanically instead of thoughtfully.

---

## Anti-Pattern 1: Testing Mock Behavior Instead of Real Behavior

### Problem

```typescript
// ❌ Weak test: proves the mock rendered, not the feature
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

### Why this is weak

- It confirms the mock exists, not that the feature behaves correctly
- It can pass even when the real integration is broken
- It gives false confidence

### Better approach

```typescript
// ✅ Better: assert on actual page behavior
test('renders navigation area', () => {
  render(<Page />);
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});
```

If you truly need to mock something, keep the assertion focused on the behavior of the unit under test, not the existence of the mock.

### Quick check

Ask:

- Am I asserting on the product behavior?
- Or am I just proving the mock was mounted?

If the second one is true, the test is probably weak.

---

## Anti-Pattern 2: Adding Test-Only Methods to Production Code

### Problem

```typescript
// ❌ Suspicious: method exists only for tests
class Session {
  async destroy() {
    await this._workspaceManager?.destroyWorkspace(this.id);
  }
}
```

### Why this is risky

- It pollutes the production API with test concerns
- It can blur ownership and lifecycle boundaries
- It often signals that test utilities belong elsewhere

### Better approach

```typescript
// ✅ Better: keep cleanup in test support code
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}
```

### Quick check

Before adding a method, ask:

- Would this method still make sense if no tests existed?
- Does this class actually own that resource lifecycle?

If not, move the concern out of production code.

---

## Anti-Pattern 3: Mocking Without Understanding the Dependency

### Problem

```typescript
// ❌ Mock hides side effects the test actually needs
test('detects duplicate server', async () => {
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);
});
```

### Why this breaks tests

- The mocked method may have real side effects the flow depends on
- Over-mocking can remove the very behavior under test
- The test can fail or pass for the wrong reason

### Better approach

Mock at the lowest level that removes cost or instability without removing the behavior your test depends on.

```typescript
// ✅ Better: isolate only the expensive external part
test('detects duplicate server', async () => {
  vi.mock('MCPServerManager');

  await addServer(config);
  await addServer(config);
});
```

### Quick check

Before mocking, ask:

- What side effects does the real implementation have?
- Does my test depend on any of them?
- Am I mocking this because I understand the system, or just to “be safe”?

If the answer is uncertain, start with less mocking.

---

## Anti-Pattern 4: Incomplete Mocks That Hide Structural Assumptions

### Problem

```typescript
// ❌ Incomplete shape
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
};
```

### Why this is risky

- The real structure may contain fields other code relies on
- Downstream consumers may break even though the test passes
- The test proves too little about real integration behavior

### Better approach

Mirror the real shape closely enough that your test remains honest.

```typescript
// ✅ Better: include the expected contract shape
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
};
```

### Quick check

Before creating a mock payload, ask:

- What does the real contract look like?
- Am I preserving the fields other layers may consume?
- Would this mock survive a realistic integration path?

---

## Anti-Pattern 5: Treating Integration Tests as an Afterthought

### Problem

```text
Implementation complete
No useful integration coverage
"Ready for testing"
```

### Why this is weak

- Some failures only appear when components collaborate
- Unit tests alone do not validate every important boundary
- Testing only at the end delays feedback unnecessarily

### Better approach

For flows that cross meaningful boundaries, add integration tests as part of development, not as an optional cleanup task.

This does not mean every change needs integration tests.
It means important interactions should not be left to hope or manual clicking.

---

## Anti-Pattern 6: Over-Complex Mock Setup

### Warning signs

- Mock setup is longer than the test itself
- You need many nested mocks just to make the test run
- The mock has to simulate most of the real system anyway
- Small implementation changes constantly break the test

### Better approach

When mocking becomes elaborate, consider whether a higher-level test with more real components would be simpler and more trustworthy.

Sometimes the cleanest test is an integration test with a small amount of controlled setup.

---

## Anti-Pattern 7: Asserting on Implementation Details by Default

### Problem examples

- Verifying private method calls indirectly
- Asserting exact internal call order when behavior is what matters
- Tying the test to helper structure rather than user-visible result

### Why this is weak

- Tests become brittle during refactors
- They discourage cleanup even when behavior stays correct
- They overfit to current implementation instead of intended outcome

### Better approach

Prefer assertions about:

- outputs
- state changes that matter
- emitted events
- returned values
- rendered behavior
- persisted effects

Only assert internal collaboration when it is itself part of the requirement.

---

## Practical Guidance on Mocks

Mocks are not bad. Misused mocks are.

Mocks are often appropriate when you need to:

- isolate slow or flaky external dependencies
- simulate failures that are hard to trigger reliably
- verify collaboration with a true boundary
- keep tests fast enough to run continuously

But prefer restraint:

- mock lower-level external concerns first
- preserve meaningful behavior when possible
- do not mock blindly just because something is available to mock

---

## Quick Reference

| Anti-Pattern | Better Direction |
|--------------|------------------|
| Asserting on mock existence | Assert on real behavior |
| Test-only methods in production | Move cleanup/support to test helpers |
| High-level mocking without understanding side effects | Mock lower-level boundaries carefully |
| Partial payload mocks | Mirror the real contract shape |
| No integration coverage where it matters | Add integration tests for important boundaries |
| Huge mock setup | Consider a higher-level or more realistic test |
| Implementation-detail assertions everywhere | Prefer observable outcomes |

---

## Bottom Line

A good test gives confidence with the least distortion possible.

Use mocks when they help isolate cost, instability, or irrelevant detail.
Do not let mocks replace the actual behavior you need to trust.

The goal is not maximum isolation at any price.
The goal is reliable feedback about real behavior.
