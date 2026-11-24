# Form-associated components

**Status:** needs-triage

## Problem Statement

A component author who wants to build a form control with microtags
(a custom input, a rich text field, a rating widget) cannot make it a
real form control. The platform way is a form-associated custom
element: the class has a static `formAssociated = true`, and each host
gets an `ElementInternals` from `attachInternals()`. The static is read
when `customElements.define` runs, and microtags calls `define` inside
`setup()` with no way to set it first. Microtags also does not expose
`ElementInternals` or the form lifecycle callbacks.

Without this, the author's only options are a hidden `<input>` inside
the host, which breaks constraint validation (the browser cannot focus
a hidden invalid control), or abandoning microtags for a hand-written
class.

The first consumer is `markdown-area`, a `<textarea>` replacement
(`../markdown-area`, ADR-016 there).

## Solution

A component opts in with a new step in the builder chain,
`.formAssociated()`. A host of such a component is a form-associated
custom element: it submits a value with its form, takes part in
constraint validation, follows `disabled` from an ancestor
`<fieldset>`, and hears about form reset and state restore. Setup
receives the host's `ElementInternals` on the setup context, and
registers handlers for the form lifecycle callbacks through the setup
context.

Components that do not opt in behave exactly as today.

## User Stories

1. As a component author, I want to mark a component as
   form-associated in the builder chain, so that its hosts become real
   form controls.
2. As a component author, I want the host's `ElementInternals` on the
   setup context, so that I can call `setFormValue`, `setValidity` and
   set ARIA defaults such as `role`.
3. As a component author, I want `ElementInternals` attached exactly
   once per host, so that I never hit the "already attached" error and
   never race it.
4. As a component author, I want a handler for form reset, so that my
   control can return to its default value when the form is reset.
5. As a component author, I want a handler for disabled changes, so
   that my control reacts when an ancestor `<fieldset>` is disabled.
6. As a component author, I want a handler for state restore, so that
   my control can restore its value after back/forward navigation or
   autofill.
7. As a component author, I want a handler for form association
   changes, so that I can react when the host moves into or out of a
   form.
8. As a component author, I want those handlers removed when the host
   disconnects, so that a disconnected host never runs stale handlers.
9. As a component author, I want a restore that arrives before setup
   has registered its handlers to still reach my handler, so that
   restored state is not lost to timing.
10. As a component author, I want TypeScript to know the setup context
    has internals and form handlers only when the component opted in,
    so that misuse is a type error, not a runtime surprise.
11. As a component author, I want the opt-in to compose with
    `withProps`, `withRefs`, `withContexts` and `.render()` in any order
    the builder chain already allows, so that adopting it does not
    reshape my component.
12. As a page author, I want a microtags form control inside a
    `<form>` to appear in `FormData` under its `name`, so that it
    submits like a built-in control.
13. As a page author, I want `form.reset()` to reset a microtags form
    control, so that it behaves like a built-in control.
14. As a page author, I want `:invalid`, `:disabled` and
    `form.checkValidity()` to account for microtags form controls, so
    that styling and validation work as for built-in controls.
15. As a user of a component that does not opt in, I want no change in
    behavior and a negligible change in bundle size, so that the
    feature costs me nothing.
16. As a server-rendering author, I want `render` and `renderParts` to
    work unchanged for form-associated components, so that opting in
    does not affect server render.
17. As a component author, I want a server-rendered host of a
    form-associated component to get its internals and handlers when
    it hydrates, so that server-rendered controls are full form
    controls in the browser.

## Implementation Decisions

- **Builder step.** `.formAssociated()` takes no arguments and returns
  a builder whose component class has a static `formAssociated` set to
  `true` before `setup()` calls `customElements.define`. It is named
  after the platform static so it needs no new vocabulary.
- **Attaching internals.** A form-associated host calls
  `attachInternals()` once, in its constructor, and keeps the result.
  Calling it in the constructor guarantees it happens before anything
  else could, including the form lifecycle callbacks.
- **Setup context.** For a form-associated component the setup context
  gains:
  - `internals`: the host's `ElementInternals`.
  - `form(handlers)`: registers handlers, all optional:
    `reset()`, `disabled(isDisabled:boolean)`,
    `restore(state, mode)`, and `associated(form:HTMLFormElement|null)`.
    Handlers are dropped by setup's cleanups when the host disconnects.
    Calling `form()` again replaces the earlier handlers.
- **Lifecycle callbacks.** The browser reads `formResetCallback`,
  `formDisabledCallback`, `formStateRestoreCallback` and
  `formAssociatedCallback` from the prototype at define time. The
  component class installs all four, and each forwards to the
  registered handler for that host, if any.
- **Callbacks before setup.** A callback can fire before setup has
  registered handlers (setup may wait on provided contexts). The most
  recent `restore` call is kept and replayed when a `restore` handler
  is registered. `reset`, `disabled` and `associated` are not replayed:
  their resulting state is readable at setup time from the host and
  `internals` (`host.matches(':disabled')`, `internals.form`).
- **Types.** The setup context type includes `internals` and `form`
  only for components built with `.formAssociated()`. `ElementInternals`
  and the handler types are public exports, so declaration emit in
  consuming packages does not hit TS2883.
- **Bundle size.** Per ADR-004, size outranks ergonomics. The cost to
  components that do not opt in must be measured with `size-limit`
  and recorded in the ADR. Target: under 150 bytes gzipped added to the
  core entry.
- **Server render.** `render-support` is unchanged: form association
  has no server-side meaning. Statics are unchanged.
- **ADR.** This adds a capability and a setup context shape that other
  code will depend on, so it gets an ADR in `docs/adr/`.

## Testing Decisions

- Tests exercise public behavior only: a component defined with the
  builder chain, placed in a real `<form>` in a real browser, observed
  through platform APIs (`FormData`, `form.reset()`,
  `form.checkValidity()`, `:disabled`, `internals.form`). No test reads
  private fields.
- Seams:
  - The browser suite (the existing tapzero suite run by tapout) for
    the opt-in, internals on the setup context, each lifecycle handler,
    handler removal on disconnect, restore replay, and hydration of a
    server-rendered host.
  - The type tests for "internals and form exist only after opt-in".
  - The bundle-size check (`size-limit`) for the cost to components
    that do not opt in.
  - The SSR suite to confirm `render` output is unchanged by the
    opt-in.
- Prior art: existing browser tests of setup context helpers and
  disconnect cleanup, and the existing type tests.
- State restore is hard to trigger from a test (it needs real
  navigation or autofill). Test it by invoking
  `formStateRestoreCallback` on the host directly. That is a platform
  callback, not a microtags internal.

## Out of Scope

- A higher-level form-control helper (value syncing, built-in
  validators, `name`/`disabled` prop conventions). Authors use
  `internals` directly.
- Any change to the Preact adapter, beyond confirming it does not
  break form-associated hosts.
- Shadow DOM `delegatesFocus` (microtags does not use Shadow DOM,
  ADR-002).

## Further Notes

Requested by `markdown-area` (`../markdown-area/docs/adr/` ADR-016).
That component uses `internals.role = 'textbox'`, `setFormValue`,
`setValidity`, and the `reset`, `disabled` and `restore` handlers.
