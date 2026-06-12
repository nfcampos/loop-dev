# Are codebases learnable?

*This post closes the loop driven development series. The [README](README.md) makes the argument and ends with a question — once you see code this way, is writing software really so different from training a model? — and the three practical posts ([a corpus benchmark](ldd-in-practice.md), [chart rendering](ldd-render-fidelity.md), [a performance campaign](ldd-perf.md)) keep raising it from different angles. This post takes the question seriously.*

## Three ways to fit a function

There are three established ways to fit an artifact to a teacher's input → output behavior:

- **Train a model.** The optimizer is gradient descent; the artifact is weights.
- **Tune a prompt.** The optimizer is a search loop, manual or automated; the artifact is a prompt that elicits the desired behavior from a base model.
- **Write code, LDD-style.** The optimizer is a coding agent reading test results; the artifact is an implementation that passes them.

Before agents, the third looked categorically different. Writing code was creative authorship; training a model was statistical fitting; nobody confused the two. With an agent in the loop, it's hard to maintain the distinction: in each case an optimizer iterates an artifact against sampled teacher input/output until the behavior matches. The artifact happens to be code.

What convinced me isn't the analogy — it's that one optimizer now performs all three jobs. The same kind of coding agent that ground through 479 chart fixtures in the [rendering post](ldd-render-fidelity.md) will, handed a GPU box, hill-climb a training script: in [Parameter Golf](https://x.com/rlancemartin/status/2064397389189071163), an open ML-engineering challenge, the agent edits the training code, launches the run, polls the log, reads the score, and decides the next experiment. Prompt optimization went programmatic years ago — [DSPy](https://github.com/stanfordnlp/dspy) compiles prompts against a metric, [GEPA](https://arxiv.org/abs/2507.19457) mutates them against one. The activity is the same loop in all three; only the artifact changes. (I've lived the overlap directly: our published [research log](https://github.com/witanlabs/research-log) records one project optimizing prompts, code, and the eval harness itself against a single benchmark.)

## The differences are properties of the artifact

If the activity is the same, what actually distinguishes the three is the properties of the thing being fitted:

|  | Weights | Prompt | Code |
| :---- | :---- | :---- | :---- |
| Determinism | No | No | Yes |
| Auditability | Low (weights opaque) | High (read the prompt) | High (read the source) |
| Off-sample behavior | Probabilistic interpolation | Depends on base model | Deterministic for implemented features; absent otherwise |

These aren't abstractions; you can watch each row in a working repo. Take auditability and off-sample behavior. The engine behind these posts keeps a checked-in coverage inventory that maps each public mutation and OOXML part to its test surface, with a section for known gaps — the *untrained region* of the artifact is a file you can read. There is no analogous file for a model's weights: off-distribution behavior isn't enumerable, only sampled. The optimization history is inspectable the same way — the [performance post](ldd-perf.md) describes a log where discarded experiments are recorded next to the kept ones, a training curve with the rejected gradient steps preserved. Even the training distribution is a legible choice: the engine's vendored JavaScript interpreter ships with the Test262 conformance suite in its fork, so the artifact carries its own teacher with it, and its fork notes say explicitly which behavior regions are cut.

Which artifact to fit, then, is a question about required properties, not about which activity feels like engineering:

**Code fits when:**

- Determinism and audit are required
- The behavior space is bounded enough to enumerate the meaningful cases

**Weights or prompts fit when:**

- The input space is too large to enumerate (natural language, vision, long-tail UI)
- Approximate generalization matters more than bounded determinism
- The oracle is itself contested or evolving, and you'd rather absorb the uncertainty than freeze it

## The judge needs better properties than the artifact

One constraint binds all three: the optimizer is only as good as its loss signal, and the judge supplying that signal needs better properties than the thing being optimized. An LLM judging an LLM-built artifact is the weakest arrangement available — nondeterministic grading nondeterministic, with [a documented bias toward confidently praising its own work](https://www.anthropic.com/engineering/harness-design-long-running-apps). Every remedy moves the judge up the property table: [an independent verifier sub-agent beats self-critique](https://x.com/rlancemartin/status/2064397389189071163); a programmatic comparison beats either; a frozen oracle snapshot beats them all.

The practical posts are, on reflection, mostly stories about judges. The corpus bench judges five engines with one twelve-line comparator, and its biggest discovery was that the judge itself was broken — a locale mismatch that accounted for 96% of an apparent 281,941-cell engine gap. The render gate judged not just the renderer but a human reviewer, settling a code-review disagreement by measurement. The performance campaign's judge was byte equality, the strongest property a judge can have, and even there the judge needed debugging twice. The feedback ranking in the README is exactly this principle in list form: it orders feedback sources by the properties of the judge — independence, then queryability — and the whole method falls out of taking that ordering seriously.

## Where the experience goes

Gradient descent has a built-in answer to "where does experience accumulate?" — the weights. The agent era breaks that. The base model is frozen, the session that learned something ends, and whatever the agent figured out about your domain dies with the context window. So when an agent spends an afternoon discovering some hard truth about its environment, where does the learning go?

This question is starting to be taken seriously as a research matter — [Yoonho Lee argues](https://yoonholee.com/blog/2026/we-should-take-text-optimization-more-seriously/) that text optimization deserves the same attention as weight optimization, because a text update serves the same functional role as a weight update, often more sample-efficiently. My working answer is a fourth fitted artifact: the [editable handbook](https://github.com/nfcampos/editable-handbooks), a skill-shaped directory of atomic markdown entries that agents both read and write. One file per concept, grep as the index, `Related:` lines as the link graph, read-before-edit so every write sees the existing terminology before extending it. Curated priors and accreted experience live in the same files.

The production instance behind these posts has ninety-odd entries, and its densest one is about the oracles themselves: seventy-some bullets of operational knowledge accreted across sessions of driving real Excel and PowerPoint programmatically. Readers of the practical posts have already brushed against this layer without seeing it named. The AppleScript clipboard technique that captures the render post's ground-truth PNGs is a handbook lesson (the obvious export API intermittently fails; the clipboard path doesn't). So is "serialize oracle jobs — Office automation state collides across parallel opens." So are the capture rules about waiting for slides to stop animating, and converting screenshots to sRGB before computing pixel metrics because display color profiles otherwise dominate the diff. Each entry is a scar from a session where the oracle misled before it taught, distilled so the next session doesn't pay for it again. The entry's hardest-won line: "a bad oracle is worse than no oracle, because it can send work in the wrong direction" — which the corpus post then demonstrated at the scale of 271,000 cells.

As an artifact, the handbook straddles the property table in an interesting way. It influences behavior like a prompt — consulted, not executed, probabilistic. But it audits like code: you can `git blame` an individual belief and see when, and from what evidence, the team's priors changed. You could describe it as the prompt row of the table running continuously in production, with the agent as its own prompt engineer — fail, investigate, verify, distill, consult, the [memory progression](https://x.com/rlancemartin/status/2064397389189071163) observed in benchmark settings, operating as ordinary engineering practice.

It also corrects a simplification in the README. There, the marginal oracle query is approximately free. True — but the oracle *harness* is not. It is a learned artifact with its own training history, and the handbook is where that training persisted. Experience then hardens along a gradient: prose (a handbook entry about the screenshot chrome crop) becomes code (the capture script that applies it) becomes data (the frozen truth PNGs the script produces). Each conversion trades flexibility for determinism, and the agent performs the conversions itself.

## So: are they learnable?

Looking back across the series, learnability took three ingredients, and each post supplied one. A signal an optimizer can iterate against — that's the corpus bench, the render gate, the perf baselines; producing the signal is what LDD is. A judge with better properties than the artifact — the shared comparator, the two-sided ratchet, byte equality. And somewhere for the experience to accumulate between sessions — the handbook. Given all three, yes: codebases are learnable, in the same sense models are, with the differences living in the artifact's properties rather than in the nature of the activity.

The common limit is also shared: what wasn't sampled isn't verified. In code the unverified region is discrete — missing tests, unwritten functions, a gap row in the coverage inventory. In weights and prompts it's continuous, off-distribution interpolation you can't enumerate. In a handbook it's simply the entry nobody has written yet, a `Related:` reference that grep can't find. And the pattern recurses one level further than I expected when I started: our docs site now regenerates its API reference from the typed sources and its chart gallery by building the renderer and rendering the fixtures — documentation as one more fitted artifact, with the code as its oracle.

One question stays open. The agent samples the inputs it can think of; an adversary, or plain bad luck, supplies inputs it can't. Whether fuzzing on top of an LDD baseline closes that gap — generate inputs randomly, ask the oracle, append — is something I haven't tested yet, and it reintroduces the oldest problem in machine learning: the training distribution is not the test distribution. Which is, perhaps, one more point in favor of the premise of this post.
