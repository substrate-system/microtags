# 04: Server render and hydration of form-associated components

**What to build:** Opting in to `.formAssociated()` does not change
the server side. `render` and `renderParts` give the same output, and
`render-support` is unchanged, since form association has no meaning
on the server. The build transform still strips `.render()` from a
define chain that includes `.formAssociated()`, in either order.

A server-rendered host of a form-associated component becomes a full
form control when it hydrates, with internals on the setup context and
form handlers working. This covers each way a host hydrates today:

- an upgrade of static server HTML
- adoption by `mount` from `microtags/htm` (browser)
- the empty-host render through a kept render link (ADR-011)

Also confirm that a form-associated component wrapped with `toPreact`
from `microtags/preact` still works as a form control. The adapter
itself is not changed (out of scope).

This ticket is mostly verification. If it turns up a defect in the
earlier tickets' code, fix it here and note it in ADR-012.

**Blocked by:** 03

**Touches:** `test/ssr.ts`, `test/index.ts`, `test/plugins.ts` or
`test/build.ts` (transform fixture), `test/fixtures/`,
`test/preact-browser.ts`, `src/build.ts` (only if the transform needs
a fix)

**Status:** ready-for-agent

- [ ] SSR suite: `render` and `renderParts` of a component with
  `.formAssociated()` give exactly the same output as the same
  component without it.
- [ ] The transform strips the render link from a chain with
  `.formAssociated()` placed both before and after `.render()`, and
  the transformed module still defines a form-associated component.
- [ ] Static server HTML for a form-associated host inside a `<form>`,
  upgraded by defining the component afterwards, contributes its value
  to `FormData` and runs its `reset` handler on `form.reset()`.
- [ ] A form-associated host adopted by `mount` over server markup
  gets `ctx.internals` and its handlers, and contributes to `FormData`.
- [ ] An empty form-associated host with a kept render link is filled
  from `renderParts`, then gets `ctx.internals` and contributes to
  `FormData`.
- [ ] A `toPreact`-wrapped form-associated component rendered inside a
  `<form>` in the browser contributes to `FormData`.
- [ ] `npm run gate` passes.
