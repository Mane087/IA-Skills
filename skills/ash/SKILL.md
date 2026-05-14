---
name: ash
description: >
  Practical guide for working with Ash in Elixir applications.
  Trigger: Use when creating or modifying resources, domains, actions,
  relationships, queries, changesets, code interfaces, policies,
  calculations, aggregates, or when deciding if domain logic should live in Ash.
metadata:
  stack: elixir
  framework: ash
  area: domain-modeling
---

## Purpose

Use this skill whenever the task involves building or changing code that should live in Ash.

This includes:
- creating or modifying resources
- defining domains
- adding or updating attributes
- creating actions
- defining relationships
- using Ash.Query or Ash.Changeset
- exposing domain code interfaces
- designing policies
- adding calculations or aggregates
- deciding whether logic belongs in Ash or outside it
- testing Ash-backed behavior

## Core mindset

Ash is an application framework centered on domain modeling.

Do not treat it as "Ecto with extra macros".
Do not start from controllers, LiveViews, or database tables first.

Start from the business domain:
- what are the domain entities
- what operations matter
- what relationships exist
- what rules and permissions govern behavior

Ash resources act as the single source of truth for domain entities and their behavior.
Domains group related resources into clear boundaries.
The goal is to model the domain once and derive the rest where possible.

## First analysis step

Before proposing code, identify:

1. What business entity or concept is being modeled?
2. Which Ash domain should own it?
3. Is this a structural concern, a behavioral concern, or an authorization concern?
4. Should this be modeled as:
   - attribute
   - relationship
   - identity
   - action
   - validation
   - change
   - preparation
   - calculation
   - aggregate
   - policy
   - notifier
   - code interface
5. Is the requested logic really domain logic, or is it web/UI/integration logic that should stay outside Ash?

## Core placement rules

### 1. Model the domain first

Prefer Ash resources for core business entities.
Prefer Ash domains for grouping related resources with shared purpose.

### 2. Keep the resource as the source of truth

Attributes, relationships, actions, validations, policies, identities, calculations, aggregates, and related rules should live close to the resource whenever possible.

### 3. Prefer explicit business behavior

If an operation has domain meaning, model it as an Ash action with a meaningful name instead of burying it in helper modules.

### 4. Avoid scattering logic

Do not spread the same rule across:
- controllers
- LiveViews
- contexts
- ad hoc helper modules
- database queries

Prefer a single Ash-centered definition when the logic is domain-level.

### 5. Use escape hatches carefully

Ash supports Ecto and lower-level integration, but use escape hatches only when the Ash-native approach is clearly insufficient.
Do not reach for Ecto-first solutions without checking whether Ash already supports the requirement.

## Code interface rules

Use code interfaces on domains to define the public contract for calling Ash resources.

Prefer domain code interfaces over direct calls like:
- `Ash.get!/2`
- `Ash.read!/1`
- `Ash.load!/2`
- hand-built queries inside LiveViews or controllers

### Good

```elixir
resource DashboardGroup do
  define :get_dashboard_group_by_id, action: :read, get_by: [:id]
end

MyApp.Domain.get_dashboard_group_by_id!(id, load: [rel: [:nested]])
````

### Bad

```elixir
group =
  MyApp.Resource
  |> Ash.get!(id)
  |> Ash.load!(rel: [:nested])
```

### Code interface guidance

* Prefer the primary read action for "get" style interfaces
* Use `get_by` when looking up by primary key or identity
* Prefer passing `query:` and `load:` options to the code interface instead of manually building queries when the query is simple
* Prefer passing extra inputs as the map argument instead of adding optional arguments everywhere

### Preferred style

```elixir
posts =
  MyApp.Blog.list_posts!(
    query: [
      filter: [status: :published],
      sort: [published_at: :desc],
      limit: 10
    ],
    load: [author: :profile, comments: [:author]]
  )
```

### Avoid verbose query assembly in outer layers

```elixir
query =
  MyApp.Post
  |> Ash.Query.filter(...)
  |> Ash.Query.load(...)

