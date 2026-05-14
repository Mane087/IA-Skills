---
name: elixir-testing
description: >
  Guide for writing, reviewing, and improving tests in Elixir applications,
  including ExUnit, OTP processes, Ecto/database flows, Phoenix behavior, and
  dependency isolation strategies.
  Trigger: Use when the task involves creating or debugging tests, reviewing
  test quality, deciding between unit/integration/OTP tests, working with
  ExUnit, testing GenServers or supervised processes, mocking or isolating
  external dependencies, validating Ecto-backed behavior, or diagnosing flaky
  Elixir test cases.
metadata:
  stack: elixir
  area: testing
---

## Purpose

Use this skill whenever the task involves testing Elixir code.

This includes:
- writing new tests
- improving existing tests
- reviewing test quality
- testing OTP processes
- testing integrations with external services
- testing Ecto or Phoenix code
- reducing flaky tests
- deciding whether a test should be unit, integration, or end-to-end

## Core mindset

Testing should optimize for:
1. confidence that the code works
2. regression prevention
3. fast failure localization
4. the least amount of necessary test code

Do not write tests just to increase line coverage.
Do not overfit tests to implementation details.
Prefer tests that verify externally observable behavior.

## Default principles

### 1. Prefer black-box tests over implementation-coupled tests
Test public behavior first.
Do not inspect internal state unless there is a strong reason and no better design is available.

### 2. Prefer pure functions when possible
If logic can be extracted from GenServers, controllers, or orchestration modules into pure functions, do that first.
Pure functions are the easiest and cheapest code to test.

### 3. Keep test scope intentional
Choose whether the test is:
- unit
- integration
- end-to-end

Do not accidentally mix all three in one test.

### 4. Use ExUnit idiomatically
Prefer:
- `describe`
- `setup`
- `setup_all` only when shared state is safe
- `on_exit` for cleanup
- `assert`, `refute`, `assert_receive`, `assert_raise`
- `start_supervised!` for OTP processes

### 5. Avoid brittle timing-based tests
Do not rely on `Process.sleep/1` as the first synchronization tool.
Prefer:
- message passing
- `assert_receive`
- explicit expectations with Mox
- controllable timers or injectable time sources

### 6. Keep setup readable
Extract repeated setup only when it genuinely improves clarity.
Do not hide all important test logic in setup helpers.

### 7. Use randomized values carefully
Use dynamic values when they help avoid accidental hardcoding assumptions.
But keep failures reproducible and understandable.

### 8. Do not share logic between production code and test expectations
Avoid DRY between production and test assertions when that would let both fail in the same way.
Tests should verify behavior independently.

## Test design rules

Before writing a test, identify:

1. What behavior is being verified?
2. What is the test boundary?
3. What dependencies must be real?
4. What dependencies should be isolated?
5. Is the failure message going to make debugging easy?
6. Is the test stable under repeated runs?

## Unit testing rules

Use unit tests for:
- pure functions
- deterministic transformation logic
- validation logic
- small domain operations
- modules with stable input/output behavior

For unit tests:
- prefer direct input/output assertions
- avoid global state
- avoid process coordination unless the unit itself is concurrent
- split tests by behavior, not by arbitrary branches

## Integration testing rules

Use integration tests for:
- module interactions
- database interactions
- boundary adapters
- HTTP client wrappers
- interface-to-service contracts

Prefer real owned dependencies when practical:
- database
- local services
- supervised components you control

Prefer doubles for third-party dependencies you do not control:
- external APIs
- SMS gateways
- payment providers
- email platforms

## OTP testing rules

For GenServers and OTP code:
- test public API first
- use `start_supervised!` instead of manually linking processes when possible
- avoid reaching into process state unless the design leaves no reasonable alternative
- if the process mixes state and business logic, extract a functional core and test it separately
- test async behavior with messages or explicit synchronization, not blind sleeps

## Ecto and database rules

For Ecto-related code:
- test changesets as validators
- test queries through actual DB interactions when query correctness matters
- use sandbox or equivalent isolation
- keep fixtures/factories explicit and readable
- do not assert on raw SQL unless the exact SQL shape is the point of the test

## External dependency rules

When testing external integrations:
- define a behaviour for the dependency boundary
- inject the dependency via module or configuration
- use Mox when verifying calls and arguments matters
- use stubs when only a replacement behavior is needed
- use real integration tests selectively for critical contracts

## Mox rules

Use Mox when:
- there is a clear behaviour boundary
- interaction with the dependency is part of what the test verifies

Prefer:
- `expect/4` when call count and arguments matter
- `stub/3` when only a response is needed
- `verify_on_exit!` in setup for expectation verification

Do not use mocks for purely functional internal code.
Do not create a behaviour just to mock trivial internal helpers.

## Anti-patterns

Do not:
- test private functions directly unless there is a compelling reason
- overuse `setup_all` with mutable shared state
- use `Process.sleep/1` as the core correctness mechanism
- assert implementation details that are free to change
- duplicate production logic in expected values
- create giant tests that verify too many unrelated things
- mock everything by default
- hide domain meaning behind generic test names like "works correctly"

## Preferred response structure

When this skill is used, produce output in this order:

1. **Test strategy**
- what should be tested
- what level of test is appropriate
- what should stay real vs isolated

2. **Recommended implementation**
- concrete test structure
- helpers or setup needed
- dependencies or behaviours needed

3. **Code examples**
- include at least one correct pattern
- include at least one incorrect pattern
- explain why the incorrect one is weak

4. **Risk notes**
- flakiness risks
- overcoupling risks
- maintenance risks

## Correct pattern examples

### Example 1: good unit test for a pure function

