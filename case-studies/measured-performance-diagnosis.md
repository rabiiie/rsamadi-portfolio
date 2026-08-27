# Data Grid Performance — Browser-Side Diagnosis

> Sanitized: no proprietary code, credentials or client data.

## Scope

The follow-up module of the FTTH platform is a set of editable data grids: hundreds of thousands of records, 40+ columns, inline editing, grouping, filtering, sorting, sticky columns, optimistic locking. Current load 50-100 concurrent users; target 500.

Reported symptoms: hovering and scrolling felt heavy, row selection took a visible moment to paint, and a heap snapshot was suspected of showing a memory leak.

Server-side capacity and query work for the same grid: [Capacity and Database Performance](capacity-and-database-performance.md).

---

## 1. Heap snapshot: no leak

**Report.** Detached nodes visible in a heap snapshot.

**Finding.** The detached nodes were retained by hot-module-replacement bookkeeping, not by application code. They were absent from a production build.

**Rule adopted.** Profile the build that ships. Development React is unminified, runs additional checks, and `<StrictMode>` double-invokes every render — in development only.

---

## 2. Hypotheses from code reading, and their test results

Four hypotheses were derived from reading the source. All four were falsified by measurement.

| Hypothesis | Test | Result |
|---|---|---|
| Blinking status indicators — seven CSS classes declare `animation: … infinite` | `document.getAnimations()` on the live page | 0 running |
| Expensive `:hover` selectors with four chained `:not()` | DevTools Selector Stats | 13.5 ms of 13,006 ms of style recalculation — 0.1 % |
| A parent component re-rendering twice per interaction | Build without `<StrictMode>` | Development-only double invoke; absent in production |
| Virtualizer broken — 100 rows mounted for 18 visible | Row count in the DOM | Profiling target was a different grid than the one modified |

Reading CSS establishes that a rule exists. It does not establish how many elements it matches.

---

## 3. Defects found by measurement

1. **952 concurrent CSS transitions.** A 0.1 s `background-color` transition was declared on the cell rather than the row. 40 columns multiply it by 40. This, not the animations, produced the reported blinking.
2. **7,332 style invalidations from one selector.** Zebra striping and hover state were applied to cells (`tr:hover td:not(.pinnedCell)`), so hovering one row invalidated every cell in it. Moved to the row.
3. **`will-change` on elements whose animated property was not composited** — one compositor layer per cell with nothing to composite.
4. **A progress indicator animating `width`**, forcing layout every frame. Replaced with `transform: scaleX()`.
5. **Virtualizer scroll-element getter returned the wrong container**, mounting 100 rows instead of 28.
6. **Three props rebuilt on every render**, defeating `React.memo` on every row. Found by logging prop identity changes per row, not by inspection.
7. **`React.memo` comparator omitted the selection flag.** Selection repainted only when an unrelated render was triggered, which produced the reported "catch up on scroll" behaviour.

Item 7 is a correctness defect located inside a performance optimization. The feature passed manual testing; the user's description — *"it doesn't repaint until I scroll"* — is what separated it from general slowness.

---

## 4. Results

Isolation: mouse movement only over the grid — no scrolling, clicks or typing — before and after. Profile proportions are reported rather than absolute milliseconds, as proportions hold between runs on a shared machine.

| Metric | Before | After |
|---|---|---|
| `Recalculate style` (self) | 45.7 % | 16.1 % |
| Rendering | 52.3 % | 36.2 % |
| Main-thread occupancy | 92.7 % | 85.3 % |
| `Event: animationiteration` | 35.0 % | not present |
| Concurrent animations (peak) | 953 | 1 · 15 · 4 |

`Recalculate style` also changed shape: from 7.5 % self / 42.1 % total to 16.1 % self ≈ 16.1 % total. Before, it was downstream of the transitions and the fix belonged upstream; the remaining work is its own.

DOM mutations per interaction, counted with a `MutationObserver` over the mounted rows:

| Interaction | Rows touched | Mutations | Assessment |
|---|---|---|---|
| Ticking a row checkbox | 1 | 3 | Minimum possible |
| Editing a cell, group already active | 2 | 25 | Correct — cell losing focus and cell gaining it |
| Editing the first cell | 28 | 232 | Legitimate — the double-click activates the group, making every visible row editable |

The third row is not a regression. Distinguishing it required isolating the variable: activate the group first, then measure the second cell. Without that step the repaint the feature requires would have been removed as an optimization.

---

## 5. Web Worker removal

**Context.** An internal ADR required computing per-group completeness in a Web Worker to keep it off the main thread. The requirement had never been measured.

**Method.** Bench with the production algorithm, 40 columns, synthetic rows shaped like production rows, medians of 40 samples. Batched, because `performance.now()` is clamped to ~0.1 ms and a single pass returns `0.000 ms`.

| Rows | Main thread | `structuredClone` alone | Persistent worker | Worker per change |
|---|---|---|---|---|
| 25 | 0.014 ms | 0.105 ms | 0.300 ms | 5.600 ms (412×) |
| 100 | 0.034 ms | 0.356 ms | 0.500 ms | 5.700 ms (170×) |
| 1,000 | 0.356 ms | 3.195 ms | 3.800 ms | 9.700 ms (27×) |
| 10,000 | 4.810 ms | 46.840 ms | 55.300 ms | 86.900 ms (18×) |

**Result.** The worker does not win at any measured size. At 10,000 rows computing costs 4.810 ms and serializing the payload costs 46.840 ms.

Transferring data to a worker does not remove the cost from the main thread: the main thread pays the full `structuredClone` before releasing. A worker is profitable when computing exceeds transferring, which a single linear pass cannot satisfy — cloning walks the same structure and additionally allocates to copy it, while the algorithm only reads. Workers apply to superlinear work, to data already resident in the worker, or to payloads eligible for `Transferable` instead of cloning: sorting at scale, joins, file parsing, cryptography.

Reference: one frame is 16.7 ms. 0.014 ms is 0.08 % of one.

**Action.** The ADR was amended from *"use a worker"* to *"only if computing exceeds transferring"*, with both figures required before any move to a worker. The worker files were deleted.

---

## 6. Instrument errors encountered

- `set scrollTop` at 36 % of the profile was the load-generation script, not the application.
- Double renders in development were `<StrictMode>`, initially read as a real double render.
- A 3-second INP measured against the dev server, which does not describe the shipped build.
- The workload script was edited between an "after" and a "before" run, invalidating that comparison. Re-run with the instrument fixed.

---

## 7. Artifacts

A `qa/` directory holding runnable scripts, a microbenchmark harness, and dated measurement write-ups — one per session, with the figures that justified each change. The method lives in the ADR that decided it; the directory holds only what runs and what it produced.

The same treatment was subsequently applied to a second grid — audit history, revert, and the shared components extracted from doing it twice — sequenced after the first grid was correct. The second pass took days rather than weeks because the mechanisms already existed and only the measurements were new.

---

## Open, by decision

- `Commit`, `Hit test` and `Layerize` are now 55 % of the hover profile: composition and paint rather than style calculation. Diminishing returns; not scheduled without a user-reported symptom.
- A 232 ms INP when opening the cell editor is not React. It is a network round-trip fetching permissions before the cell opens. Different layer, tracked separately.

## Tools

Chrome DevTools (heap snapshots and comparison view, Performance panel with self-vs-total time, Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React DevTools Profiler · custom workload scripts and microbenchmark harness · ADRs
