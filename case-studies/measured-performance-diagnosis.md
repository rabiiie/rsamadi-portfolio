# Measured Performance Diagnosis — Making a Data Grid Fast Without Guessing

> Sanitized write-up — no proprietary code, credentials, or client data. This one is about method: how the problems were found, not just what was changed.

## Context

The follow-up module of the FTTH platform is a set of editable data grids: hundreds of thousands of records, 40+ columns, inline editing, grouping, filtering, sorting, sticky columns, optimistic locking. Today 50-100 concurrent users; the target is 500.

Users reported that hovering and scrolling felt heavy and that row selection took a visible moment to paint. A heap snapshot was suspected of showing a memory leak.

The interesting part is not the fixes. It's that **most of what looked wrong from reading the code was not the problem**, and one of the problems was a rule I had written myself.

## First finding: the leak wasn't a leak

The detached nodes in the heap snapshot were retained by hot-module-replacement bookkeeping, not by application code. They disappeared entirely in a production build.

**Rule adopted:** profile the build you ship. Development React is unminified and runs extra checks, and `<StrictMode>` double-invokes every render — in development only.

## Four hypotheses from reading the code. All four were wrong.

| Hypothesis, from reading the source | How it was tested | Result |
|---|---|---|
| Blinking status indicators — seven CSS classes declare `animation: … infinite` | `document.getAnimations()` on the live page | **0** of them running |
| Expensive `:hover` selectors with four chained `:not()` | DevTools **Selector Stats** | 13.5 ms out of 13,006 ms of style recalculation — **0.1 %** |
| A parent component re-rendering twice per interaction | Build without `<StrictMode>` | Development-only double invoke; does not exist in production |
| The virtualizer is broken — 100 rows mounted for 18 visible | Counted rows in the DOM | I was profiling a **different grid** than the one I had changed |

Every one of them was a plausible reading of real code. Reading CSS tells you a rule exists; only measuring tells you how many elements it reaches.

## What the measurements actually found

1. **952 concurrent CSS transitions.** A 0.1 s `background-color` transition was declared on the *cell* instead of the *row*. Forty columns multiply it by forty. The perceived "blinking" was this, not the animations.
2. **7,332 style invalidations from one selector.** Zebra striping and hover state were applied to cells (`tr:hover td:not(.pinnedCell)`), so hovering a single row invalidated every cell in it. Backgrounds belong on the row.
3. **`will-change` on elements whose animated property wasn't composited** — a compositor layer per cell with nothing to compose.
4. **A progress indicator animating `width`**, which forces layout on every frame. `transform: scaleX()` does not.
5. **A virtualizer whose scroll-element getter returned the wrong container**, so it mounted 100 rows instead of 28.
6. **Three props rebuilt on every render**, defeating `React.memo` on every row — found by logging prop identity changes per row rather than by inspection.
7. **A `React.memo` comparator that omitted the selection flag.** Selection worked by accident: the row repainted only when something *else* forced a render, which is why users saw it "catch up" on scroll.

Number 7 is the one worth dwelling on. The feature appeared to work. It was a correctness bug hiding inside a performance optimization, and only a user's precise description — *"it doesn't repaint until I scroll"* — separated it from ordinary slowness.

## Results

Isolation experiment: **moving the mouse only** over the grid — no scrolling, no clicks, no typing — before and after. Profile *proportions* are reported, because those hold between runs on a shared machine; absolute milliseconds do not.

| Metric | Before | After |
|---|---|---|
| `Recalculate style` (self) | **45.7 %** | **16.1 %** |
| Rendering | 52.3 % | 36.2 % |
| Main-thread occupancy | **92.7 %** | **85.3 %** |
| `Event: animationiteration` | 35.0 % | not present |
| Concurrent animations (peak) | **953** | **1 · 15 · 4** |

DOM mutations per interaction, counted with a `MutationObserver` over the mounted rows:

| Interaction | Rows touched | Mutations | Reading |
|---|---|---|---|
| Ticking a row checkbox | **1** | 3 | Minimum possible |
| Editing a cell, group already active | **2** | 25 | Correct: the cell losing focus and the one gaining it |
| Editing the **first** cell | 28 | 232 | **Legitimate** — the double-click activates the group, so every visible row becomes editable |

That third row looked like a regression and wasn't. Isolating the variable — activate the group first, *then* measure the second cell — is what distinguished them. Without that step I would have "optimized away" a repaint that the feature requires.

