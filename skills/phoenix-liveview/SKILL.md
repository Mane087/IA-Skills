---
name: phoenix-liveview
description: >
  Guide for building, reviewing, and refactoring Phoenix LiveView code,
  including routes, live sessions, mounts, events, assigns, components,
  forms, streams, PubSub, and LiveView tests.
  Trigger: Use when the task mentions or contains `use ... :live_view`,
  `live_session`, `on_mount`, `mount/3`, `handle_event/3`, `handle_info/2`,
  `assign`, `stream`, function components, live components, LiveView forms,
  PubSub/Presence updates, or when deciding whether behavior belongs in a
  LiveView, component, context, or message-driven flow.
metadata:
  stack: elixir
  framework: phoenix-liveview
  area: web-ui
---

## Purpose

Use this skill whenever the task involves Phoenix or LiveView application code.

This includes:
- creating or modifying LiveViews
- defining LiveView routes
- working with `live_session` and `on_mount`
- handling `mount/3`, `handle_event/3`, `handle_info/2`, and `render/1`
- structuring `socket.assigns`
- building function components and live components
- wiring forms, changesets, uploads, and validation
- using streams for collections
- handling PubSub or Presence updates
- testing LiveView behavior

## Core mindset

Treat a LiveView as a long-lived stateful server process that:
1. mounts state
2. renders state
3. receives events
4. changes state
5. renders again

Do not think in traditional controller-response terms once the LiveView is connected.
Think in terms of:
- event flow
- state transitions
- socket assigns
- isolated UI responsibilities
- minimal diffs sent to the client

## First analysis step

Before proposing code, identify:

1. Is this a plain Phoenix controller/template problem or a LiveView problem?
2. What state must live in `socket.assigns`?
3. What events change that state?
4. Is this responsibility better handled by:
   - the LiveView itself
   - a function component
   - a live component
   - a context
   - a changeset or embedded schema
   - PubSub / Presence
5. Does the route require authentication or authorization?
6. Should the UI update through assign changes, streams, or message passing?

## Design rules

### 1. Keep the LiveView thin when domain logic grows

A LiveView should coordinate UI state and user interaction.
Do not bury core business logic in `handle_event/3` or `handle_info/2`.

Prefer:
- contexts for application operations
- changesets for change tracking and validation
- pure modules for deterministic domain logic
- components for UI composition

### 2. Put UI state in `socket.assigns`

Use assigns for state that drives rendering.
If a value affects the UI, it should usually be in assigns.

Do not compute unstable values directly in the template if you expect them to update reactively.
LiveView tracks assign changes, not arbitrary function calls inside templates. :contentReference[oaicite:1]{index=1}

### 3. Use `mount/3` only for initial state and setup

`mount/3` should:
- establish initial state
- load required data
- honor auth/session context
- subscribe to PubSub when connected if needed

Do not turn `mount/3` into a dumping ground for unrelated logic.

### 4. Model interactions through explicit events

Use `handle_event/3` for user-triggered actions.
Each event should have a clear purpose and predictable effect on assigns.

Avoid giant event handlers with multiple unrelated branches.

### 5. Prefer composition over giant templates

Use:
- function components for reusable presentational pieces
- live components when a UI region needs its own state or event handling

Do not keep growing one massive LiveView template if the page clearly contains independent conceptual sections. :contentReference[oaicite:2]{index=2}

### 6. Use `live_session` and `on_mount` for shared auth behavior

Group related authenticated LiveViews in a `live_session`.
Use `on_mount` for reusable authentication/authorization setup.

Do not rely only on the initial HTTP pipeline for security-sensitive LiveViews because live navigation can bypass a fresh controller-style request path. :contentReference[oaicite:3]{index=3}

### 7. Use changesets intentionally

For forms:
- use changesets for validation and error presentation
- use embedded schemas when the form is not directly backed by a persisted schema
- keep form validation rules close to the change model

Do not put all validation logic directly in `handle_event/3`.

### 8. Use streams for large or dynamic collections

Prefer streams when the UI manages changing collections and you want efficient updates.

Do not rebuild and reassign large lists casually if stream semantics are a better fit.

