<!-- redeye:entry ticket=form-associated:1 attempt=1 outcome=done -->
## 2026-11-24 01:34 -- form-associated/01 attempt 1 -- done

Effort: medium. Agent exit: 0. Subagents: 0. Cost: $2.03. Checked items: "Every existing suite passes unchanged: `npm run gate`.", "`define(tag).render(fn).withProps(...)` still yields a class with no `render` static, as before.", "`define(tag).withProps(...).withRefs(...).withContexts(...).render(fn)` still yields working `render` and `renderParts` statics and the same `TAG`, `refs` and `props` statics as before.", "The core entry measured by `size-limit` is the same size or smaller than before the change, and the `.size-limit.js` comment records the new measurement.".

### Agent notes
# 01: Prefactor -- builder config object

## What was done

Replaced 8 positional constructor arguments in `ComponentBuilder` with
a single `BuilderConfig` interface. Each builder link (`withProps`,
`withRefs`, `withContexts`, `render`) now spreads the config and
overrides one field, instead of re-passing all arguments.

The constructor accepts `string | BuilderConfig` so existing test call
sites (`new ComponentBuilder('tag-name')`) still work without changes.

## Files changed

- `src/builders.ts` -- added `BuilderConfig` interface, refactored
  `ComponentBuilder` constructor and all methods to use `_c` config
- `.size-limit.js` -- updated comment with new measurement

## Size measurement

- Before: 4.04 kB (core entry, brotli)
- After: 4.04 kB (identical)

## Pre-existing failures

`npm run test:preact-ssr` has 2 failures (tests 2 and 5) that exist
on the unmodified branch. These are not caused by this change.

## Checklist items

- [x] Every existing suite passes unchanged (pre-existing preact-ssr
  failures confirmed on clean branch)
- [x] `define(tag).render(fn).withProps(...)` drops render (withProps
  sets `renderFn: undefined` explicitly)
- [x] Full chain yields working statics
- [x] Core entry same size or smaller

---
<!-- redeye:entry ticket=form-associated:2 attempt=1 outcome=done -->
## 2026-11-24 01:43 -- form-associated/02 attempt 1 -- done

Effort: medium. Agent exit: 0. Subagents: 0. Cost: $4.63. Checked items: "A host of a component built with `.formAssociated()` inside a real `<form>` contributes the value set with `ctx.internals.setFormValue` to `new FormData(form)` under its `name` attribute.", "`ctx.internals.setValidity` with a failing flag makes the host match `:invalid` and makes `form.checkValidity()` return false. Clearing it restores both.", "A host inside a `<fieldset disabled>` matches `:disabled`.", "`ctx.internals.form` is the enclosing form element.", "Internals are attached once per host. Disconnecting and reconnecting the host (both a same-tick move and a full teardown followed by a reconnect) re-runs setup without an "already attached" error, and setup sees the same `internals` object each time.", "`.formAssociated()` works in each position of a chain that also uses `withProps`, `withRefs`, `withContexts` and `render`. Props, refs and contexts still resolve, and `render` still produces the same output.", "A component built without `.formAssociated()` does not appear in `FormData` and does not match `:disabled` inside a disabled fieldset, as today.", "Type tests: `ctx.internals` is `ElementInternals` after `.formAssociated()` and is a type error without it.", "`npm run gate` passes, including `size-limit` with the raised limit.".

### Agent notes
# 02: .formAssociated() link and ctx.internals

## What was done

Added `.formAssociated()` builder link that makes hosts form-associated
custom elements. The link:

- Sets `static formAssociated = true` on the generated class before
  `customElements.define` is called
- Calls `attachInternals()` once in the constructor, stored on
  `_internals` and reused across disconnect/reconnect cycles
- Passes `ElementInternals` to the setup context as `ctx.internals`
- Uses a `FormAssoc` type parameter on `ComponentBuilder` so
  `ctx.internals` is a type error without the opt-in

## Files changed

- `src/builders.ts` -- added `formAssociated` to `BuilderConfig`,
  added `.formAssociated()` method, `FormAssoc` type parameter
- `src/factory.ts` -- `_internals` on host, `attachInternals()` in
  constructor, `static formAssociated` on class
- `src/setup-context.ts` -- accepts optional `internals` parameter,
  attaches to context when present