The profile also changed character, not just size: `Recalculate style` went from **7.5 % self / 42.1 % total** to **16.1 % self ≈ 16.1 % total**. Before, it was the victim of a thousand transitions and the fix belonged upstream; now the small amount of work left is genuinely there.

## Deleting a Web Worker that my own architecture decision required

An ADR I had written required computing per-group completeness in a Web Worker, to keep it off the main thread. It had never been measured.

I built a bench with the production algorithm, 40 columns, synthetic rows shaped like the real ones, medians of 40 samples. Batched, because `performance.now()` is clamped to ~0.1 ms — measuring a single pass returns `0.000 ms` and reads as "free".

| Rows | Main thread | `structuredClone` alone | Persistent worker | Worker per change |
|---|---|---|---|---|
| 25 | **0.014 ms** | 0.105 ms | 0.300 ms | **5.600 ms** (412×) |
| 100 | **0.034 ms** | 0.356 ms | 0.500 ms | **5.700 ms** (170×) |
| 1,000 | 0.356 ms | 3.195 ms | 3.800 ms | 9.700 ms (27×) |
| 10,000 | 4.810 ms | 46.840 ms | 55.300 ms | 86.900 ms (18×) |

**The worker never wins.** Not even at 10,000 rows: computing costs 4.8 ms and merely *serializing the payload to send it* costs 46.8 ms.

Sending data to a worker does not take it off the main thread — the main thread pays the full `structuredClone` before letting go. A worker only pays off when computing costs more than transferring, and for a **single linear pass** that cannot happen: cloning walks the same structure *and* allocates memory to copy it, while the algorithm only reads.

Workers earn their keep on superlinear work, or on data that already lives inside the worker: sorting millions of rows, joins, file parsing, cryptography, or anything that can use `Transferable` instead of cloning.

For scale: **a frame is 16.7 ms**. 0.014 ms is 0.08 % of one.

The ADR was rewritten from *"use a worker"* to *"only if computing exceeds transferring"*, with the rule that both numbers must be measured before anything goes into a worker. The worker files were removed.

## Instrument traps

Three, each of which would have led to a wrong conclusion:

- **`set scrollTop` at 36 % of the profile** was my own load-generation script, not the application.
- **Double renders in development** were `<StrictMode>`, taken for a real double render that production doesn't have.
- **Measuring in the dev server.** A 3-second INP there says nothing about the shipped build.

And one process mistake worth recording: I edited the workload script *between* the "after" and "before" runs once, which invalidated the comparison. The instrument does not change mid-experiment.

## What this became

A `qa/` directory in the repository holding runnable scripts, a microbenchmark harness, and dated measurement write-ups — one per session, with the numbers that justified each change. The *method* lives in the ADR that decided it; the folder holds only what runs and what it produced. Duplicating the explanation in two places guarantees one copy rots.

## Still open, deliberately

- `Commit`, `Hit test` and `Layerize` are now 55 % of the hover profile — composition and paint, not style calculation. That is the honest remaining ceiling in the browser: a different problem, with diminishing returns, and not worth attacking without a real complaint.
- A 232 ms INP when opening the cell editor turned out **not to be React at all**: it was a network round-trip fetching permissions before opening the cell. Different layer, different fix.

The other half of this grid's performance — what happens when fifty people use it at once, and what the database is doing under them — is a separate write-up: [Capacity and Database Performance](capacity-and-database-performance.md).

## Where it went after that

The reference grid was closed out and the same treatment carried to a second one — audit history, revert, the shared components that came out of doing it twice. The propagation was deliberately sequenced after the first grid was correct, which is why the second one took days rather than weeks: by then the mechanisms existed and only the measurements were new.

One habit survived the move intact and is worth naming, because it is what made the second pass cheap: **the numbers that justified each change live next to the code they justify.** A change that regresses one of them is detectable rather than arguable.

## Takeaway

Four confident hypotheses, all from reading real code, all false. One architecture decision of my own, reversed by its first measurement.

The habit that produced every finding here is the same one: **measure the variance before you measure the difference, and isolate one variable at a time.** Everything else was tooling.

## Tools

Chrome DevTools (heap snapshots and comparison view, Performance panel with self-vs-total time, Selector Stats, Live Metrics/INP) · `document.getAnimations()` · `MutationObserver` · React DevTools Profiler · custom workload scripts and microbenchmark harness · ADRs as the record of what was decided and why
