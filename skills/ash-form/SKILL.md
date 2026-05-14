---
name: ash-phoenix-form
description: >
  Guide for building, validating, submitting, and debugging action-backed forms
  with AshPhoenix.Form in Phoenix and LiveView, including nested forms,
  parameter preparation, error transformation, and form lifecycle handling.
  Trigger: Use when the task mentions or contains `AshPhoenix.Form`,
  `for_create`, `for_update`, `for_read`, `for_destroy`, `validate`, `submit`,
  nested form operations like `add_form` or `remove_form`, or when working on
  Ash-backed form flows in LiveView/Phoenix and deciding how validation,
  submission, params, or errors should be handled.
metadata:
  stack: elixir
  framework: ash_phoenix
  area: forms
---

## Purpose

Use this skill whenever the task involves `AshPhoenix.Form`.

This includes:
- building forms for Ash actions
- handling LiveView `phx-change` and `phx-submit`
- using forms in dead views or controllers
- creating forms for create, read, update, or destroy actions
- working with nested forms
- debugging form validation or submission behavior
- customizing errors or submitted params
- deciding whether form logic belongs in `AshPhoenix.Form` options or elsewhere

## Core mindset

`AshPhoenix.Form` is not just a wrapper around `to_form/1`.

It is an action-backed form abstraction over Ash resources and actions.
The form lifecycle is:

1. create the form
2. render it
3. validate it with incoming params
4. submit it
5. on success, continue the workflow
6. on failure, reassign the returned form

Treat the form as the UI projection of an Ash action, not as a free-form params map.

## First analysis step

Before proposing code, identify:

1. Which Ash action is the form targeting?
2. Is this a create, read, update, or destroy flow?
3. Is the form used in LiveView or dead view?
4. Does the form need nested related data?
5. Does the form need custom param preparation?
6. Does the form need custom error transformation?
7. Does the action require actor, tenant, or context?
8. Is the problem actually about the Ash action, not the form wrapper?

## Lifecycle rules

Always follow the standard lifecycle:

1. build the form with `for_create`, `for_update`, `for_read`, `for_destroy`, or `for_action`
2. render with Phoenix form helpers
3. on `phx-change`, call `AshPhoenix.Form.validate/3`
4. on submit, call `AshPhoenix.Form.submit/2`
5. if submit fails, reassign the returned form
6. if submit succeeds, redirect, navigate, flash, or reassign as needed

Do not bypass this lifecycle casually.

## Form state signals

`AshPhoenix.Form` exposes state that is useful in UI logic:

- `submitted_once?` means the form has been submitted at least once
- `just_submitted?` means it was just submitted and has not been revalidated yet
- `changed?` means the form differs from the original state
- `touched_forms` tracks modified keys and only touched keys are included on submit

Use these flags instead of inventing parallel form-state trackers.

## Builder rules

### Prefer explicit builders

Use the most specific builder that matches the action:

- `for_create/3`
- `for_update/3`
- `for_read/3`
- `for_destroy/3`

Use `for_action/3` only when you intentionally want a generic action-backed entry point.

### Good example: create form in LiveView

```elixir
def mount(_params, _session, socket) do
  form =
    MyApp.Accounts.User
    |> AshPhoenix.Form.for_create(:register)
    |> to_form()

  {:ok, assign(socket, form: form)}
end
````

### Better example with code interface-generated form

If the domain/resource uses the `AshPhoenix` extension and code interfaces, prefer the generated `form_to_*` helpers because they make the intended action more discoverable.

```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, form: MyApp.Accounts.form_to_register_with_password() |> to_form())}
end
```

### Bad example

```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, form: to_form(%{}))}
end
```

Why this is weak:

* not action-backed
* loses Ash validation/submission pipeline
* bypasses the actual form abstraction

## LiveView validation rules

For `phx-change`, pass the submitted params to `AshPhoenix.Form.validate/3`.

### Correct pattern

```elixir
def handle_event("validate", %{"form" => params}, socket) do
  form = AshPhoenix.Form.validate(socket.assigns.form, params)
  {:noreply, assign(socket, :form, form)}
end
```

### Incorrect pattern

```elixir
def handle_event("validate", %{"form" => params}, socket) do
  changeset = MyApp.Accounts.change_user(params)
  {:noreply, assign(socket, :form, to_form(changeset))}
