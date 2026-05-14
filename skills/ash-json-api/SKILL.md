---
name: ash-json-api
description: >
  Guide for exposing Ash resources through AshJsonApi, including domain and
  resource configuration, JSON:API route design, relationship endpoints,
  includes, pagination, filtering, sorting, router setup, and safe public API
  exposure.
  Trigger: Use when the task mentions or contains `AshJsonApi.Domain`,
  `AshJsonApi.Resource`, `json_api do`, JSON:API route definitions such as
  `get`, `index`, `post`, `patch`, `delete`, `related`, `relationship`,
  `post_to_relationship`, `patch_relationship`, `delete_from_relationship`,
  or when deciding how to expose Ash resources and relationships through a
  JSON:API-compliant public API.
metadata:
  stack: elixir
  framework: ash_json_api
  area: api
---

## Purpose

Use this skill whenever the task involves exposing Ash resources through AshJsonApi.

This includes:
- enabling JSON:API on Ash domains and resources
- defining JSON:API route exposure for actions
- building REST-style endpoints from Ash resources
- exposing relationship endpoints
- supporting filtering, sorting, pagination, and includes
- wiring the AshJsonApi router into Phoenix
- designing JSON:API-safe public action surfaces
- reviewing whether a resource is ready to be exposed externally

## Core mindset

AshJsonApi is not a separate business layer.
It is an API projection over your Ash domain model.

Do not design the API first and force the resource to fit it.
Model the resource correctly in Ash first, then expose only the actions and relationships that are safe and intentional.

Treat AshJsonApi as:
- a transport and contract layer
- a JSON:API-compliant projection of your Ash resources
- a public surface that must remain explicit, minimal, and stable

## First analysis step

Before proposing code, identify:

1. Which domain owns the resource?
2. Which resources should actually be public?
3. Which actions are safe to expose?
4. Which relationships should be reachable externally?
5. Which fields should be visible by default?
6. Which includes should be allowed?
7. Does the API require authorization?
8. Does the resource need special relationship mutation routes?
9. Does the resource need a generic custom route instead of plain CRUD?

## Required building blocks

A working AshJsonApi integration typically requires:

1. A domain using `AshJsonApi.Domain`
2. Each exposed resource using `AshJsonApi.Resource`
3. A `json_api` block on the resource with a required `type`
4. Routes defined either:
   - on the domain
   - or on the resource
5. An `AshJsonApi.Router`
6. A Phoenix router `forward`
7. JSON:API MIME type support
8. Optionally `AshJsonApi.Plug.Parser` if file uploads are involved

## Domain rules

Use `AshJsonApi.Domain` on domains that serve JSON:API endpoints.

Example:

```elixir
defmodule MyApp.Blog do
  use Ash.Domain,
    extensions: [AshJsonApi.Domain]

  json_api do
    authorize? true

    routes do
      base_route "/posts", MyApp.Blog.Post do
        get :read
        index :read
        post :create
        patch :update
        delete :destroy
      end
    end
  end

  resources do
    resource MyApp.Blog.Post
    resource MyApp.Blog.Comment
  end
end
````

Use domain-level routes when:

* you want routing grouped in one place
* you want API exposure defined centrally
* you want the domain to act as the API surface owner

Do not put routes in both domain and resource casually for the same public surface unless there is a deliberate reason.

## Resource rules

Every exposed resource must use `AshJsonApi.Resource`.

Each exposed resource must define a JSON:API `type`.

Example:

```elixir
defmodule MyApp.Blog.Post do
  use Ash.Resource,
    domain: MyApp.Blog,
    extensions: [AshJsonApi.Resource]

  attributes do
    uuid_primary_key :id
    attribute :title, :string, allow_nil?: false
    attribute :body, :string
    attribute :published, :boolean, default: false
  end

  relationships do
    belongs_to :author, MyApp.Accounts.User
    has_many :comments, MyApp.Blog.Comment
  end

  actions do
    defaults [:create, :read, :update, :destroy]

    read :list_published do
      filter expr(published == true)
    end

    update :publish do
      accept []
      change set_attribute(:published, true)
    end
  end

  json_api do
    type "post"

    includes [
      :author,
      comments: [:author]
    ]

    default_fields [:title, :body, :published]

    routes do
      base "/posts"

      get :read
      index :read
      post :create
      patch :update
      delete :destroy

      related :comments, :read
      relationship :comments, :read
    end
  end
