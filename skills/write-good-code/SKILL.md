---
name: write-good-code
description: >
  Guide for writing, generating, reviewing, and refactoring maintainable code
  using practical software design heuristics such as DRY, KISS, YAGNI, SOLID,
  separation of concerns, high cohesion, low coupling, fail fast, and clear naming
  without applying them dogmatically.
  Trigger: Use when the task involves creating new code, refactoring existing code,
  reviewing implementation quality, improving readability or maintainability,
  reducing duplication or coupling, choosing clearer abstractions, or deciding how
  to structure logic so the code stays simple, cohesive, and easy to evolve.
---

# Write Good Code Skill

## Purpose

This skill guides code generation and code review toward maintainable, readable, and practical software.

The goal is not to force every principle into every solution. The goal is to avoid code that is unnecessarily complex, redundant, fragile, ambiguous, over-engineered, or difficult to change.

Use these principles as decision tools, not as absolute rules.

## When to Use This Skill

Use this skill when the user asks to:

* generate new code;
* refactor existing code;
* review code quality;
* simplify duplicated or confusing logic;
* design modules, services, APIs, components, or database-facing code;
* decide whether an abstraction, interface, pattern, helper, service, or layer is justified;
* explain why a piece of code is hard to maintain.

## Core Principle

Prefer code that is:

* clear before clever;
* simple before generic;
* explicit before magical;
* cohesive before fragmented;
* practical before “architecturally impressive”;
* consistent with the existing project before introducing new patterns.

A technically valid solution can still be bad if it is hard to understand, hard to test, hard to change, or solves problems that do not exist yet.

## Code Generation Process

Before writing code, identify:

1. **The actual problem**
   Do not solve adjacent or hypothetical problems unless the user explicitly asks for them.

2. **The current project conventions**
   Follow the existing naming, folder structure, framework style, dependency injection style, error handling style, and testing style when visible.

3. **The minimum useful design**
   Choose the simplest structure that solves the current requirement without blocking reasonable future changes.

4. **The boundaries of responsibility**
   Decide what belongs in UI, domain/business logic, data access, validation, integration, and infrastructure.

5. **The failure modes**
   Identify invalid inputs, missing records, external service failures, permission failures, null/empty values, race conditions, and partial persistence problems.

## Practical Heuristics

### DRY — Avoid Duplicating Knowledge

Avoid duplicating business rules, validation rules, calculations, mappings, constants, permissions, and persistence logic.

Do not remove repetition just because two code blocks look similar. Similar shape is not always duplicated knowledge.

Bad abstraction signs:

* generic function names like `handleData`, `processThing`, or `doAction`;
* boolean flags that radically change behavior;
* helpers that need many optional parameters;
* abstractions created before there is a real second use case;
* shared code that makes each caller harder to understand.

Apply DRY when duplication represents the same decision or rule in more than one place.

### KISS — Keep the Solution Simple

Do not add patterns, layers, factories, adapters, interfaces, or configuration systems unless they solve a real problem.

A simple solution is not careless. It should still be correct, readable, validated, and maintainable.

Prefer a direct function or module when a larger architecture would only add ceremony.

### YAGNI — Do Not Build Hypothetical Features

Do not implement extensibility for providers, modes, flags, roles, strategies, schemas, or workflows that are not currently required.

Avoid phrases like:

* “maybe later we need…”;
* “just in case…”;
* “to make it generic…”;
* “future-proof…” without evidence.

Design so the code can evolve, but do not pay complexity up front without a concrete requirement.

### Single Responsibility

A module, function, class, or component should have one clear reason to change.

Do not mix unrelated responsibilities such as:

* rendering UI;
* querying the database;
* applying business rules;
* formatting external API payloads;
* sending notifications;
* writing audit logs;
* calculating derived values.

Do not over-fragment either. A function does not need to become five files just to look “clean”. Split only when the separation improves understanding, testability, or change isolation.

### Open/Closed Principle

When behavior truly varies by type, provider, operation, or strategy, prefer extension points over long conditional chains.

Do not create plugin-like architecture for a single known case.

Use extension when there is proven variation. Keep direct code when there is only one behavior.

### Interface Segregation

Do not force consumers to depend on methods they do not use.

Prefer small contracts that describe one capability.

Do not create interfaces for everything by default. An interface is useful when there are multiple implementations, test seams, external boundaries, or framework constraints.