end
```

Why this is weak:

* bypasses the original `AshPhoenix.Form`
* breaks the intended lifecycle
* loses form-managed state such as `submitted_once?`, `just_submitted?`, and `touched_forms`

## Submission rules

On submit, call `AshPhoenix.Form.submit/2`.

### Correct pattern

```elixir
def handle_event("submit", %{"form" => params}, socket) do
  case AshPhoenix.Form.submit(socket.assigns.form, params: params) do
    {:ok, user} ->
      {:noreply,
       socket
       |> put_flash(:info, "User registered")
       |> push_navigate(to: ~p"/users/#{user.id}")}

    {:error, form} ->
      {:noreply,
       socket
       |> put_flash(:error, "Please review the form")
       |> assign(:form, form)}
  end
end
```

### Incorrect pattern

```elixir
def handle_event("submit", %{"form" => params}, socket) do
  case MyApp.Accounts.register(params) do
    {:ok, user} -> {:noreply, assign(socket, :user, user)}
    {:error, error} -> {:noreply, put_flash(socket, :error, inspect(error))}
  end
end
```

Why this is weak:

* bypasses the form submission pipeline
* loses structured form errors
* makes the UI harder to keep aligned with action validation

## Actor, tenant, and action context rules

Any additional options passed when building the form are retained and reused during submit.
Use this for things like:

* `actor`
* `tenant`
* `domain`
* other action-building options

Example:

```elixir
form =
  MyApp.Billing.Invoice
  |> AshPhoenix.Form.for_create(:create,
    actor: current_user,
    tenant: current_tenant
  )
  |> to_form()
```

Do not manually re-inject action context in ad hoc ways if the form already owns that context.

## Parameter preparation rules

Use:

* `prepare_source` when you need to modify the underlying changeset or query before validation
* `prepare_params` when you need to normalize raw params before validation
* `transform_params` when you need post-processing before validation/submission

### Good `prepare_source` example

```elixir
AshPhoenix.Form.for_create(MyApp.Post, :create,
  prepare_source: fn changeset ->
    Ash.Changeset.set_argument(changeset, :locale, "es-MX")
  end
)
```

### Good `prepare_params` example

```elixir
AshPhoenix.Form.for_create(MyApp.Post, :create,
  prepare_params: fn params, _phase ->
    Map.update(params, "title", "", &String.trim/1)
  end
)
```

### Bad pattern

```elixir
def handle_event("validate", %{"form" => params}, socket) do
  cleaned = Map.put(params, "status", String.downcase(params["status"]))
  form = AshPhoenix.Form.validate(socket.assigns.form, cleaned)
  {:noreply, assign(socket, :form, form)}
end
```

Why this is weak:

* mixes normalization logic into the event layer
* makes the form pipeline less reusable
* hides param shaping away from the form definition

Prefer putting stable param transformations in form options.

## Error handling rules

Use:

* `transform_errors` for low-level reshaping of source errors
* `post_process_errors` for simpler display-oriented remapping/filtering after conversion to standard triples
* `add_error/3` when you need to inject an external error into the form

### Good `post_process_errors` example

```elixir
AshPhoenix.Form.for_create(MyApp.Transfer, :create,
  post_process_errors: fn _form, _path, {field, message, vars} ->
    case field do
      :amount ->
        {:amount_value, message, vars}

      _ ->
        {field, message, vars}
    end
  end
)
```

### Good `add_error/3` example

```elixir
form =
  socket.assigns.form
  |> AshPhoenix.Form.add_error(%{field: :email, message: "Email already taken"})
```

### Bad pattern

```elixir
{:noreply, assign(socket, :errors, ["Something went wrong"])}
```

Why this is weak:

* creates a parallel error channel
* does not integrate with the form's displayable error model
* weakens consistency with the action-backed form state

## Nested form rules

When working with related or embedded data:

* use `add_form/3` to add nested forms
* use `remove_form/3` to remove them
* use `update_form/4` or `update_forms_at_path/4` for targeted updates
* use `sort_forms/3` when ordering nested collections matters
* use `parse_path!/3` when converting encoded form paths to internal path format

`add_form/3` requires the parent form to have a configured `create_action` and `resource`.

### Good nested add/remove example

```elixir
def handle_event("add_comment", _params, socket) do
  form = AshPhoenix.Form.add_form(socket.assigns.form, [:comments])
  {:noreply, assign(socket, :form, form)}