end
```

## Route design rules

AshJsonApi supports these route families:

* `get` for a single record
* `index` for a collection
* `post` for create actions
* `patch` for update actions
* `delete` for destroy actions
* `related` for loading related resources
* `relationship` for relationship linkage data
* `post_to_relationship` for adding related identifiers
* `patch_relationship` for replacing relationship identifiers
* `delete_from_relationship` for removing relationship identifiers
* `route` for generic custom-action endpoints

Use route families intentionally.

### Good route usage

```elixir
json_api do
  type "post"

  routes do
    base "/posts"

    get :read
    index :read
    post :create
    patch :update
    delete :destroy

    related :comments, :read
    relationship :comments, :read
    post_to_relationship :comments
    patch_relationship :comments
    delete_from_relationship :comments
  end
end
```

Why this is good:

* CRUD exposure is explicit
* relationship reading and mutation are intentional
* JSON:API linkage endpoints are modeled properly

### Bad route usage

```elixir
json_api do
  type "post"

  routes do
    base "/posts"
    get :read
    index :read
    post :create
    patch :update
    delete :destroy
    post_to_relationship :comments
    patch_relationship :author
  end
end
```

Why this is weak:

* relationship mutation is exposed without showing the ownership rules
* the relationship surface is not obviously justified
* author replacement may be too sensitive to expose as a generic public route
* it exposes write capability before proving the resource policy model is ready

## Domain vs resource route placement

### Prefer domain-level routes when:

* the domain is the public API boundary
* you want one place to audit exposure
* multiple resources belong to the same API surface

### Prefer resource-level routes when:

* the resource owns its API exposure clearly
* the project keeps routing close to the resource definition
* the API surface is small and self-contained

Do not split route ownership arbitrarily across both layers unless the project already uses that pattern consistently.

## Filtering, sorting, pagination, and includes

AshJsonApi supports:

* filtering
* sorting
* top-level pagination
* includes
* paginated included relationships

Design guidance:

* expose `index` only when the read action supports the intended query shape
* allow includes intentionally, not reflexively
* prefer stable `default_fields` for public responses
* use `paginated_includes` when relationship payloads can become large

Example resource with includes and paginated includes:

```elixir
defmodule MyApp.Blog.Post do
  use Ash.Resource,
    domain: MyApp.Blog,
    extensions: [AshJsonApi.Resource]

  json_api do
    type "post"

    includes [
      :author,
      comments: [:author]
    ]

    paginated_includes [
      :comments,
      [:comments, :author]
    ]

    always_include_linkage [:author]

    default_fields [:title, :published]
  end
end
```

Example request patterns:

* `GET /posts?filter[published]=true`
* `GET /posts?sort=title,-inserted_at`
* `GET /posts?page[number]=2&page[size]=10`
* `GET /posts?include=author,comments.author`
* `GET /posts/1?include=comments&included_page[comments][limit]=10`

## Relationship route selection rules

Use `related` when:

* you want the actual related resource documents

Use `relationship` when:

* you want relationship linkage data only

Use `post_to_relationship`, `patch_relationship`, or `delete_from_relationship` when:

* the relationship should be mutated via JSON:API resource identifier semantics
* the relationship is part of your intended public contract

Do not expose relationship mutation routes for every relationship by default.

## Custom generic route usage

Use `route` when a meaningful public action does not fit CRUD or relationship mutation directly.

Example:

```elixir
json_api do
  type "post"

  routes do
    base "/posts"

    get :read
    index :read
    post :create
    patch :update
    delete :destroy

    route :post, "/:id/publish", :publish
  end
end
```

Why this is good:

* `publish` is a business action
* it is exposed explicitly
* it avoids pretending the operation is a normal generic update

Bad version:

```elixir
update :publish do
  accept [:published]