Ash.read!(query)
```

Use manual query building only when the case is genuinely complex.

## Action design rules

When defining actions:

* Use default CRUD actions when the behavior is truly standard
* Use explicit custom actions when the operation has business meaning or needs custom rules
* Name actions according to domain behavior, not generic implementation details
* Use `accept` intentionally; do not expose writable fields casually
* Use `argument` when the action needs input that is not just a writable resource attribute
* Use `prepare` for read-time query shaping
* Use `change` for controlled create/update behavior
* Put business logic inside actions rather than external web modules
* A resource may be behavior-first and expose mostly generic/custom actions if that models the domain clearly

## Changes, validations, and preparations

### Validations

Use validations to enforce business rules before execution.
Validations validate; they do not mutate data.

Prefer:

* built-in validations for straightforward checks
* custom validation modules for reusable or complex logic
* `where` for conditional validations
* `only_when_valid? true` when expensive validations should be skipped after earlier failures

Avoid redundant validations that duplicate attribute constraints.

### Changes

Use changes to mutate the changeset before execution.

Prefer:

* built-in changes for simple attribute or relationship behavior
* custom change modules for reusable logic
* lifecycle hooks like:

  * `before_transaction`
  * `before_action`
  * `after_action`
  * `after_transaction`

Use changes when the system should transform or derive data, not when you only need to reject invalid input.

### Preparations

Use preparations to shape read queries before execution.

Prefer:

* `prepare build(...)` for stable read shaping
* conditional preparations with `where`
* `only_when_valid? true` when expensive preparation work should be skipped

## Custom modules vs anonymous functions

Prefer dedicated modules for reusable or non-trivial logic in:

* changes
* validations
* preparations
* calculations
* policy checks

### Prefer

```elixir
defmodule MyApp.MyDomain.MyResource.Changes.SlugifyName do
  use Ash.Resource.Change

  def change(changeset, _, _) do
    Ash.Changeset.before_action(changeset, fn changeset, _ ->
      slug = MyApp.Slug.get()
      Ash.Changeset.force_change_attribute(changeset, :slug, slug)
    end)
  end
end

change MyApp.MyDomain.MyResource.Changes.SlugifyName
```

### Avoid

Large anonymous functions inline inside actions when the logic is reusable or domain-important.

## Query rules

Use `Ash.Query` to build reads declaratively.

Common operations:

* filter
* sort
* load
* limit
* offset

### Important macro rule

`Ash.Query.filter/2` is a macro.
If you use it directly, `require Ash.Query`.

### Good

```elixir
defmodule MyApp.Example do
  require Ash.Query

  def by_id(id) do
    MyApp.Resource
    |> Ash.Query.filter(id == ^id)
  end
end
```

### Bad

Using `Ash.Query.filter/2` without `require Ash.Query` and then debugging pin/operator errors as if the query was wrong.

## Relationship rules

Use relationships to model domain connections.

Prefer:

* `belongs_to` for ownership/reference from the source resource
* `has_one` for single related record at the destination
* `has_many` for collections
* `many_to_many` with an explicit join resource

Prefer relationships over manual foreign key reasoning in calling code.

When managing relationships:

* use `manage_relationship` in actions when input comes from action arguments
* use `Ash.Changeset.manage_relationship/3-4` in custom changes when values are built programmatically

Do not fake relationship handling with ad hoc param merging in outer layers.

## Calculations and aggregates

Use calculations for derived values.
Use aggregates for summarized values across related or unrelated data.

Prefer:

* expression calculations when the logic is simple and data-layer-friendly
* module calculations when the logic is more complex
* aggregates for counts, sums, mins, maxes, averages, existence checks, and first/list patterns

Do not put repeated derived-value logic into controllers or LiveViews when it is really resource-level behavior.

## Authorization rules

Use policies to define authorization rules at the resource level.

Prefer:

* explicit policies by action type or action name
* actor-aware checks
* field policies when field visibility matters
* bypass policies almost exclusively for admin-style bypasses

### Critical policy rule

The first check inside a policy that yields a decision determines that policy outcome.

Do not accidentally write OR logic when you meant AND logic.

### Wrong

```elixir
policy action_type(:update) do
  authorize_if actor_attribute_equals(:admin?, true)
  authorize_if relates_to_actor_via(:owner)
end
```

### Correct AND-style intent

```elixir
policy action_type(:update) do
  forbid_unless actor_attribute_equals(:admin?, true)
  authorize_if relates_to_actor_via(:owner)