### Dependency Inversion

Business logic should not depend directly on infrastructure details when those details are likely to change or need isolation.

Examples of infrastructure details:

* database clients;
* HTTP clients;
* file storage;
* message brokers;
* email/SMS providers;
* framework-specific transport objects.

Use abstractions where they reduce coupling. Do not add abstractions that only rename a concrete class without reducing risk.

### Separation of Concerns

Keep concerns separated by actual responsibility, not just by folder names.

A project can have good-looking folders and still have bad separation if the same function mixes UI state, database queries, business rules, and external calls.

When generating code, avoid placing domain rules inside presentation code unless the framework convention explicitly favors it and the rule is local/trivial.

### High Cohesion / Low Coupling

Group related behavior together. Avoid modules that know too much about unrelated parts of the system.

A module is suspicious if it needs to know:

* database table structure;
* UI labels;
* external API formats;
* permission rules;
* audit behavior;
* and business calculations at the same time.

Reduce coupling by passing focused data, using clear boundaries, and avoiding deep object navigation.

### Principle of Least Astonishment

Names, signatures, and behavior must match expectations.

A function named `getUser` should not update state, send emails, clear cache, or write audit records unless that side effect is explicit in its name or documentation.

Prefer names that reveal intent:

* `buildPaymentRequest` instead of `processData`;
* `markNotificationsAsRead` instead of `updateStatus`;
* `calculateInvoiceTotal` instead of `handleInvoice`.

### Fail Fast, But With Control

Validate required inputs early.

Fail clearly when the system receives invalid data, impossible states, missing configuration, unsupported modes, or inconsistent records.

Do not hide errors with silent defaults unless the default is explicitly part of the business rule.

Fail fast does not mean throwing exceptions everywhere. For expected failures, return controlled errors that fit the project’s error-handling style.

### Convention Over Configuration

Follow the project’s existing conventions unless there is a concrete reason to change them.

Avoid introducing new naming schemes, directory structures, DTO formats, return shapes, or error formats in isolated changes.

Consistency often matters more than personal preference.

### Composition Over Inheritance

Prefer composing behavior through focused functions, modules, traits, protocols, injected dependencies, or small objects instead of deep inheritance trees.

Use inheritance only when the relationship is stable and substitution is valid.

If subclasses constantly override base behavior or disable inherited methods, the abstraction is probably wrong.

### Law of Demeter

Avoid code that reaches deeply into object internals.

Suspicious example:

```ts
order.customer.address.city.name
```

Deep navigation couples the caller to internal structure. Prefer focused methods, DTOs, or mapped values when the structure crosses boundaries.

### Boy Scout Rule

When modifying existing code, leave the touched area slightly better than before.

Acceptable small improvements:

* remove dead code;
* rename confusing variables;
* extract a clear helper;
* simplify a condition;
* remove redundant branches;
* clarify an error message;
* align formatting with project conventions.

Do not use a small change as an excuse to refactor unrelated modules.

## Naming Guidelines

Use names that communicate domain intent.

Avoid:

* one-letter variables outside tiny scopes;
* vague names like `data`, `item`, `obj`, `result`, `manager`, `helper`, `utils` when a clearer name exists;
* abbreviations that are not standard in the project;
* names that describe implementation instead of purpose.

Prefer:

* `activeMembers` instead of `list`;
* `paymentRequest` instead of `payload` when the type is known;
* `authorizedRepresentative` instead of `rep`;
* `buildAuditLogEntry` instead of `makeLog`.

## Function Design Guidelines

A good function should usually:

* have a clear purpose;
* have a name that matches its behavior;
* receive only the data it needs;
* avoid hidden side effects;
* validate important preconditions;
* return a predictable result;
* avoid mixing I/O with pure calculations unless necessary.

Review a function critically when it:

* has many parameters;
* has several boolean flags;
* mixes validation, persistence, formatting, and external calls;
* is hard to name;
* requires a long comment to explain what it actually does;
* changes unrelated things as a side effect.

## Error Handling Guidelines

When generating code, handle errors deliberately.

Consider:

* missing or invalid input;
* unauthorized access;
* missing records;
* duplicate records;
* invalid state transitions;
* external service timeouts;
* database constraint failures;
* partial success scenarios;
* retries and idempotency where relevant.