```elixir
defmodule MyApp.PriceCalculatorTest do
  use ExUnit.Case, async: true

  alias MyApp.PriceCalculator

  describe "total/2" do
    test "adds tax to subtotal" do
      assert PriceCalculator.total(100, 0.16) == 116.0
    end

    test "returns subtotal when tax is zero" do
      assert PriceCalculator.total(100, 0.0) == 100.0
    end
  end
end
````

Why this is good:

* tests public behavior
* deterministic
* no unnecessary setup
* easy to debug

### Example 2: good OTP test using start_supervised!

```elixir
defmodule MyApp.RollingAverageServerTest do
  use ExUnit.Case, async: true

  alias MyApp.RollingAverageServer

  describe "average/1" do
    test "returns the rolling average of inserted values" do
      pid = start_supervised!({RollingAverageServer, max_measurements: 2})

      RollingAverageServer.add_element(pid, 5)
      RollingAverageServer.add_element(pid, 7)

      assert RollingAverageServer.average(pid) == 6.0
    end
  end
end
```

Why this is good:

* tests the server through its public API
* supervised lifecycle is controlled
* no manual state inspection
* no arbitrary sleeps

### Example 3: good async verification with assert_receive

```elixir
defmodule MyApp.NotifierTest do
  use ExUnit.Case, async: true

  test "sends notification after processing" do
    test_pid = self()

    worker =
      start_supervised!({MyApp.Worker, notify: fn message ->
        send(test_pid, {:notified, message})
      end})

    MyApp.Worker.run(worker, "done")

    assert_receive {:notified, "done"}
  end
end
```

Why this is good:

* synchronizes on a real event
* avoids timing guesses
* asserts observable behavior

### Example 4: good Mox usage with behaviour boundary

```elixir
defmodule MyApp.SMSClientBehaviour do
  @callback send_sms(String.t(), String.t()) :: :ok | {:error, term()}
end
```

```elixir
Mox.defmock(MyApp.SMSClientMock, for: MyApp.SMSClientBehaviour)
```

```elixir
defmodule MyApp.AlertServiceTest do
  use ExUnit.Case, async: false

  import Mox

  setup :verify_on_exit!

  test "sends one SMS alert with the expected payload" do
    expect(MyApp.SMSClientMock, :send_sms, 1, fn phone, text ->
      assert phone == "+521234567890"
      assert text =~ "rain"
      :ok
    end)

    assert :ok =
             MyApp.AlertService.send_alert(
               "+521234567890",
               "rain expected",
               sms_client: MyApp.SMSClientMock
             )
  end
end
```

Why this is good:

* tests the boundary contract
* checks arguments and call count
* isolates third-party dependency cleanly

## Incorrect pattern examples

### Example 1: bad test coupled to internal state

```elixir
defmodule MyApp.RollingAverageServerTest do
  use ExUnit.Case, async: true

  test "stores the latest numbers internally" do
    {:ok, pid} = MyApp.RollingAverageServer.start_link(max_measurements: 2)

    MyApp.RollingAverageServer.add_element(pid, 5)
    MyApp.RollingAverageServer.add_element(pid, 7)

    assert %{measurements: [7, 5]} = :sys.get_state(pid)
  end
end
```

Why this is weak:

* tied to internal representation
* breaks on harmless refactors
* verifies structure, not real public behavior

Better approach:

* assert through `average/1` or another public API
* extract functional core if the logic needs deeper testing

### Example 2: bad timing-based test

```elixir
defmodule MyApp.WorkerTest do
  use ExUnit.Case, async: true

  test "eventually sends a message" do
    start_supervised!(MyApp.Worker)

    MyApp.Worker.run()

    Process.sleep(200)

    assert MyApp.Worker.finished?()
  end
end
```

Why this is weak:

* timing-dependent
* potentially flaky in CI
* sleep duration is a guess, not proof

Better approach:

* send a message to the test process
* use `assert_receive`
* inject a callback or event sink if needed

### Example 3: bad overuse of setup_all with mutable state

```elixir
defmodule MyApp.AccountsTest do
  use ExUnit.Case

  setup_all do
    user = MyApp.Repo.insert!(%MyApp.User{name: "Alice"})
    %{user: user}
  end

  test "updates the user", %{user: user} do
    MyApp.Accounts.rename_user(user, "Bob")
    assert MyApp.Accounts.get_user!(user.id).name == "Bob"
  end

  test "expects original user name", %{user: user} do
    assert MyApp.Accounts.get_user!(user.id).name == "Alice"
  end
end
```

Why this is weak:

* shared mutable state across tests
* order sensitivity
* hidden coupling between tests

Better approach:

* create data per test
* use sandboxed DB isolation
* reserve `setup_all` for safe immutable setup

### Example 4: bad mock-everything style

```elixir
defmodule MyApp.MathServiceTest do
  use ExUnit.Case

  import Mox

  test "adds numbers" do
    expect(MyApp.CalculatorMock, :add, fn 1, 2 -> 3 end)
    assert MyApp.MathService.sum(1, 2, MyApp.CalculatorMock) == 3
  end
end
```

Why this is weak:

* mocking trivial internal logic
* adds indirection without value
* hides simpler pure-function testing

Better approach:

* test the function directly without mocks
* reserve mocks for true external boundaries

## Review checklist

When reviewing an Elixir test, ask:

* Is this test verifying behavior or implementation detail?
* Is there unnecessary state inspection?
* Is `Process.sleep/1` hiding a synchronization problem?
* Could a pure function be extracted?
* Should this dependency be real, stubbed, or mocked?
* Is setup helping readability or hurting it?
* Will the failure message clearly reveal the problem?
* Is shared state isolated correctly?

