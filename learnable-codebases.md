# Are codebases learnable?

*Draft — follow-up to [Loop Driven Development](README.md), which makes the practical case: a coding agent looping against oracle-derived tests. This post asks what that loop is, categorically.*

With coding agents in the loop, is writing code categorically different from training a model or tuning a prompt — or are all three doing the same thing?

## Three optimizers

Three ways to fit a function to a teacher's input → output behavior:

- **Train a model**: optimizer = gradient descent. Artifact = weights. And the experimenter wrapping that optimizer is now often itself a coding agent: in [Parameter Golf](https://x.com/rlancemartin/status/2064397389189071163) — an open ML-engineering challenge to train the best model that fits in a 16MB artifact — the agent edits the training script, launches the run, polls the log, reads the score, and decides the next experiment.
- **Tune a prompt**: optimizer = a search loop. Artifact = a prompt that elicits desired behavior from a base model. The loop went programmatic years ago — [DSPy](https://github.com/stanfordnlp/dspy) compiles prompts against a metric; [GEPA](https://arxiv.org/abs/2507.19457)-style reflective prompt evolution mutates them against one.
- **Write code (LDD)**: optimizer = a coding agent reading test cases. Artifact = an implementation that passes them.

Pre-agent, the third looked categorically different — "writing code" was creative authorship; "training a model" was statistical fitting. With agents in the loop, the substance is the same: an optimizer iterates an artifact toward sampled teacher I/O. The artifact happens to be code.

The strongest evidence that the activities have merged is that one optimizer now performs all three. The same coding agent that hill-climbs a conformance suite will, handed a GPU box, hill-climb a training script — and handed an eval, hill-climb a prompt. The activity is constant; only the artifact varies. (We've lived this: our published [research log](https://github.com/witanlabs/research-log) records one project optimizing prompts, code, and the eval harness itself against a single benchmark.)

## The differences are artifact properties

|  | Weights | Prompt | Code |
| :---- | :---- | :---- | :---- |
| Determinism | No | No | Yes |
| Auditability | Low (weights opaque) | High (read the prompt) | High (read the source) |
| Off-sample behavior | Probabilistic interpolation | Depends on base model | Deterministic for implemented features; absent otherwise |

These rows are not abstractions; each is observable in a working repo. From the same OOXML engine work behind the LDD post's Excel oracle:

- **The unverified region of code is discrete — literally a file you can read.** The repo keeps a checked-in coverage inventory mapping each public mutation and OOXML part to its test surface, with a section for known gaps. You can point at the untrained region. No analogous file can be written for weights: off-distribution behavior isn't enumerable, only sampled.
- **The optimization history is auditable, including the rejected steps.** The performance work logs discarded experiments alongside the kept ones — each entry a hypothesis, a measurement against a fixed fixture matrix, and a keep/discard decision. It's a training curve with the rejected gradient steps preserved and readable.
- **The training distribution is chosen, and the choice is legible.** Vendored forks freeze behavior boundaries explicitly: the spreadsheet library fork names which API surface is in scope and which is deliberately cut; the vendored JavaScript engine ships with the Test262 conformance suite — the artifact carries its own teacher with it.

## When each fits

**Code (LDD) fits when:**

- Determinism and audit are required
- Behavior space is bounded enough to enumerate the meaningful cases

**Weights or prompts fit when:**

- Input space is too large to enumerate (natural language, vision, long-tail UI)
- Approximate generalization matters more than bounded determinism
- The oracle is itself contested or evolving and you'd rather absorb the uncertainty than freeze it

## The judge needs better properties than the artifact

One constraint binds all three: the optimizer is only as good as its loss signal, and the judge supplying that signal needs *better* artifact properties than the thing being optimized. An LLM judging an LLM-built artifact is the worst arrangement in the system — nondeterministic grading nondeterministic, with [a documented bias toward confidently praising its own work](https://www.anthropic.com/engineering/harness-design-long-running-apps). The remedies all move the judge up the property table: [an independent verifier sub-agent beats self-critique](https://x.com/rlancemartin/status/2064397389189071163); a programmatic comparison beats an LLM judgment; a frozen oracle snapshot beats both. The feedback ranking in the LDD post is exactly this: judge selection, ordered by the judge's own determinism and independence.

## Where the experience goes

Gradient descent has a built-in answer to "where does experience accumulate?": the weights. The agent era breaks that — the base model is frozen, and the session that learned something ends. So when a coding agent spends an afternoon discovering some hard truth about its environment, where does the learning go?

Our answer is a fourth fitted artifact — and the research side is converging on the same view: [text optimization deserves to be taken as seriously as weight optimization](https://yoonholee.com/blog/2026/we-should-take-text-optimization-more-seriously/), because a text update serves the same functional role as a weight update, often more sample-efficiently. Ours is the [editable handbook](https://github.com/nfcampos/editable-handbooks) — a skill-shaped directory of atomic markdown entries that agents both read and write. One file per concept, grep as the index, `Related:` lines as the link graph, read-before-edit so every write sees existing terminology. Curated priors and accreted experience coexist in the same files.

The production instance behind our OOXML work has ninety-odd entries, and its densest one is about the oracles themselves: seventy-some accreted bullets of operational knowledge about driving real Excel and PowerPoint programmatically. PowerPoint window captures include 32 logical pixels of app chrome, not the 28 the title bar implies — under-crop and a strip of non-slide pixels biases every diff. Screenshots carry display ICC profiles that must be normalized to sRGB before pixel metrics, or color management dominates the comparison. VBA macro hosts must be registered `.ppam` add-ins, not `.pptm` files. Animated slides must stabilize before capture. Each bullet is a scar from a session where the oracle misled before it taught, distilled so the next session doesn't pay for it again. The entry's hardest-won line: **"a bad oracle is worse than no oracle, because it can send work in the wrong direction."**

Two things make the handbook interesting as an artifact class. First, its properties straddle the table: it influences behavior like a prompt (consulted, not executed — probabilistic), but it audits like code — you can `git blame` an individual belief and see when, and from what evidence, the team's priors changed. Second, it is the prompt row of the table running continuously in production, with the agent as its own prompt engineer: fail, investigate, verify, distill, consult — the [memory progression](https://x.com/rlancemartin/status/2064397389189071163) observed in benchmark settings, operating as ordinary engineering practice.

It also corrects an oversimplification in the LDD post. There, the marginal oracle *query* is approximately free. True — but the oracle *harness* is not. It is a learned artifact with its own training history, and the handbook is where that training persisted. Experience then hardens along a gradient: prose (a handbook entry about the 32-pixel crop) becomes code (the screenshot harness that applies it) becomes data (the frozen snapshots the harness produces). Each conversion trades flexibility for determinism — and the agent performs the conversions itself.

## So: are they learnable?

**Common to all the artifacts:** what wasn't sampled isn't verified. The unverified region is discrete in code (missing tests, unwritten functions); continuous in weights and prompts (off-distribution interpolation); and in a handbook it's simply the entry nobody has written yet — a `Related:` reference that grep can't find.

So: yes — codebases are learnable, when there's a signal an optimizer can iterate against, a judge with better properties than the artifact, and somewhere for the experience to accumulate. LDD produces the signal; the feedback ranking selects the judge; the handbook keeps the experience. And the pattern recurses: our docs site regenerates its API reference from the typed sources and its chart gallery by building the renderer and rendering fixtures — documentation as one more fitted artifact, with the code as its oracle.

## Open threads

- **Generalization gap.** The agent samples inputs it can think of. What about adversarial / never-thought-of inputs? Does fuzzing on top of an LDD baseline help — generate inputs randomly, query the oracle, append? Reintroduces the train/test distribution problem from ML.