### 9. Use PubSub / Presence only for real distributed state needs

Use PubSub when other processes or users must push updates into the LiveView.
Use Presence when tracking connected users or sessions.

Do not introduce PubSub for local page-only state.

### 10. Test behavior, not internals

LiveView tests should verify:
- rendered output
- user interactions
- state-driven UI changes
- distributed updates when relevant

Do not overtest private implementation details of the LiveView module.

## Routing and auth rules

When working with routes:
- use `live` routes for LiveViews
- group related routes into appropriate scopes
- use `live_session` for shared auth and layout concerns
- use `on_mount` to enforce auth and assign shared user/session context

When auth is required:
- protect the HTTP entry route in the router
- also protect the LiveView mount path using `on_mount`

## Component rules

### Function components
Use when:
- the UI is reusable
- there is no local server-side state
- the component is presentational or parameter-driven

### Live components
Use when:
- the component owns a meaningful part of page state
- the component handles its own events
- the component represents a bounded interactive sub-feature

Do not use a live component just because the template is long.
Length alone is not the criterion; state and responsibility are.

## Form rules

For LiveView forms:
- prefer changesets or embedded schemas
- validate on change when the UX requires it
- persist through contexts, not direct Repo calls in the LiveView when a context already exists
- keep form assigns explicit and stable

For uploads:
- treat upload state as UI state
- keep validation and persistence boundaries clear

## Messaging rules

Use `handle_info/2` when:
- the LiveView must react to PubSub
- a child process or component sends a message
- asynchronous work returns results into the LiveView

Do not misuse `handle_info/2` for ordinary user events that belong in `handle_event/3`.

## Performance rules

- Avoid unnecessary assigns churn.
- Avoid recomputing heavy derived data in templates.
- Prefer assigning derived values when they are reused.
- Prefer streams for large changing collections.
- Keep templates declarative and driven by assigns.

## Common anti-patterns

Do not:
- stuff business logic directly into LiveView callbacks
- compute time-sensitive or random values directly in templates and expect reactive updates
- rely only on the browser pipeline for LiveView auth
- use a live component where a function component is enough
- create one giant `handle_event/3` with unrelated responsibilities
- call Repo directly from the LiveView when a context boundary should own the operation
- use PubSub for local-only interactions
- make templates responsible for complex domain decisions

## Output expectations when this skill is used

When helping with a Phoenix LiveView task, structure the answer like this:

1. What layer should own the change
2. What LiveView lifecycle point is involved
3. What state belongs in assigns
4. Whether a component/context/changeset is needed
5. Minimal implementation shape
6. What to avoid
7. Test impact if relevant

## Correct pattern examples

