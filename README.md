# Loop Driven Development

*This repo is a series of posts on how I've been using coding agents lately (June 2026). This page summarises my main thoughts, then several examples of this in practice, and a coda:*

1. *[Loop Driven Development, in practice](01-ldd-in-practice.md) — building a corpus benchmark, decision by decision*
2. *[Grinding chart rendering against Excel](02-ldd-render-fidelity.md) — a domain with no equality assertions*
3. *[Performance work as a loop](03-ldd-perf.md) — same bytes, faster, with the oracle in an unusual place*
4. *[Are codebases learnable?](04-learnable-codebases.md) — the closing post: is writing software so different from training a model?*

## The oracle problem

TDD with coding agents works. [Red-green-refactor](https://simonwillison.net/guides/agentic-engineering-patterns/red-green-tdd/) — write a failing test, make it pass, then clean up — at agent speed is genuinely good.

But the bottleneck shifts. Implementation code is approximately free. Test scaffolding is approximately free. **What is not free is knowing what the test should assert.** The hard problem is no longer "writing the test" or "writing the code". It's: *what is the right test or answer?* This is the test-oracle problem (Howden 1978, Weyuker, Barr et al. 2015) restated for the agent era. An oracle needs to provide authoritative answers about the system and behaviors you're developing, while being **external** (not the system you're testing), **deterministic** (same query → same answer), and **queryable** (able to answer arbitrary new inputs on demand). That's loop-driven development (LDD): pick the feedback source deliberately, and let the agent loop against it.

## Behavior coverage ≠ code coverage

Line, statement, and branch coverage all measure the same thing: did this line execute under some test? You can hit 100% on every one of those metrics and still have most of your system's behavior **undefined** — no test pins what should happen for a given input. The code does *something*; nothing in the suite says it's the right something.

Behavior coverage asks a different question: for the meaningful input scenarios in the domain, **is the expected output pinned where a test can assert?** For an Excel-compatible engine, `SUM("42", TRUE)`, `SUMIF` with empty criteria, a formula with a circular reference, negative zero through `0.0;-0.0`, and date-serial 60 (the 1900 leap-year ghost) are each a distinct *behavior*. Code coverage cares about one function call; behavior coverage cares about all the variations.

Coverage tools cannot tell you which behaviors are missing, because they don't know which behaviors *exist*. That's a domain question — and the oracle problem is: who answers it?

## Not all feedback is equal

Agents improve by looping against feedback. As [Lance Martin puts it](https://x.com/rlancemartin/status/2064397389189071163), rather than directly prompting and steering the model, design loops that let it self-correct in response to environment feedback. So the quality of the loop is the quality of its feedback source. **The holy grail is feedback that is external and queryable on demand.** Two dimensions sort every source:

- **Independence.** Does the feedback share provenance with the work, or does it come from somewhere the work can't reach? An agent grading its own output is student and examiner at once.
- **Queryability.** Can you get an answer for a *new, specific input* on demand — or is the feedback a fixed artifact you can only read?

![Feedback sources plotted by independence and queryability: in-thread self-review is queryable but has zero independence; standards text is independent but static, with mined test suites above it; off-thread review reaches the independent-and-queryable quadrant but well below the live oracle, which alone sits at the top-right corner — the holy grail](images/feedback-2x2.png)

Ranked, worst to best:

1. **In-thread review.** The context that wrote the code grades it. Queryable — you can ask it anything — but with zero independence: [agents asked to evaluate their own work "tend to respond by confidently praising the work — even when the quality is obviously mediocre"](https://www.anthropic.com/engineering/harness-design-long-running-apps). The answers come from the thing being evaluated.

2. **Standards/RFC text.** Less useful than expected. Fully independent — an external authority at last — but static. You cannot ask a spec what your implementation should return for one specific malformed input; you can only read prose and interpret, and the interpreter is the same agent being graded, so dependence sneaks back in through the reading. Spec text also underdetermines: silent on edge cases, drifted from what real implementations do. I learned this firsthand building a headless Excel engine: OOXML (ECMA-376) is one of the most extensive specs in existence, and coding agents made strikingly little productive use of it — while real Excel, queried as an oracle, drove the test suite directly. If the goal is compatibility with a real system, the spec is a lossy proxy for it.

3. **Off-thread review** (fresh thread, subagent, evaluator agent). Independent *context*, same *kind* — and queryable, which turns out to matter more than the residual dependence. Lance Martin reports [a verifier sub-agent tends to outperform self-critique](https://x.com/rlancemartin/status/2064397389189071163) because grading happens in an independent context window; [Anthropic's harness-design work](https://www.anthropic.com/engineering/harness-design-long-running-apps) splits generator from evaluator, GAN-style, and the evaluator catches stub buttons and display-only features the generator believed were done. Still a judgment, not a measurement — no ground truth in the loop, only a second opinion with less motivated reasoning. It is also the best available rung in domains with no scriptable authority at all (design quality, UX feel), where a fresh-context judge approximates an oracle rather than substituting for one.

4. **High-quality test suites.** Independent and *pre-queried*: each case is a frozen question with an executable answer, zero interpretation step. Still static — coverage is fixed by someone else's enumeration; you can't ask about the case they didn't write down.

5. **Oracle to mimic.** Independent, deterministic, *and* queryable: any input you can formulate gets an authoritative, executable answer on demand. The quadrant the other four approximate.

The ordering encodes two lessons. **Measurement beats judgment**: test suites and oracles outrank both forms of review because they return per-input, machine-checkable answers rather than opinions. And **prose authority underdelivers**: a spec text sounds like ground truth, but because it can't answer per-input questions, in practice it does less work in a loop than even a same-kind reviewer in a fresh context — interpretation hands the oracle role straight back to the agent being graded.

## Two techniques

When agent capacity is high, the limiting factor becomes knowing what the system should do. The top two rungs of the ranking give two answers.

### A. Mine a high-quality open-source test suite

Find a project that has already enumerated behaviors in your domain. Use *their* tests as your spec. The agent ports / adapts / regenerates against your implementation. Not a new idea (TCKs, conformance suites, characterization tests over a reference) — what changed is that an agent can do the porting in minutes for niches no human would have bothered with.

**What makes a suite useful:**

1. **Assertions pin behaviors of the system class, not implementation details.** A test that asserts `parse(input) == expected_ast` pins a behavioral invariant. One that asserts "uses an LRU cache of size 100" or "stores tokens in this struct shape" pins one project's choice and dies on reimplementation.

2. **The contract is at the system's external boundary, not at one project's chosen internal seam.** Real conformance suites — test262 ("what JavaScript does"), SQLLogicTest ("what SQL does") — test the system class. False ones validate one project's chosen interface design, useful only if a reimplementer happens to factor things the same way. The class-level behaviors usually live in the project's main test suite, even when those tests look superficially "internal" (host-language coupling, private imports). Polyglot agents absorb that as port cost. Read what's *asserted*; ask whether the contract is at the system's external boundary or at one project's internal seam.

3. **Source language is nearly irrelevant.** Python tests can seed a Rust durable-execution engine; JS tests can seed a C# number-format port; the [CommonMark spec's JSON cases](https://github.com/commonmark/commonmark-spec) already seed Markdown implementations in dozens of languages.

For concreteness: a [JS number-format library's tests](https://github.com/borgar/numfmt/tree/master/test) can be regenerated as parametric cases in any target language, and upstream commits then flow through as detected failures. A grammar parser can be validated by feeding a real-world input corpus to both the implementation and a reference parser and diffing the outputs — no expected outputs need authoring, because the reference provides them. Pure differential.

### B. Query an authoritative oracle programmatically

Returning to what makes an oracle an oracle – it is **external** (not the system you're testing), **deterministic** (same query → same answer), and **queryable** (it answers arbitrary new inputs on demand). External rules out self-checking loops where the agent is both case author and authority. Deterministic rules out non-deterministic references like LLMs and ungrounded generators. Queryable rules out static references — spec texts, fixed case lists — that cannot answer a per-input question they didn't anticipate. Anything that fails any of the three conditions isn't an oracle for LDD's purposes — it's a different kind of artifact and a different problem.

Script the live reference. Capture inputs, ask the oracle, freeze the answers in the repo, assert against the frozen answers in CI.

**What makes an oracle useful:**

1. **Authoritative.** For the domain, its answer *is* the answer. Building an Excel-compatible engine? Real Excel. Building a Postgres-compatible parser? Real Postgres. [DuckDB tests its SQL implementation using SQLLogicTest against reference engines](https://duckdb.org/docs/stable/dev/sqllogictest/intro) — same pattern, in a popular database.

2. **Scriptable, full-surface.** An agent can drive any observable behavior of the reference, not just a curated subset. Canonical full-surface examples: [AppleScript](https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/index.html) on macOS, [COM/OLE Automation](https://learn.microsoft.com/en-us/windows/win32/com/) on Windows, [curl](https://curl.se/) for HTTP services, language bindings that expose the full object model (xlwings for Excel, libpq for Postgres). Narrow gateways like MCP servers or single-purpose REST APIs are usually too restricted — you can only ask what someone else thought to expose.

3. **Slow and expensive is fine.** Each query runs once; the result is frozen in-repo. CI doesn't pay; only refresh runs do.

**The operational artifact is a re-runnable recipe checked in alongside the snapshot.** The script does the live oracle query; the snapshot captures the result; CI asserts against the snapshot. When you need different output, suspect drift, or behavior changes — modify the script and re-run. The snapshot regenerates. The recipe is the provenance, the script is the version pin, the snapshot is the captured output. Nothing else is needed.

**Distance assertions can stand in for equality assertions.** Instead of pinning each behavior by equality (`SUM("42", TRUE) == 43`), assert a distance over the whole output (`pixel_diff(impl, oracle) < 1%`; [Playwright's `toHaveScreenshot({ maxDiffPixelRatio })`](https://playwright.dev/docs/test-snapshots) is a mainstream instance; relative error for floats and edit distance for strings work the same way). Distance is cheaper to set up — one assertion covers the whole output, with no upfront enumeration of which details matter — and it works where clean equality isn't available (anti-aliasing, floating-point accumulation, natural-language outputs). The cost is actionability: a failing diff doesn't say which detail is wrong or what to fix next, and domain invariants stay implicit. Either way the agent gets an aggregate hill-climb signal: % green across an equality suite, or % differing pixels driven down across iterations. (The [chart rendering post](02-ldd-render-fidelity.md) is a full worked example of this, including a gate design that stops the metric going slack.)

**A typical setup:** a snapshot generator takes a list of cases, invokes the oracle programmatically, captures the output keyed by a per-case fingerprint, and writes JSON. The oracle is *never* invoked by the test runner — the CI lane and the developer/refresh lane are decoupled by the snapshot file. The generator supports targeted refresh (`--case`) and full rebuilds (`--refresh-all`). When behavior questions arise, modify the script's case definitions and re-run. *(For an Excel-compatible calc engine, this looks like: formula cases across math/trig, statistical, lookup, text, lambda, date categories; xlwings invokes Excel; results captured per-fingerprint to JSON.)*

## What's actually new in the agent era

Three things shift when both producer *and* consumer of the tests are coding agents:

1. **Cost-benefit inverts.** For a human, querying a real reference for 50 edge cases is half a day of grunt work, so humans skip it; the suite stays anemic; bugs surface in production. For an agent, 50 oracle queries is 90 seconds. **The marginal oracle is approximately free.** This is the actual unlock — not a faster version of the same loop, but a fundamental shift in the equilibrium of how many oracle-derived tests projects converge on.

2. **Oracle, test, and impl in one workflow.** Pre-agent, the workflow had multiple stages: consult the oracle, internalize the answer, later write a test from your mental model, later write the impl. Each handoff lost fidelity — the test ended up reflecting the developer's recollection, not the oracle itself. With agents, all of this collapses to one session: query, freeze the answer as a test, write impl. The test is the oracle's answer, not a recollection.

3. **OSS Mining goes polyglot.** Cross-language test mining used to require a human bilingual in both source and target stacks, so it almost never happened. Agents do it casually: tests stop being "in your stack" and start being behavioral specifications that happen to be written in some stack.

## Where human creativity moves

LDD does not remove the human developer; it moves the work up a level. The creative act is no longer hand-deriving `SUM("42", TRUE)` or copying answers into assertions. It is making four decisions:

1. **Which oracle.** Real Excel via xlwings? CPython's `re` module for a regex port? Postgres for a SQL parser? The choice fixes both the authority you'll cite and the surface you can query.

2. **Which properties to pin.** The oracle produces a lot of observable output; you're declaring which of it counts. Formula result value: yes. Result kind (number vs error vs blank): yes. Floating-point precision to 15 digits: yes, with per-case tolerance. Wall-clock evaluation time, calc-chain order, internal representation: not the oracle's job to dictate, even though Excel "has an answer" for them. Pin everything and the suite is brittle; pin too little and most behavior is left undefined.

3. **What shape the comparison takes.** Per-cell equality on a normalized `(kind, value)` tuple. Pixel distance under a threshold. Edit distance under a budget. The choice determines what a failure looks like — a single mismatched cell, or a 1.7% pixel delta with no pointer at the cause.

4. **When the agent is done.** Not "all tests pass" — that's trivially true of an empty suite. Done is: every behavior dimension you named has cases, the case list covers the feature's input space, the oracle recipe is checked in, the snapshot is reproducible from it, CI asserts without the live oracle, and new behavior cannot land without either matching the oracle or deliberately updating it. Looping coding agents overshoot or stop short unless this is written down somewhere they re-check each turn — Codex's [`/goal` continuation template](https://github.com/openai/codex/blob/6014b6679ffbd92eeddffa3ad7b4402be6a7fefe/codex-rs/core/templates/goals/continuation.md) is one concrete spelling; [`/goal` in Claude Code and Outcomes in Claude Managed Agents](https://x.com/rlancemartin/status/2064397389189071163) are others — Outcomes spawns an independent grader sub-agent that must confirm the rubric is met before the agent may stop, keeping the done-check off-thread.

That is still creative engineering — just creativity over the training environment rather than over each line of implementation. The companion posts in this repo show these four decisions being made on real projects, with the transcript excerpts where each happened: [the corpus benchmark](01-ldd-in-practice.md) walks through all four, [the chart rendering work](02-ldd-render-fidelity.md) is a deep dive on the comparison-shape decision, and [the performance campaign](03-ldd-perf.md) is all four again with the oracle in an unusual place — the current implementation itself.

## Prior art

Every component predates 2026. 

- **Characterization tests** (Feathers 2004): use the existing system as its own oracle. LDD = same mechanic, with the system being characterized being external and authoritative.
- **Approval / snapshot testing** (Falco c. 2008, Verify, Jest): identical persistence model. The only distinction: LDD's source of truth is exogenous; snapshot testing's is endogenous ("what the code did last time").
- **Conformance suites** (test262, SQLLogicTest, WPT, TCK): closest organizational analog. SQLLogicTest's completion mode (run prototype against reference engine to populate expected results) is essentially LDD at scale, in 2003. LDD's contribution is making the same pattern casual and in-repo for any niche.
- **Differential testing** (McKeeman 1998, Csmith 2011): technique B is differential testing where the diff happens at planning time and is frozen, not run live in CI.
- **Test oracle problem** (Barr et al. 2015): LDD is "derived oracle" in their taxonomy. Inheriting the framing.

The parts are old and the loop is cheap; what's left to engineer is the feedback — and once you see code this way, it's hard not to ask whether writing software is really so different from training a model.

## Resources

Open-source artifacts from this series:

- [xlsx-corpus-bench](https://github.com/witanlabs/xlsx-corpus-bench) — the benchmark from [the corpus post](01-ldd-in-practice.md): 15,970 real-world workbooks, real Excel as the recalculation oracle, per-file receipts
- [editable-handbooks](https://github.com/nfcampos/editable-handbooks) — the agent-editable memory format from [the closing post](04-learnable-codebases.md)
- [research-log](https://github.com/witanlabs/research-log) — four months of building an LLM spreadsheet agent; the earlier work referenced in the closing post

Other open-source resources mentioned along the way:

- [numfmt's test suite](https://github.com/borgar/numfmt/tree/master/test) and the [CommonMark spec's JSON cases](https://github.com/commonmark/commonmark-spec) — examples of minable suites (technique A)
- [test262](https://github.com/tc39/test262) and [SQLLogicTest as used by DuckDB](https://duckdb.org/docs/stable/dev/sqllogictest/intro) — conformance suites as oracles
- [Playwright's screenshot assertions](https://playwright.dev/docs/test-snapshots) — mainstream distance-assertion tooling
- [Codex's `/goal` continuation template](https://github.com/openai/codex/blob/6014b6679ffbd92eeddffa3ad7b4402be6a7fefe/codex-rs/core/templates/goals/continuation.md) — one spelling of the done-criteria pattern
- [DSPy](https://github.com/stanfordnlp/dspy) — programmatic prompt optimization, for the coda's comparison of fitted artifacts