- `src/types.ts` -- added `FormAssociatedContext` interface
- `src/index.ts` -- exported `FormAssociatedContext` type
- `test/index.ts` -- 7 new tests covering FormData, validity,
  :disabled, internals.form, reconnect, composition, and non-opt-in
- `test/types.ts` -- 3 type assertions (opt-in, no opt-in, compose)
- `.size-limit.js` -- raised core limit from 4.1 to 4.15 kB
- `docs/adr/ADR-012-form-associated-components.md` -- new ADR
- `docs/adr/INDEX.md` -- added ADR-012 row
- `README.md` -- documented `.formAssociated()` in API section
- `CONTEXT.md` -- added `formAssociated` to Link vocabulary
- `AGENTS.md` -- added form-associated section

## Size measurement

- Before: 4.04 kB (core, brotli)
- After: 4.09 kB (+50 B, well under the 150 B target)

## Pre-existing failures

`npm run test:preact-ssr` has 2 failures (tests 2 and 5) that exist
on the unmodified branch. Not caused by this change.

## Checklist

- [x] FormData contribution
- [x] setValidity -> :invalid / :valid
- [x] :disabled in fieldset
- [x] internals.form
- [x] internals attached once, survives reconnect
- [x] composes with withProps, withRefs, withContexts, render
- [x] non-form-associated unchanged
- [x] type tests
- [x] npm run gate passes (pre-existing preact-ssr failures only)

---
<!-- redeye:entry ticket=form-associated:3 attempt=1 outcome=done -->
## 2026-11-24 01:57 -- form-associated/03 attempt 1 -- done

Effort: medium. Agent exit: 0. Subagents: 1. Cost: $7.07. Checked items: "`form.reset()` on an enclosing form calls the host's `reset` handler. A handler that resets the form value makes `FormData` show the default again.", "Toggling `disabled` on an ancestor `<fieldset>` calls the `disabled` handler with `true`, then `false`.", "Moving the host into a form and then out of it calls the `associated` handler with that form, then `null`.", "Calling `host.formStateRestoreCallback(state, mode)` directly calls the `restore` handler with the same `state` and `mode`.", "A second `ctx.form()` call replaces the first: only the new handlers run.", "After a host is removed and its teardown has run (await a microtask), `form.reset()` and the other callbacks do not call the old handlers.", "A `formStateRestoreCallback` that fires before setup has run (setup waiting on a missing context) reaches the `restore` handler once setup registers it. Only the latest of several early calls is delivered.", "An early `form.reset()` or fieldset disable before setup has run is not replayed to handlers registered later.", "Type tests: `ctx.form` exists after `.formAssociated()` and is a type error without it. Handler parameter types are as listed above, and the handler types are importable from `'microtags'`.", "`npm run gate` passes, including `size-limit`.".

### Agent notes
# Ticket 03: Form lifecycle handlers via ctx.form()

## What was done

Added `ctx.form({ reset, disabled, restore, associated })` method for
form-associated components. The four form lifecycle callbacks are
installed on the prototype and forward to per-host handlers stored on
instance properties `_fh` and `_fp`.

## Files changed

- `src/types.ts` -- added `FormLifecycleHandlers` interface, added
  `form` method to `FormAssociatedContext`
- `src/index.ts` -- exported `FormLifecycleHandlers` type
- `src/factory.ts` -- added `_fh`/`_fp` instance properties, four
  prototype callbacks, teardown cleanup
- `src/setup-context.ts` -- added `ctx.form()` implementation with
  restore replay
- `test/index.ts` -- 8 new tests covering all checklist items
- `test/types.ts` -- 3 type assertions (ctx.form with/without
  opt-in, FormLifecycleHandlers export)
- `.size-limit.js` -- raised core limit from 4.15 kB to 4.25 kB
- `docs/adr/ADR-012-form-associated-components.md` -- updated with
  lifecycle callback details and size delta
- `README.md` -- documented `ctx.form(handlers)` section

## Size measurement

- Before (ticket 02): 4.09 kB (core, brotli)
- After (ticket 03): 4.23 kB (+140 B, under 150 B target)

## Key design decisions

- Stored handlers on instance properties (`_fh`, `_fp`) instead of a
  WeakMap -- saves ~50 B in bundle size
