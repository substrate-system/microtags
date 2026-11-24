# 02: `.formAssociated()` link and `ctx.internals`

**What to build:** A component author adds `.formAssociated()` to the
builder chain, and its hosts become form-associated custom elements.
The link takes no arguments. It sets a static `formAssociated = true`
on the component class before `setup()` calls
`customElements.define`. It composes with `withProps`, `withRefs`,
`withContexts` and `render` in any order the chain already allows.

A form-associated host calls `attachInternals()` exactly once, in its
constructor, and keeps the result. Setup receives it as
`ctx.internals`. The author can now call `setFormValue`, call
`setValidity` and set ARIA defaults such as `internals.role`.

A page author then gets the platform behaviour with no extra code. The
control's value appears in `FormData` under its `name`, it counts
towards `:invalid` and `form.checkValidity()`, and it matches
`:disabled` inside a disabled `<fieldset>`.

Components that do not opt in behave exactly as today.

The setup context type includes `internals` only for components built
with `.formAssociated()`. A component that did not opt in gets a type
error when it reads `ctx.internals`.

Write ADR-012 for this feature, covering the builder link, internals
attached in the constructor, the setup context shape, the form
handlers planned in ticket 03 (with restore replay), and the size
cost. Add its row to `docs/adr/INDEX.md`. Measure the core entry
before and after with `size-limit` and record both numbers in the
ADR. The spec's target is under 150 B gzipped added to core. Raise the
core limit in `.size-limit.js` with a comment in the existing style.

Document the link in the README API section and add it to the
**Link** entry in `CONTEXT.md`.

**Blocked by:** 01

**Touches:** `src/builders.ts` (`ComponentBuilder`, new
`formAssociated` link), `src/factory.ts` (`factory`, `FactoryConfig`,
`MicrotagElement` constructor), `src/setup-context.ts`
(`makeSetupContext`), `src/types.ts` (`SetupContext`), `src/index.ts`
(type exports), `test/index.ts`, `test/types.ts`, `.size-limit.js`,
`docs/adr/ADR-012-form-associated-components.md`, `docs/adr/INDEX.md`,
`README.md`, `CONTEXT.md`

**Status:** ready-for-agent

- [ ] A host of a component built with `.formAssociated()` inside a
  real `<form>` contributes the value set with
  `ctx.internals.setFormValue` to `new FormData(form)` under its
  `name` attribute.
- [ ] `ctx.internals.setValidity` with a failing flag makes the host
  match `:invalid` and makes `form.checkValidity()` return false.
  Clearing it restores both.
- [ ] A host inside a `<fieldset disabled>` matches `:disabled`.
- [ ] `ctx.internals.form` is the enclosing form element.
- [ ] Internals are attached once per host. Disconnecting and
  reconnecting the host (both a same-tick move and a full teardown
  followed by a reconnect) re-runs setup without an "already
  attached" error, and setup sees the same `internals` object each
  time.
- [ ] `.formAssociated()` works in each position of a chain that also
  uses `withProps`, `withRefs`, `withContexts` and `render`. Props,
  refs and contexts still resolve, and `render` still produces the
  same output.
- [ ] A component built without `.formAssociated()` does not appear in
  `FormData` and does not match `:disabled` inside a disabled
  fieldset, as today.
- [ ] Type tests: `ctx.internals` is `ElementInternals` after
  `.formAssociated()` and is a type error without it.
- [ ] `npm run gate` passes, including `size-limit` with the raised
  limit.
