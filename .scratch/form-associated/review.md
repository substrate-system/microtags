<!-- redeye:review from=2f282a9c00349e125d97d7c0c8e6f56f1cfede1a to=84ac4d3835a9b794f1a326e7baea4038816b1f95 status=fixed -->
## 2026-11-24 02:17 -- review 2f282a9..84ac4d3 -- 1 of 1 fixed
- [x] `_fp` (pending restore tuple) not cleared on teardown, causing stale replay after reconnect
    src/factory.ts:259 -- teardown clears `_fh` but not `_fp`. A host that received `formStateRestoreCallback` before setup, then disconnects and reconnects, still has the old `_fp`. The new setup cycle's `ctx.form({ restore })` replays the stale state/mode from the previous connection.

### Fix
- attempts: 1 of 4
- effort: medium
- agent exit code: 0
- result: merged

# Review Fix Notes

## Finding: `_fp` not cleared on teardown

**Summary:** `_fp` (pending restore tuple) was not cleared during
teardown in `src/factory.ts`, causing stale replay after reconnect.

**Root cause:** Line 259 cleared `_fh` (form handlers) but not `_fp`
(pending restore state). A host that received
`formStateRestoreCallback` before setup, then disconnected and
reconnected, would still hold the old `_fp` from the previous
lifecycle, and the new setup's `ctx.form({ restore })` call would
replay it.

**Fix:** Added `this._fp = undefined` on the line after
`this._fh = undefined` in the teardown microtask
(`disconnectedCallback`).

**Test:** Added `formLifecycle: pending restore cleared on teardown`
in `test/index.ts`. Uses `withContexts` to delay setup, fires
`formStateRestoreCallback` before the first setup, verifies replay,
then teardown + reconnect and verifies the stale state is NOT
replayed.

**Files changed:**
- `src/factory.ts` -- one line added (clear `_fp` in teardown)
- `test/index.ts` -- one test added

**Gate status:** All steps pass. `test:preact-ssr` has 2 pre-existing
failures unrelated to this change.
---
