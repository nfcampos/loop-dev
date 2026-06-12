# Performance work as a loop: same bytes, faster

*Third in a series of practical examples of [Loop Driven Development](README.md) (the others: [building a corpus benchmark](ldd-in-practice.md) and [grinding chart rendering against Excel](ldd-render-fidelity.md)). This one is about a performance campaign, which turns out to be LDD with the oracle in an unusual place. Excerpts are verbatim from the Claude Code session and commit messages.*

A 68MB spreadsheet — 12.4 million cells of London Fire Brigade incident data — took our production xlsx service down with an out-of-memory error. The session that followed opened like this:

> this file made xlsx-serve oom yesterday. since then we've removed one bottleneck we found, the xml parser used for sheet parts, replacing it with a streaming parser. i beieve there are other bottlenecks further down the pipeline (think about the sequence of ops starting in open and ending in save, and passing through a representative sample of the major types of read and write, render and lint ops we support). explore this in depth, identify the next bottleneck(s). copy the file to fixtures/stress. consider exploring some of this with smaller files, to avoid spending needless time waiting for multi-second or even multiple-minute ops when they scale with a known quantity of which we have smaller examples

Note that beyond stating the problem, this sets up the economics of the loop itself: add the failing file as a fixture, but iterate on smaller files where the bottleneck still shows, because a loop you wait minutes per turn on is a loop you'll run fewer turns of.

## Where the oracle is

In the corpus bench, the oracle was Excel. In the chart work, it was Excel screenshots. Performance work has a different shape: the specification is "do exactly what the engine does today — same outputs, same behavior — just faster, in less memory." Which means the oracle is the current implementation. Today's engine, run on the same inputs, defines the correct answer for every output the optimized engine produces.

That gives the loop two signals instead of one. A correctness signal that must stay flat: the optimized code has to produce identical results to the unoptimized code. And a performance signal that must move: time and memory against a baseline. Every optimization is judged on both, and the first one is non-negotiable. (Readers of the LDD post will recognize this as characterization testing — Feathers' technique for safely changing code you can't fully specify — with the agent running the loop.)

The interesting question in this kind of campaign is how strong you can make the flat signal, because the standard test suite is not enough. Tests pin the behaviors someone thought to write down; an optimization can change behavior nobody pinned. The strongest available check is byte equality on the artifacts the engine produces, and that became the campaign's standard gate: open each fixture, edit, save, edit, save again, with timestamps pinned so even time can't introduce noise — and require the outputs to be byte-identical between the old and new code, across every part of the zip package. The double save is deliberate: the second save's decisions derive entirely from state reconstructed by the first, so byte equality of save number two validates the whole reconstruction, not just the serializer.

Where byte equality was too blunt, the check got more specific rather than weaker. A new fast path for formatting numbers was brute-forced against the old serializer over the boundary values, ±0.0, NaN, infinities, 1e15, and four million random integral doubles — byte-identical. When a lint change legitimately altered output (collapsing thousands of duplicate diagnostics), the agent machine-checked the entire diff: every surviving diagnostic byte-identical to before, every removed one a subset of a collapsed group.

And one rule sat above all of it: anything that would change output bytes needs explicit sign-off. The agent flagged parallel zip compression as a possible win and noted it "changes output bytes and needs a contract call" — it didn't happen. The one trade that did happen went through the human:

> as long as the dom skeleton doesnt blow up memory on open for v large files i think fine to make it slightly slower