end
```

## Actor and scope rules

Always set actor-related context on the query, changeset, or form input, not only at the final action call.

### Good

```elixir
Post
|> Ash.Query.for_read(:read, %{}, actor: current_user)
|> Ash.read!()
```

### Bad

```elixir
Post
|> Ash.Query.for_read(:read, %{})
|> Ash.read!(actor: current_user)
```

When using `Ash.Scope` in LiveViews, prefer using the assigned `scope`.

### LiveView style

```elixir
MyApp.Blog.create_post!("new post", scope: socket.assigns.scope)
```

### Inside hooks or callbacks

Use the provided `context` parameter as the scope.

```elixir
|> Ash.Changeset.before_transaction(fn changeset, context ->
  MyApp.ExternalService.reserve_inventory(changeset, scope: context)
  changeset
end)
```

## Project-specific date and time rule

### Critical rule

Never use these directly in this project:

* `DateTime.utc_now()`
* `NaiveDateTime.utc_now()`
* `Date.utc_today()`

Always use `EctoHelpers.current_datetime_from_db/0` or `/1` through helper functions defined on the resource.

### Required pattern

```elixir
defmodule MyApp.MyResource do
  use Ash.Resource

  def get_current_datetime do
    alias Lynxweb.Utilities.Agnostic.EctoHelpers
    {:ok, _date, _time, datetime} = EctoHelpers.current_datetime_from_db()
    datetime
  end

  def get_current_date do
    alias Lynxweb.Utilities.Agnostic.EctoHelpers
    {:ok, date, _time, _datetime} = EctoHelpers.current_datetime_from_db()
    date
  end

  def get_current_time do
    alias Lynxweb.Utilities.Agnostic.EctoHelpers
    {:ok, _date, time, _datetime} = EctoHelpers.current_datetime_from_db()
    time
  end
end
```

Then use those helpers in actions.

```elixir
actions do
  create :create do
    accept [:field1, :field2]
    change set_attribute(:created_at, &__MODULE__.get_current_datetime/0)
  end

  update :update do
    accept [:field1, :field2]
    change set_attribute(:updated_at, &__MODULE__.get_current_datetime/0)
  end
end
```

## Data layer rules

Use the appropriate data layer intentionally.

Examples:

* `AshPostgres.DataLayer` for Postgres-backed resources
* `Ash.DataLayer.Ets` for in-memory patterns
* `Ash.DataLayer.Mnesia` when explicitly required
* `data_layer: :embedded` for embedded resources
* default simple data layer for non-persisted behavior resources

Do not choose a data layer mechanically; choose it based on persistence and usage requirements.

## Migration rules

After creating or modifying Ash code, run code generation to keep derived artifacts aligned.

Prefer:

* `mix ash.codegen --dev` during active development
* final named codegen with a lower_snake_case migration name

Do not leave resource changes unreflected in generated artifacts.

## Testing rules

When testing Ash-backed behavior:

* test through domain code interfaces
* use `Ash.Test` utilities where appropriate
* test authorization with `Ash.can?`
* use `authorize?: false` only when authorization is not the test subject
* prefer raising `!` variants over manual tuple pattern matching when the operation should succeed
* use globally unique values for identity fields in concurrent tests

### Good

```elixir
%{
  email: "user-#{System.unique_integer([:positive])}@example.com",
  username: "user_#{System.unique_integer([:positive])}"
}
```

### Bad

```elixir
%{email: "test@example.com", username: "testuser"}
```

The bad version can cause deadlocks or flaky failures in concurrent tests if those fields participate in identities.

## Common anti-patterns

Do not:

* treat Ash resources as mere schema declarations with all real logic elsewhere
* default to Ecto queries before checking Ash.Query capabilities
* duplicate validations in multiple places without reason
* create generic helper functions for behavior that deserves a named action
* overload a resource with UI-only concerns
* confuse domain boundaries with technical folder structure
* use code interfaces blindly without understanding the underlying action
* use `authorize?: false` as a casual shortcut in production logic
* call `Ash.get!/2`, `Ash.load!/2`, or similar directly from LiveViews/controllers when a code interface should own the contract
* use raw system-clock date/time helpers directly in this project

## Output expectations when this skill is used

When helping with an Ash task, structure the answer like this:

1. **What the domain problem is**
2. **What Ash construct should own the solution**
3. **Why that placement is correct**
4. **What code interface or public API should expose it**
5. **Minimal implementation shape**
6. **What to avoid**
7. **Any tradeoffs or edge cases if relevant**

## Response rules

* Be concrete and implementation-oriented
* Prefer the Ash-native approach first
* Explain placement decisions clearly
* Prefer domain code interfaces as the public contract
* Keep business logic in actions, validations, changes, preparations, calculations, aggregates, and policies when it belongs there
* Do not invent framework behavior that is not grounded in Ash concepts
* If multiple Ash constructs could work, recommend the most maintainable one
* If the request is too broad, narrow it to the correct Ash abstraction first
