# Loop Driven Development, in practice

*In [Loop Driven Development](README.md) we made an argument: when coding agents do the implementation, the engineering that matters moves to the feedback loop, and the human's job comes down to four decisions — which oracle to trust, which properties to pin, what shape the comparison takes, and when the agent is done. This post walks through one real project, decision by decision, with verbatim excerpts from the two Claude Code sessions where it happened. The result is public: [xlsx-corpus-bench](https://github.com/witanlabs/xlsx-corpus-bench) (Apache 2.0).*

Here's the first message of the session that built it:

> how could we quantify what % of the ooxml spec or excel feature set our xlsx product covers, compared to alternatives, like openpyxl, libre office, aspose, epplus, closedxml etc. is this doable without fudging numbers, ie in a way that people wont question it

Notice the question isn't really about tests. It's about credible numbers. Anyone can publish a benchmark where their own product wins; the hard part is building one where the methodology holds up for readers who have every incentive to find the flaw. "In a way that people won't question it" ended up being the constraint behind most of the decisions below.

One session later we had a benchmark that runs **15,970 real-world workbooks (6.8 million formula cells)** through five xlsx engines and compares every recalculated formula cell against what real Excel produces. The session ran for about 39 hours of wall clock (roughly 800 turns), most of it unattended. Two public corpora bracket the space: [FUSE](https://github.com/witanlabs/xlsx-corpus-bench/tree/main/corpus-fuse), 10,544 workbooks crawled from the open web — old, hostile, every producer imaginable — and SpreadsheetBench, 5,426 modern, formula-heavy workbooks from real Excel forum questions. Every number has an external denominator and a binary, automated pass/fail.

The agent built all of it: the corpus plumbing, the Excel automation, the comparators, the reports, the charts. What we contributed is the four decisions. Let's go through them one at a time.

## Decision 1: which oracle

The first version of the bench measured recalculation the obvious way: open each workbook, recalculate, and compare against the values already cached inside the file. Spreadsheet files store the last computed value of every formula, so this looks like free ground truth — millions of expected values, no extra work.

We dropped it as soon as we noticed:

> definition for recalculation is compared to what was in the file, or to what excel would produce, only the 2nd is valuable i think? as we dont know where the file came from, for all i know it was last edited by libreoffice or something

The problem with cached values is provenance. A workbook crawled from the open web might have last been saved by LibreOffice, by a Java library from 2009, or by a tool that never ran the calculation at all. Comparing against those caches doesn't measure "does this engine calculate like Excel"; it measures "does this engine agree with whatever last touched the file." Only the first is worth publishing.

So the harness asks Excel itself, fresh, for every file. [`harness/excel_truth.py`](https://github.com/witanlabs/xlsx-corpus-bench/blob/main/harness/excel_truth.py) stages a copy of each workbook with `fullCalcOnLoad="1"` injected into its `<calcPr>` element (this forces Excel to recompute everything on open instead of trusting the stored values), opens it via AppleScript with links not updated, saves the result, and that recalculated copy becomes the frozen truth. This is slow, but it doesn't matter: truth generation runs once per corpus, and everything downstream compares against the frozen output, offline, forever.

## Decision 2: which properties to pin

The bench pins three behaviors per library, and the [README defines each in a sentence](https://github.com/witanlabs/xlsx-corpus-bench#what-is-measured): does the file *open*, does it *survive open→save→reopen* (where the written output also has to pass a library-neutral structural validation — readable zip, well-formed XML, all relationship targets resolve), and does *recalculation match Excel*. Just as deliberate is what we didn't pin: speed, memory, API design. Excel has no answer for those, so the bench makes no claim about them.

Inside those three metrics is where the human time actually went, on judgment calls that all followed the same rule: when in doubt, make the choice you'd be comfortable defending. Two examples from the transcript.

The agent proposed skipping formulas that reference external workbooks. Reasonable on the surface — the bench can't open files it doesn't have. But that's not how Excel handles them:

> im not sure this is right, as there should be cached values for the external refs, which should be used by good engines to compute those formulas

Excel uses the cached values of external references it can't resolve, so an engine that does the same can be judged on those cells. Excluding them would have quietly excused real failures, and it's exactly the kind of hole a critical reader goes looking for.

Later the agent went the other way and started implementing filters that would only ever be halfway houses — incomplete exclusions that, however well intentioned, could come across as cherry-picking:

> [Request interrupted by user]
> actually lets leave this as is, i dont want heuristics or exclusions that might be deemed dubious

One exclusion did survive, and the reason it survived is the point: it can be defended in a sentence. Some corpus files were written by generator tools that cache an empty string for formulas they never evaluated, which is indistinguishable from a genuine empty-string result. Judging an engine against those cells would count the file's staleness as the engine's failure. The comparator's docstring in [`harness/cached_values.py`](https://github.com/witanlabs/xlsx-corpus-bench/blob/main/harness/cached_values.py) carries the full argument, and cells the bench can't judge are reported, not judged.

The same rule settled the denominator question:

> when we then compute stats for each of the runners we should fully exclude files excel cant open from both numerator and denominator, we must use exact same file set for all tools being tested

This is now a load-bearing comment in [`harness/report.py`](https://github.com/witanlabs/xlsx-corpus-bench/blob/main/harness/report.py): one universe for every tool — the files Excel itself could process — excluded from numerator and denominator of every metric, for every library. No engine gets a friendlier denominator than any other.

## Decision 3: what shape the comparison takes

Per-cell, binary, equality with a tolerance. The entire comparison policy fits in one function, shared by every engine:

```python
def values_match(a, b):
    """Shared comparator: numbers/bools with 1e-9 relative tolerance,
    strings exact, errors exact by code."""
    (ka, va), (kb, vb) = a, b
    num_a, num_b = ka in ("n", "b"), kb in ("n", "b")
    if num_a and num_b:
        tol = max(1e-9, abs(va) * 1e-9)
        return abs(va - vb) <= tol
    if ka != kb:
        return False
    return va == vb
```

Numbers get a 1e-9 relative tolerance because floating-point accumulation legitimately differs between engines, and pinning the 16th digit would punish implementation order rather than correctness. Strings and error codes get exact equality, because there's no legitimate reason for `#VALUE!` and `#REF!` to disagree. And the fact that this is *one* function matters as much as its contents: every engine is judged by the same dozen lines.

Aggregation gets reported two ways, because they answer different questions. *Percent of workbooks with zero mismatched cells* is the user's question: is my file right, yes or no? *Percent of all formula cells matching* is the engineering signal: how much behavior is correct? A workbook with one wrong cell out of ten thousand fails the first metric and barely moves the second, so publishing both keeps either one from flattering anybody. Underneath the aggregates, every file gets a JSONL receipt, so any number in the README can be traced down to the exact cells that disagreed.

## Decision 4: when the agent is done

We never asked the agent to decide when it was finished. Done-ness lived in files: the bench writes its failures into checked-in reports — `WITAN-FAILURES.md`, `WITAN-RECALC-GAPS.md` — that are enumerated and prioritized. The second session, this time in the engine repo, opened by handing the agent one of them:

> in a new worktree, investigate these failures ~/dev/xlsx-corpus-bench/results-fuse/WITAN-FAILURES.md report back on your findings, ideally we want to handle all of these as gracefully as possible, if need be use excel oracle

Scope was fenced explicitly, rather than trusting the agent to stop in the right place:

> proceed with the others in suggested priority, start a branch for this, atomic commits, add tests as well, stop when you get to the perf tracks

And when the agent claimed a behavioral subtlety — that `COUNTIFS` and related functions ignore cached external-reference values where most functions use them — it didn't get taken at its word:

> you've definitely confirmed with excel that countifs etc are different from most functions in that they ignore cache for external refs?

This habit is worth copying. An agent in this loop is investigator and implementer at once, and the failure mode is subtle: it reasons its way to what Excel probably does instead of asking. The fix is cheap — ask for the receipt. The oracle is right there.

## Closing the loop

Everything above is the first half of LDD: producing the signal. The second session — a long one in the engine repo, working through the failure reports — is what the signal is for. It didn't go the way we expected, and the way it actually went is more instructive.

The load failures were the straightforward part. The agent worked through the checked-in report, fixed everything (it had already caught a false positive in the harness on its way in: "the harness bug is being fixed by someone else, thanks for the catch"), and the published tables now show 100.0% of both corpora opening and surviving round-trip.

The recalculation report was the interesting one. It listed 281,941 mismatched cells, and the agent's investigation collapsed them into eight root causes — of which the single biggest wasn't an engine bug at all. The bench was running the engine under a different locale than the machine that generated the Excel ground truth, and the largest "function bug" groups in the report turned out to be cascades of that one mismatch: on the order of 128,000 cells that came down to whether `31/12/2023` parses as a date. The old truth data had problems of its own and was regenerated. With the locale pinned and the truth rebuilt, the real engine gap was 10,604 cells across 235 files — about 4% of the original number. The other 96% was measurement error. The LDD post says a bad oracle is worse than no oracle because it sends work in the wrong direction; here is that sentence as 271,000 cells. Debugging the signal *is* part of the loop, and it has to come first.

The residual was genuine engine work, and the agent fixed it in priority order ("fix all in proposed order, do same as excel does"): lookup functions swallowing `#REF!` errors instead of propagating them, external-workbook references served from cache on some code paths but not others, legacy CSE array formulas, implicit intersection in pre-dynamic-array workbooks, `TEXT` formatting bugs down in the 1900 leap-bug region. One fix was a deliberate degradation: Excel returns `#VALUE!` for `SUMIF`/`COUNTIFS` over closed external workbooks even when cached values exist, so matching Excel meant making the engine strictly less useful on those cells. That's a call the oracle decision makes for you — parity is the contract, even when parity is worse — and it's also why the audit question quoted earlier got asked before the behavior was locked in.

One more thing from that session worth mentioning: it ended with two off-thread review comments on the branch — a legacy-intersection flag silently dropped when formulas move between cells, and lowercase `INDIRECT` column references being rejected — both valid, both fed straight back to the agent. In terms of the LDD post's feedback ranking, that's the independent-reviewer rung running on top of the oracle rung, which is where it belongs.

Where things stand, from the [published tables](https://github.com/witanlabs/xlsx-corpus-bench#results): witan opens and round-trips 100.0% of both corpora, recalculates 95.3% of FUSE and 95.7% of SpreadsheetBench workbooks fully Excel-identically, and matches Excel on 99.7–99.8% of all formula cells. The remaining gap is enumerated in the same reports that drove the last session. That's the queue for the next one.

## The recipe

Strip away the spreadsheet specifics and what's left is a template we expect to reuse:

1. **A public corpus as the denominator.** Behaviors sampled from the wild, not authored by the team being measured.
2. **The real application as oracle, scripted.** Frozen truth generated once, compared against forever.
3. **One comparator, one universe, no dubious exclusions.** Every methodological choice biased toward defensibility, because the numbers are only useful if people can't question them.
4. **Failure reports as the agent's work queue.** Done is enumerated gaps going to zero, re-verified by the same loop that found them.
5. **Everything pinned.** Corpus, truth snapshots, comparator — even [a frozen engine binary](https://github.com/witanlabs/xlsx-corpus-bench/tree/main/engine-frozen), pinned by commit, so any published number reproduces offline.

The total cost was two long agent sessions, most of them unattended. In the LDD post we argued that when oracle queries become approximately free, the equilibrium number of oracle-derived tests shifts dramatically. This is what that looks like in practice: 6.8 million assertions against real Excel, fix sessions whose work arrives pre-enumerated and pre-prioritized, and a loop that will judge every future engine change the same way it judged the last ones.

One last detail: the result charts in the README are [authored and rendered by witan itself](https://github.com/witanlabs/xlsx-corpus-bench/blob/main/harness/make_charts.py). The engine under test draws its own report card — which is fine, because the loop is what established the report card is honest.
