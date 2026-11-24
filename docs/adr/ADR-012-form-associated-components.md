# ADR-012: Form-associated custom elements

**Date:** 2026-11-24

**Status:** Accepted

**Related:** [ADR-004][adr-004] (client size)

## Context

A component author who wants to build a form control (a custom input,
a rich text field, a rating widget) cannot make it a real form control
with microtags. The platform way is a form-associated custom element:
the class sets a static `formAssociated = true`, each host gets an
`ElementInternals` from `attachInternals()`, and the browser wires it
into the containing form for submission, validation, and disabled
state. Microtags calls `customElements.define` inside `setup()` with
no way to set the static first, and does not expose
`ElementInternals` to the setup callback.

The first consumer is `markdown-area`, a `<textarea>` replacement.

## Decision

A new builder link, `.formAssociated()`, opts a component into the
platform's form-associated custom element behaviour. It takes no
arguments.

- **Builder link.** `.formAssociated()` returns a new builder with an
  internal `formAssociated` flag. The flag propagates through all
  subsequent links. It composes with `withProps`, `withRefs`,
  `withContexts` and `render` in any order.

- **Static.** The factory sets `static formAssociated = true` on the
  generated class before the builder calls `customElements.define`,
  so the browser reads it at define time.

- **Internals.** A form-associated host calls `attachInternals()`
  once, in its constructor. The result is kept on the instance and
  reused across disconnect/reconnect cycles. Setup receives it as
  `ctx.internals`.

- **Setup context.** For a form-associated component the setup context
  gains `internals:ElementInternals`. Components that did not opt in
  get a type error when they read `ctx.internals`.

- **Form lifecycle callbacks.** The class installs
  `formResetCallback`, `formDisabledCallback`,
  `formStateRestoreCallback` and `formAssociatedCallback` on the
  prototype. Each forwards to handlers registered through the
  `ctx.form(handlers)` method. The `restore` callback is replayed
  when a handler is registered; the others are not (their state is
  readable from the host and `internals`). Calling `ctx.form()` again
  replaces the earlier handlers. Handlers are dropped on teardown.

- **Server render.** `render-support` is unchanged: form association
  has no server-side meaning.

- **Types.** `FormAssociatedContext` is a public type export from
  `microtags`, so declaration emit in consuming packages does not hit
  TS2883. The builder carries a `FormAssoc` type parameter that
  controls whether the setup context includes `internals`.

## Size cost

| Entry | Before | After | Delta |
|-------|--------|-------|-------|
| `microtags` (core, ticket 02) | 4.04 kB | 4.09 kB | +50 B |
| `microtags` (core, ticket 03) | 4.09 kB | 4.23 kB | +140 B |

The ticket 02 delta (50 B) covers the `formAssociated` static, the
`attachInternals()` call in the constructor, and passing internals to
the setup context. The ticket 03 delta (140 B) adds four prototype
callbacks, `ctx.form()` with handler storage, and restore replay.
Both are under the 150 B-per-ticket target.

## Consequences

- Authors can build form controls that submit values, participate in
  constraint validation, follow `disabled` from ancestor fieldsets,
  and hear about form reset and state restore.
- Components that do not opt in pay no runtime cost and no API change.
- The core entry limit rises from 4.1 kB to 4.25 kB.

[adr-004]: ADR-004-client-bundle-size-outranks-ergonomics.md