end
```

with only a generic `patch :update` route and expecting clients to infer business intent.

Why this is weak:

* domain behavior is hidden behind a generic update
* the API contract is less expressive
* permission and intent become harder to reason about

## Router integration rules

Create a dedicated AshJsonApi router module.

Example:

```elixir
defmodule MyAppWeb.JsonApiRouter do
  use AshJsonApi.Router,
    domains: [MyApp.Blog],
    open_api: "/open_api",
    json_schema: "/json_schema",
    prefix: "/api/json"
end
```

Then forward to it from Phoenix:

```elixir
scope "/api/json" do
  pipe_through :api

  forward "/blog", MyAppWeb.JsonApiRouter
end
```

Do not manually recreate controller logic for the same exposed resources unless you have a very specific reason.

## MIME and parser rules

JSON:API support requires the JSON:API media type.

Example:

```elixir
config :mime,
  extensions: %{"json" => "application/vnd.api+json"},
  types: %{"application/vnd.api+json" => ["json"]}
```

If the project handles file uploads through JSON:API actions, add the AshJsonApi parser:

```elixir
plug Plug.Parsers,
  parsers: [:urlencoded, :multipart, :json, AshJsonApi.Plug.Parser],
  pass: ["*/*"],
  json_decoder: Jason
```

If no file uploads are involved, the parser is optional.

## Authorization rules

If the API should enforce Ash authorization, keep `authorize? true` at the domain level unless there is a specific reason not to.

Do not disable authorization casually just to “make the API work”.
If an endpoint fails under authorization, fix the policy or actor context instead of weakening the surface blindly.

## Correct use cases

### Use case 1: standard CRUD API for a public resource

Use:

* `type`
* `base` or `base_route`
* `get/index/post/patch/delete`
* curated `default_fields`

Good when:

* the resource is a conventional public entity
* CRUD matches business behavior
* read and write actions are already meaningful and safe

### Use case 2: relationship-aware API

Use:

* `related`
* `relationship`
* `includes`
* optionally `paginated_includes`

Good when:

* clients need traversable relationships
* related payloads are part of the API contract
* you want JSON:API-compliant linkage and includes

### Use case 3: business action as endpoint

Use:

* `route`
* or `post`/`patch` mapped to a meaningful custom action

Good when:

* the action has business meaning
* CRUD does not express the workflow clearly
* the public contract should reflect intent explicitly

## Incorrect use cases

### Wrong: exposing every Ash action automatically

Bad pattern:

* adding routes for every action because the framework allows it

Why this is weak:

* expands public attack surface
* exposes internal workflows unnecessarily
* creates unstable external contracts

### Wrong: exposing relationship mutation without policy thinking

Bad pattern:

* adding `patch_relationship` and `delete_from_relationship` everywhere

Why this is weak:

* relationship mutation is often sensitive
* ownership and authorization rules may not be ready
* write semantics become hard to audit

### Wrong: putting API-only hacks into the resource model

Bad pattern:

* distorting a resource action only to satisfy one transport shape

Why this is weak:

* the resource should model domain behavior first
* transport concerns should not degrade domain clarity

## Useful introspection and debugging

To inspect generated routes:

```elixir
MyApp.Blog.Post
|> AshJsonApi.Resource.Info.routes(MyApp.Blog)
```

Use this when:

* checking whether routes were actually generated
* verifying whether domain-level or resource-level routes are effective
* auditing the public JSON:API surface

## Preferred output when this skill is used

When helping with an AshJsonApi task, answer in this order:

1. What should be exposed publicly
2. Whether the routing belongs on the domain or resource
3. Which route family fits the requirement
4. What JSON:API options are needed
5. Minimal implementation shape
6. What should not be exposed
7. Router / MIME / parser implications if relevant

## Response rules

* Be concrete and implementation-oriented
* Prefer explicit API exposure over broad automatic exposure
* Keep the Ash resource as the source of truth for domain behavior
* Keep the JSON:API layer as a deliberate public projection
* Do not invent route types or DSL options
* Do not recommend disabling authorization casually
* If the request is broad, narrow it first to the correct public API surface