Do not swallow errors with empty `catch` blocks.

Do not convert all errors into generic messages if the caller needs actionable context.

Do not leak sensitive implementation details to the user.

## Data and Persistence Guidelines

When code touches persistence:

* validate business rules before writing;
* keep transaction boundaries clear;
* avoid partial writes when the operation must be atomic;
* avoid loading more data than needed;
* avoid duplicating database rules in multiple layers unless necessary;
* respect existing schema constraints and indexes;
* handle concurrency where updates can conflict.

Do not assume that frontend validation is enough.

## API and Integration Guidelines

When generating APIs or integration code:

* define explicit request and response shapes;
* validate required fields;
* return consistent error structures;
* avoid exposing internal models directly when they contain irrelevant or sensitive fields;
* isolate external provider payloads from internal domain models;
* handle provider failures and timeouts;
* avoid hardcoding secrets, tokens, URLs, or credentials.

## Test Guidance

Do not add tests just to satisfy a principle. Add tests where they reduce risk.

Prioritize tests for:

* business rules;
* calculations;
* permissions;
* state transitions;
* bug fixes;
* integrations with mocked boundaries;
* edge cases that are easy to break.

For trivial code, manual verification or existing coverage may be enough.

When proposing tests, specify what behavior is being protected, not only which file is being tested.

## Security and Safety Baseline

Generated code should not:

* hardcode secrets;
* log tokens, passwords, private keys, or sensitive personal data;
* trust user input without validation;
* build SQL or shell commands through unsafe string concatenation;
* ignore authorization checks;
* expose stack traces or internal errors to end users;
* store sensitive data unnecessarily.

Prefer secure defaults and explicit validation.

## Review Checklist

Before finalizing generated code or review feedback, check:

* Does the code solve the requested problem and not a larger imaginary one?
* Is there duplicated business knowledge?
* Is the solution simpler than the abstraction being introduced?
* Are names specific and readable?
* Are responsibilities separated enough without unnecessary fragmentation?
* Is the code consistent with project conventions?
* Are errors handled clearly?
* Are edge cases considered?
* Is there unnecessary flexibility?
* Would another developer understand this without asking the original author?
* Would changing this requirement later be localized or painful?

## Warning Signs of Bad Code

Flag these directly when reviewing:

* generic helpers with unclear purpose;
* abstractions with only one caller and no proven variation;
* duplicated validation or business rules;
* functions that do many unrelated things;
* hidden side effects;
* long conditional chains that encode multiple behaviors;
* inconsistent error handling;
* names that hide domain meaning;
* code that depends on internal structure from far away;
* defensive code that masks real bugs;
* comments that explain confusing code instead of making the code clearer.

## Response Style When Applying This Skill

When giving advice:

* be direct about weak design choices;
* explain the tradeoff, not just the rule name;
* avoid saying “this violates SOLID” without explaining the actual maintenance problem;
* prefer concrete replacement code when possible;
* avoid overengineering the answer itself;
* distinguish between “must fix”, “should improve”, and “acceptable for now”.

Use this classification when useful:

### Must Fix

Issues that can cause bugs, data corruption, security problems, broken behavior, or serious maintenance risk.

### Should Improve

Issues that make the code harder to read, test, extend, or debug, but do not immediately break behavior.

### Acceptable for Now

Tradeoffs that are not perfect but are reasonable given the current scope, risk, and project context.

## Anti-Dogma Rules

Do not apply principles mechanically.

* DRY should not create premature abstractions.
* KISS should not justify careless code.
* YAGNI should not block obvious required extension points.
* SOLID should not create unnecessary ceremony.
* Separation of Concerns should not produce empty layers.
* Fail Fast should not replace controlled error handling.
* Composition should not be forced when simple inheritance is clearly appropriate.

The final criterion is maintainability in context, not compliance with slogans.

## Final Output Expectations

When generating code:

* provide complete relevant code, not isolated fragments that depend on unstated helpers;
* avoid assuming invisible functions or modules exist;
* preserve the user’s stack, language, and project conventions;
* explain only the important decisions;
* mention tradeoffs when the design choice is not obvious;
* avoid unnecessary comments in the code unless they clarify non-obvious behavior.

When reviewing code:

* identify the concrete problem;
* explain why it matters;
* propose a practical fix;
* avoid rewriting everything unless the current design is fundamentally wrong.