(That was ~200ms of open time on two large fixtures traded for multi-second save savings, and it came with a cap on how much the skeleton may retain, so the trade can't grow unbounded.)

## The harness

The perf harness is a small console tool that runs each operation against a fixture matrix and records duration and allocations per op, with peak memory measured outside the process. Three design choices in it earn their keep:

**Fixtures are tagged by profile.** Each fixture declares what it stresses: a small baseline workbook; a formula-heavy financial model; a dependency-heavy workbook; the 12.4M-cell LFB file tagged `stress, values, save`; a 54MB Monte Carlo simulator tagged `stress, formulas, lint`. Performance work is profile-specific — a value-heavy file and a formula-heavy file have different bottlenecks — so the matrix is the perf equivalent of behavior coverage: every profile that matters has a fixture that represents it.

**Baselines merge by slice.** A focused run refreshes only its own (fixture, operation) entries in the baseline file, so you can iterate on one op without invalidating the rest, and noisy entries get flagged rather than trusted.

**Iterations adapt.** A probe run sizes the iteration count to the operation's duration, so fast ops get statistics and slow ops don't burn the afternoon.

The workflow around the harness is documented in the repo and is exactly a loop: capture baseline, identify targets, then per experiment — implement, run all tests, compare to baseline, and either *keep* (commit, re-capture the baseline, continue) or *discard* (revert, and log what was tried in `PERFORMANCE_LOG.md`, whose header explains its purpose: a record of experiments both kept and discarded, "so we don't repeat work").

## The loop, with re-baselining

A round of the loop looks like: profile (a sampling profiler against the AOT binary), attribute the cost, fix, verify both signals, re-baseline. The attribution step is where the agent earned its keep — each fix landed with a quantified diagnosis up front: recompression was ~37% of a clean save (the fix: rebuild the zip raw from preserved parts; outputs byte-identical, verified with `unzip -t`). ~27% of a dirty save was re-parsing 414MB of XML it then threw away (the fix: a DOM skeleton that skips sheet data). A single commit's message carries the lint trajectory on the LFB file as the fixes stacked: 238s → 190s → 97s → 59s wall, 67.7GB → 24.2GB allocated, peak memory from 8.8GB-and-swapping to 1.32GB.

Re-baselining is what kept the campaign pointed at the right target. Partway through:

> now re-measure all ops on the large fixtures and report back

The re-measure surfaced something no baseline had ever captured: lint on the formula-heavy simulator ran for 37 minutes at full CPU before being aborted. Not a regression — there had never been a baseline for it; fixing the earlier bottlenecks made it observable, and it was a much bigger problem than whatever was next on the list. The campaign redirected (with the loop economics restated in passing):

> lets look at lint formula heavy (but obvs dont wait around 37mins every time you call it)

That lint went from "doesn't terminate in any reasonable time" to 102.5 seconds. This is the perf-work version of a lesson from the corpus bench post: the signal itself needs maintenance. Each bottleneck you remove reshuffles the profile, and a baseline captured before the reshuffle is pointing you at yesterday's problem.

## A discarded experiment

The repo convention of logging discarded experiments got exercised mid-campaign, and the episode is worth retelling because the human's contribution was one sentence long. The profiler showed heavy time in text measurement during column autofit, and the agent hypothesized the measurement cache was thrashing — 10,000 entries bounded, 365,000 distinct strings in the fixture. Plausible. The response:

> does the cache help at all?

So the agent built A/B variants behind environment flags and measured instead of theorizing: cache as-is, 30.3s; cache disabled, 61.2s; eviction disabled, 31.1s. The cache was halving autofit cost; the bound was irrelevant (per-column access locality means entries are reused before eviction); the thrashing hypothesis was simply wrong. The experiment was reverted and logged — the log entry is its own commit — and the corrected diagnosis (fixed per-call overhead building cache keys and taking locks, not eviction) became the next kept commit: autofit 29.7s → 18.6s, allocations 15.3GB → 6.7GB. The lint snapshots, as always, byte-identical.

The same skepticism ran the other way too — at the loop's own instrumentation. Twice, a measurement claim got challenged ("i dont see where you rebuilt the binary?"), and twice the challenge was right: one A/B run was void because the copied binary had silently failed to load its graphics library, making every op fail and the "identical outputs" result meaningless; one false alarm came from a comparison script that treated empty output as success. Both were harness bugs, found because measurement claims got audited like any other claim. In a loop where the agent runs the measurements, "show me the rebuild" is the perf equivalent of the corpus post's "you've definitely confirmed with excel?"

## Results

Against the previous release (v2.47.0, June 9, pre-campaign), on the three large fixtures the harness tracks — the formula-heavy Portfolio simulator, a dashboard workbook, and a dependency-heavy workbook:

| op | Portfolio | parade | OK_HO |
|---|---|---|---|
| save dirty | 110.2s → 12.8s (−88%) | 4.6s → 1.4s (−69%) | 3.4s → 1.5s (−57%) |
| save clean | 28.4s → 6.6s (−77%) | 2.5s → 0.7s (−74%) | 1.7s → 0.34s (−80%) |
| save clean repeat | 30.5s → 2.3s (−92%) | 2.6s → 0.21s (−92%) | 1.6s → 0.35s (−78%) |
| lint | >600s timeout → 102.5s | 14.5s → 11.1s (−23%) | 9.2s → 8.0s (−13%) |
| autofit | ~par | −23% | 35.8 → 11.7ms (−67%) |
| describesheet | ~par | −25% | −38% |

Save allocations dropped 66–93% across the board (Portfolio dirty save: 21.8GB → 6.5GB allocated).

And the headline is the file that started it: v2.47.0 cannot open the LFB file at all — `INTERNAL_ERROR Insufficient memory`, the production OOM, reproduced on a dev machine. The current binary opens it in 7.9s using 2.9GB, and every operation works: setCells 15ms, lint 65s, autofit 19s, dirty save 18s.

All of it byte-identical where bytes were promised, behind the same test suites (11,390 + 4,386 tests green at the end), in a campaign of roughly thirty-five stacked commits over two days.

## Who did what

The agent did all the profiling, all the implementation, all the measurement runs, the A/B verification batteries, and the bookkeeping — including writing its own discard log entries. The human never wrote code and never personally ran a measurement. What the human did do, going by the transcript: framed the problem and the loop economics (the opening message); chose the fixtures and their profiles; picked the next target each round, usually from options the agent proposed; supplied domain hypotheses worth testing (is style uniformity cheap to check, and if so, can autofit measure only the longest value?); forced empiricism when the agent theorized ("does the cache help at all?"); decided when to re-baseline; audited the instrumentation; and made the two calls only an owner can make — which behavior trade is acceptable, and which lever (anything that changes output bytes) stays untouched without a contract decision.

That list is the same list as the other two posts in this series, wearing different clothes: which oracle (the current implementation, held to byte equality), which properties to pin (everything, minus one signed-off trade), what shape the comparison takes (bytes for outputs, a baseline matrix for speed), and when the agent is done (targets exhausted down to the re-baselined profile, gates green, log written). The artifact type changes; the four decisions don't.