end

def handle_event("remove_comment", %{"path" => path}, socket) do
  form = AshPhoenix.Form.remove_form(socket.assigns.form, path)
  {:noreply, assign(socket, :form, form)}
end
```

### Bad nested pattern

```elixir
def handle_event("add_comment", _params, socket) do
  comments = socket.assigns.comments ++ [%{}]
  {:noreply, assign(socket, :comments, comments)}
end
```

Why this is weak:

* creates nested UI state outside the form
* breaks alignment between rendered inputs and submitted action params
* makes validation and submit behavior harder to reason about

## Composite and compound type rules

Compound values such as `Ash.Money` may need explicit input decomposition.
Render separate fields for the compound parts and map display errors back to the sub-inputs when needed.

### Good compound input example

```elixir
<.input
  name={@form[:amount].name <> "[amount]"}
  id={@form[:amount].id <> "_amount"}
  value={if @form[:amount].value, do: @form[:amount].value.amount}
/>

<.input
  type="select"
  name={@form[:amount].name <> "[currency]"}
  id={@form[:amount].id <> "_currency"}
  options={[:USD, :EUR, :MXN]}
  value={if @form[:amount].value, do: @form[:amount].value.currency}
/>
```

Use `post_process_errors` when you need to remap compound field errors to concrete rendered inputs.

## Hidden fields and params rules

Use:

* `hidden_fields/1` to render hidden metadata the form needs
* `params/2` when you need the action-ready params for custom handling
* `value/2` to inspect the current value of a field
* `clear_value/2` to clear a field intentionally
* `touch/2` to mark fields as touched when the UI needs that behavior explicitly

Do not manually reconstruct hidden form state if `hidden_fields/1` already provides it.

## Ignore rules

Use `ignore/1` and `ignored?/1` when a nested form or branch should be excluded from submission semantics.
Do not fake “ignored state” by merely hiding DOM elements if the form branch should not be submitted.

## Correct use cases

### Use case 1: standard LiveView create form

* `for_create`
* `validate`
* `submit`
* reassign returned form on error

### Use case 2: update form on existing record

* `for_update`
* action-aware validation and submit
* actor/tenant carried through the form options

### Use case 3: nested relationship editing

* `add_form`
* `remove_form`
* `update_form`
* optionally `sort_forms`

### Use case 4: external or custom form errors

* `add_error`
* `transform_errors`
* `post_process_errors`

## Incorrect use cases

### Wrong: replacing the form abstraction with ad hoc params handling

If the action is Ash-backed and the UI is form-driven, do not bypass `AshPhoenix.Form` and manually juggle params unless there is a very specific reason.

### Wrong: parallel validation state outside the form

Do not keep separate assigns for:

* touched state
* validation messages
* submit flags
  when the form already exposes them.

### Wrong: nesting UI state outside the form tree

If the form owns related/embedded data, do not track it in separate assigns and hope to merge it back later.

## Review checklist

When reviewing an `AshPhoenix.Form` integration, ask:

1. Is the form built from the correct action?
2. Is the standard validate/submit lifecycle being followed?
3. Are actor, tenant, and domain options attached at form creation time?
4. Is param normalization placed in the right hook?
5. Are errors being transformed with form APIs instead of parallel assigns?
6. Are nested forms managed with form helpers instead of ad hoc list state?
7. Is the UI using the form's own state signals such as `submitted_once?` and `changed?`?
8. Is the problem actually in the Ash action rather than in the form integration?

## Preferred response structure

When this skill is used, respond in this order:

1. What action the form should target
2. Which `AshPhoenix.Form` builder or helper should own the solution
3. What lifecycle stage is involved
4. Minimal implementation shape
5. What to avoid
6. Nested/error/param implications if relevant

## Response rules

* Be concrete and implementation-oriented
* Prefer the built-in form lifecycle over custom event pipelines
* Keep the Ash action as the source of truth
* Use form options for reusable param/error customization
* Do not invent unsupported lifecycle steps
* If the requirement is broad, narrow it first to the correct form-action boundary

