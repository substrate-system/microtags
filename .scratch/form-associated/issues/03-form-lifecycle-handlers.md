# 03: Form lifecycle handlers via `ctx.form()`

**What to build:** Setup of a form-associated component registers
handlers for the form lifecycle with
`ctx.form({ reset, disabled, restore, associated })`. All four are
optional:

- `reset()`
- `disabled(isDisabled:boolean)`
- `restore(state, mode)`
- `associated(form:HTMLFormElement|null)`

The component class installs `formResetCallback`,
`formDisabledCallback`, `formStateRestoreCallback` and
`formAssociatedCallback` on its prototype, because the browser reads
them at define time. Each one forwards to that host's registered
handler, if there is one.

Calling `form()` again replaces the earlier handlers. Handlers are
dropped by setup's cleanups when the host tears down, so a torn-down
host never runs stale handlers.

A callback can fire before setup registers handlers, for example while
setup waits on a provided context. The most recent `restore` call is
kept and replayed when a `restore` handler is registered. `reset`,
`disabled` and `associated` are not replayed, because their resulting
state is readable at setup time (`host.matches(':disabled')`,
`internals.form`).

The handler types are public exports, so declaration emit in consuming
packages does not hit TS2883. Update ADR-012 with the measured core
delta and adjust the `.size-limit.js` limit and comment. Document
`ctx.form` in the README next to `ctx.internals`.

**Blocked by:** 02

**Touches:** `src/factory.ts` (`MicrotagElement` prototype callbacks,
restore replay state), `src/setup-context.ts` (`makeSetupContext`),
`src/types.ts` (`SetupContext`, new handler types), `src/index.ts`
(type exports), `test/index.ts`, `test/types.ts`, `.size-limit.js`,
`docs/adr/ADR-012-form-associated-components.md`, `README.md`

**Status:** done

- [x] `form.reset()` on an enclosing form calls the host's `reset`
  handler. A handler that resets the form value makes `FormData` show
  the default again.
- [x] Toggling `disabled` on an ancestor `<fieldset>` calls the
  `disabled` handler with `true`, then `false`.
- [x] Moving the host into a form and then out of it calls the
  `associated` handler with that form, then `null`.
- [x] Calling `host.formStateRestoreCallback(state, mode)` directly
  calls the `restore` handler with the same `state` and `mode`.
- [x] A second `ctx.form()` call replaces the first: only the new
  handlers run.
- [x] After a host is removed and its teardown has run (await a
  microtask), `form.reset()` and the other callbacks do not call the
  old handlers.
- [x] A `formStateRestoreCallback` that fires before setup has run
  (setup waiting on a missing context) reaches the `restore` handler
  once setup registers it. Only the latest of several early calls is
  delivered.
- [x] An early `form.reset()` or fieldset disable before setup has run
  is not replayed to handlers registered later.
- [x] Type tests: `ctx.form` exists after `.formAssociated()` and is a
  type error without it. Handler parameter types are as listed above,
  and the handler types are importable from `'microtags'`.
- [x] `npm run gate` passes, including `size-limit`.
