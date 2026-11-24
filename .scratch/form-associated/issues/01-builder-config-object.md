# 01: Prefactor: the builder carries its config as one object

**What to build:** `ComponentBuilder` currently takes eight positional
constructor arguments, and every link (`withProps`, `withRefs`,
`withContexts`, `render`) re-passes all of them. Replace them with a
single config object that each link copies with one field changed, so
a later link (`.formAssociated()`, ticket 02) is a one-field change
instead of a ninth argument threaded through every link.

There is no behaviour change. `withProps` must still drop a render
function set earlier in the chain (its types are tied to the old
props), so the copy in `withProps` clears it explicitly.

This ticket is gated on bundle size (ADR-004). Build with the
`build-esm:min` flags and run `npx size-limit`. If the core entry
(`dist/index.min.js`) grows, do not land the change: set this ticket's
status to `wontfix` with the measured numbers in a comment, and ticket
02 threads its flag positionally instead.

**Blocked by:** None (can start immediately)

**Touches:** `src/builders.ts` (`ComponentBuilder`, `define`),
`.size-limit.js` (comment only, if the measurement changes)

**Status:** ready-for-agent

- [ ] Every existing suite passes unchanged: `npm run gate`.
- [ ] `define(tag).render(fn).withProps(...)` still yields a class with
  no `render` static, as before.
- [ ] `define(tag).withProps(...).withRefs(...).withContexts(...)
  .render(fn)` still yields working `render` and `renderParts` statics
  and the same `TAG`, `refs` and `props` statics as before.
- [ ] The core entry measured by `size-limit` is the same size or
  smaller than before the change, and the `.size-limit.js` comment
  records the new measurement.