### Example 1: correct basic LiveView flow

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end

  def handle_event("increment", _params, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <section>
      <p>Count: {@count}</p>
      <button phx-click="increment">+</button>
    </section>
    """
  end
end
````

Why this is good:

* state is explicit in assigns
* event handler changes state only
* render stays declarative
* lifecycle is easy to follow

### Example 2: correct auth-protected LiveView route

```elixir
scope "/", MyAppWeb do
  pipe_through [:browser, :require_authenticated_user]

  live_session :require_authenticated_user,
    on_mount: [{MyAppWeb.UserAuth, :require_authenticated}] do
    live "/dashboard", DashboardLive
  end
end
```

Why this is good:

* route is protected at router level
* mount is protected for live navigation too
* auth behavior is centralized and reusable

### Example 3: correct function component extraction

```elixir
defmodule MyAppWeb.ProductComponents do
  use Phoenix.Component

  attr :product, :map, required: true

  def product_card(assigns) do
    ~H"""
    <article class="product-card">
      <h3>{@product.name}</h3>
      <p>{@product.description}</p>
      <span>{@product.price}</span>
    </article>
    """
  end
end
```

Usage:

```elixir
<.product_card product={@product} />
```

Why this is good:

* reusable presentational unit
* no unnecessary server-side state
* keeps the parent template smaller

### Example 4: correct LiveView form flow with changeset

```elixir
defmodule MyAppWeb.ProductFormLive do
  use MyAppWeb, :live_view

  alias MyApp.Catalog
  alias MyApp.Catalog.Product

  def mount(_params, _session, socket) do
    changeset = Catalog.change_product(%Product{})
    {:ok, assign(socket, form: to_form(changeset))}
  end

  def handle_event("validate", %{"product" => params}, socket) do
    changeset =
      %Product{}
      |> Catalog.change_product(params)
      |> Map.put(:action, :validate)

    {:noreply, assign(socket, form: to_form(changeset))}
  end

  def handle_event("save", %{"product" => params}, socket) do
    case Catalog.create_product(params) do
      {:ok, _product} ->
        {:noreply, put_flash(socket, :info, "Product created")}

      {:error, changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end

  def render(assigns) do
    ~H"""
    <.form for={@form} phx-change="validate" phx-submit="save">
      <.input field={@form[:name]} type="text" />
      <.input field={@form[:description]} type="text" />
      <button type="submit">Save</button>
    </.form>
    """
  end
end
```

Why this is good:

* validation lives in the changeset path
* save goes through the context
* form state is stable and explicit
* UI reacts to assign changes cleanly

### Example 5: correct PubSub subscription pattern

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "orders")
  end

  {:ok, assign(socket, orders: [])}
end

def handle_info({:order_created, order}, socket) do
  {:noreply, update(socket, :orders, fn orders -> [order | orders] end)}
end
```

Why this is good:

* subscription only occurs when connected
* external updates flow through `handle_info/2`
* state transition is explicit

## Incorrect pattern examples

### Example 1: bad template computation pattern

```elixir
def render(assigns) do
  ~H"""
  <p>Current time: {DateTime.utc_now()}</p>
  """
end
```

Why this is weak:

* value is not tracked through assigns
* re-render behavior becomes misleading
* state is implicit and unstable

Better approach:

* assign the value in `mount/3` or update it through an event/message path

### Example 2: bad business logic inside event handler

```elixir
def handle_event("checkout", params, socket) do
  order =
    params
    |> validate_everything_inline()
    |> calculate_taxes_inline()
    |> persist_everything_inline()
    |> notify_everything_inline()

  {:noreply, assign(socket, order: order)}
end
```

Why this is weak:

* UI layer owns too much domain logic
* hard to test
* hard to reuse
* error handling becomes tangled

Better approach:

* move business operations into a context or dedicated service module

### Example 3: bad auth assumption for LiveView

```elixir
scope "/", MyAppWeb do
  pipe_through [:browser, :require_authenticated_user]

  live "/admin", AdminLive
end
```

Why this is weak:

* protects the initial HTTP request
* does not define shared mount-time auth behavior for live navigation
* invites inconsistent LiveView auth handling

Better approach:

* wrap authenticated LiveViews in a `live_session` with `on_mount`

### Example 4: bad live component overuse

```elixir
defmodule MyAppWeb.TitleComponent do
  use MyAppWeb, :live_component

  def render(assigns) do
    ~H"""
    <h1>{@title}</h1>
    """
  end
end
```

Why this is weak:

* no stateful responsibility
* unnecessary lifecycle overhead
* should be a function component instead

### Example 5: bad Repo usage from UI boundary

```elixir
def handle_event("save", %{"product" => params}, socket) do
  changeset = MyApp.Catalog.Product.changeset(%MyApp.Catalog.Product{}, params)

  case MyApp.Repo.insert(changeset) do
    {:ok, product} -> {:noreply, assign(socket, product: product)}
    {:error, changeset} -> {:noreply, assign(socket, form: to_form(changeset))}
  end
end
```

Why this is weak:

* bypasses the context boundary
* makes the LiveView harder to evolve
* duplicates data access concerns in the web layer

Better approach:

* call the context function and keep DB orchestration out of the LiveView

## Review checklist

When reviewing Phoenix LiveView code, ask:

* Is the state that drives rendering explicit in assigns?
* Is this logic really UI logic, or should it move into a context/core module?
* Does the route need `live_session` and `on_mount`?
* Should this be a function component or a live component?
* Are forms driven by changesets or embedded schemas correctly?
* Is `handle_info/2` being used only for message-driven updates?
* Are collections better served by streams?
* Is the code testing user-visible behavior rather than internals?