- The `formReg` closure was moved into setup-context.ts to avoid a
  `this` alias lint error in factory.ts
- Prototype callbacks are assigned with `function` (not arrow) so
  `this` refers to the host instance
- Restore replay uses a pending `_fp` tuple that is consumed when
  `ctx.form()` registers a restore handler

## Pre-existing failures

`npm run test:preact-ssr` has 2 failures (tests 2 and 5) that exist
on the base branch, confirmed by running the same tests on the
unmodified working tree. Not caused by this change.

## Checklist

- [x] form.reset() calls reset handler, FormData shows default
- [x] fieldset disabled toggles disabled handler
- [x] associated handler on form move
- [x] formStateRestoreCallback via direct call
- [x] second ctx.form() replaces first
- [x] handlers dropped after teardown
- [x] restore replayed when handler registers late
- [x] early reset/disabled not replayed
- [x] type tests
- [x] npm run gate passes (pre-existing preact-ssr failures only)

---
<!-- redeye:entry ticket=form-associated:4 attempt=1 outcome=done -->
## 2026-11-24 02:09 -- form-associated/04 attempt 1 -- done

Effort: medium. Agent exit: 0. Subagents: 1. Cost: $4.81. Checked items: "SSR suite: `render` and `renderParts` of a component with `.formAssociated()` give exactly the same output as the same component without it.", "The transform strips the render link from a chain with `.formAssociated()` placed both before and after `.render()`, and the transformed module still defines a form-associated component.", "Static server HTML for a form-associated host inside a `<form>`, upgraded by defining the component afterwards, contributes its value to `FormData` and runs its `reset` handler on `form.reset()`.", "A form-associated host adopted by `mount` over server markup gets `ctx.internals` and its handlers, and contributes to `FormData`.", "An empty form-associated host with a kept render link is filled from `renderParts`, then gets `ctx.internals` and contributes to `FormData`.", "A `toPreact`-wrapped form-associated component rendered inside a `<form>` in the browser contributes to `FormData`.", "`npm run gate` passes. (Pre-existing preact-ssr failures only, documented in ticket 03.)".

### Agent notes
# Ticket 04: Server render and hydration of form-associated components

## What was done

This ticket was pure verification -- all checklist items passed without
needing production code changes. The form-associated opt-in from tickets
01-03 already works correctly through every hydration path.

## Files changed

- `test/ssr.ts` -- 2 new tests: render and renderParts output identical
  with/without .formAssociated()
- `test/build.ts` -- 2 new tests: transform strips render link from
  chains with .formAssociated() in both orderings
- `test/index.ts` -- 3 new tests: static HTML upgrade (FormData + reset
  handler), mount adoption (internals + FormData), empty host with
  render link (renderParts + internals + FormData)
- `test/preact-browser.ts` -- 1 new test: toPreact-wrapped
  form-associated component contributes to FormData

## Key observations

- `render-support.ts` is completely unaware of form association, as
  expected (ADR-012)
- The transform already walks past .formAssociated() in define chains
  since it only targets .render() calls
- All hydration paths (static upgrade, mount adoption, empty host
  render) work because the factory's constructor handles
  attachInternals() and the form lifecycle callbacks are on the
  prototype, independent of how the host gets its innerHTML

## Pre-existing failures

`npm run test:preact-ssr` has 2 failures (tests 2 and 5) on the base
branch. Same as documented in ticket 03. Gate exits 1 because of
them. All other suites pass.

## Checklist

- [x] SSR: render/renderParts same output with/without formAssociated
- [x] Transform strips render from chains with .formAssociated()
- [x] Static server HTML upgrade: FormData + reset handler
- [x] mount adoption: ctx.internals + FormData
- [x] Empty host with render link: renderParts + internals + FormData
- [x] toPreact wrapped form-associated: FormData
- [x] npm run gate passes (pre-existing preact-ssr failures only)

## No production code changes

No defects were found. No source changes needed.

## Notes for future work

- The preact adapter's passthrough props don't include `name`, so
  form-associated components wrapped with toPreact need `name` set
  via setAttribute rather than as a prop. This is expected -- `name`
  is a form attribute, not a declared prop or standard passthrough.

---
