# Strategy: speed, quality, cost

The archetype table describes the *task*. Strategy describes *your constraints* — the deadline, the quality bar, the budget. Same task, different constraints, different spawn. Strategy is the input; model, effort, and precision are what it resolves to.

## The three knobs

| Knob | Values | Default | What it actually controls |
|---|---|---|---|
| `speed` | `fast` · `normal` | `normal` | Wall-clock latency. Buys down **effort** and return length. Never lowers the model tier. |
| `quality` | `top-notch` · `normal` | `normal` | A **floor** on model tier and thoroughness. Bounded by the archetype's fitness — see below. |
| `cost` | `cheap` · `moderate` · `richie-rich` | `moderate` | A **ceiling** on model tier and effort. The only knob that can make a task unservable. |

Defaults resolve to the `standard` profile — the archetype table in `routing.md` exactly as written. An unset strategy changes nothing.

## What each knob may touch

**Precedence: cost ceiling > quality floor > speed lever.** A ceiling always wins; you cannot spend what you do not have.

- **Speed lowers effort, not the model.** Opus at low effort is five minutes with an expert — real recognition, shallow reading. That is often exactly what a deadline wants. So `fast` + `top-notch` is not a contradiction: it is a high tier at low effort, capability without deliberation. Only `cost` moves the tier down.
- **`cost: richie-rich` with `quality: normal` buys nothing.** Money is not a quality dial on its own. Spending more only helps if you also asked for more, and overthinking degrades results — it is not free thoroughness. If you meant "get this right," set `quality`.
- **`quality: top-notch` has a fitness ceiling.** It raises the tier where judgment is the bottleneck. It does not put Opus on a locator: nothing in "find every call site" rewards judgment, so premium buys a tier that catches dynamic references, not a tier that reasons about them.
- **`cost: cheap` is the only knob that can refuse.** For some archetypes no cheap configuration succeeds. Those cells say so rather than degrading quietly.

## Resolving the combination

Twelve combinations, five profiles.

| `speed` | `quality` | `cost` | Resolves to |
|---|---|---|---|
| normal | normal | moderate | `standard` |
| normal | normal | richie-rich | `standard` — budget with no quality demand buys nothing |
| fast | normal | moderate | `swift` |
| fast | normal | richie-rich | `swift`; where the harness exposes it, Opus 5.5 fast mode is the one thing the budget can buy here |
| normal | normal | cheap | `economy` |
| fast | normal | cheap | `economy` — `cheap` already floors both dials; `fast` adds nothing |
| normal | top-notch | moderate | `premium` |
| fast | top-notch | moderate | `premium`, one effort level down |
| normal | top-notch | richie-rich | `no-limits` |
| fast | top-notch | richie-rich | `no-limits`, effort capped at low–medium — tier stays, deliberation goes. Or keep effort and buy speed with Opus 5.5 fast mode |
| normal | top-notch | cheap | **contested** → `economy` + maximum precision. See *When cheap can't buy it* |
| fast | top-notch | cheap | **contested** → same, plus a tighter return contract |

Strategy resolves to an effort level, and a top-level effort change invalidates prompt caching. Fix the strategy for a subagent's whole run.

## Prices that drive the tables

| | Input $/MTok | Output $/MTok | Relative |
|---|---|---|---|
| Haiku 5.5 | 0.10 (0.50 above 100K-token prompts) | 0.50 (2.50) | 1× |
| Sonnet 5.5 | 2 | 10 | 20× |
| Opus 5.5 | 4 | 20 | 40× |
| Fable 5.1 | 10 | 50 | 100× |

Two consequences shape every profile below:

- **The Sonnet → Opus step is cheap (2×), and the Sonnet → Haiku step is a 95% saving.** The tier that `cost` gives up is now usually Sonnet → Haiku, not Opus → Sonnet.
- **Judge cost per completed task, not per token.** Each 5.5 model at a lower effort beats its predecessor at a higher one, and finishes in fewer requests. A cheaper model that needs a retry is not cheaper.

## The five profiles

**`economy`**: Sonnet 5.5 is the tier ceiling and `medium` the effort ceiling. Haiku 5.5 takes everything it can, which is more than Haiku used to: it now has effort levels and follows instructions well. The saved capability is paid back in precision: name the globs, the schema, the stop condition, the skip-and-list rule. Cheap models do not wander when there is nowhere to wander. Keep Haiku jobs short so their prompts stay under the 100K price cliff. Four archetypes have no economy configuration.

**`swift`**: The tier holds; effort drops, precision rises, and return contracts get hard. Most perceived latency is output length and tool-call count, not model choice, so cap both explicitly. Haiku 5.5 is the fastest model at standard speed; Sonnet 5.5 is 30% faster than Sonnet 5. This is also the fan-out profile: breadth comes from parallelism, not from any one agent's depth.

**`standard`**: The archetype table in `routing.md`, unchanged. Not reproduced here.

**`premium`**: Opus 5.5 is the floor wherever judgment is the bottleneck. The easy half of the table moves up a tier (cheap now that Opus is 2× Sonnet), and the hard half moves up one effort level, `high` → `xhigh`.

**`no-limits`**: Top tier at `xhigh`–`max`, plus the thing an unlimited budget actually buys: **redundancy**. Two agents with different strategies reconciled, a worker paired with an independent verifier, or Opus 5.5 and Fable 5.1 attacking the same problem: each beats one agent at a higher number. CodeRabbit's data makes the case concretely: its lower- and higher-effort Opus 5.5 configurations each caught 10 of 13 hard bugs, but not the same 10. On the API, set `max_tokens` around 64k on every row in this profile so it has room to think across tool calls. Don't run Sonnet 5.5 at `max`; it over-delegates and scores below its own `xhigh`.

## Archetype × profile

Copy the cell. `standard` lives in `routing.md`.

### `economy`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Haiku 5.5 | low | Maximum: globs, directories, stop condition | ≤300 tok, `file:line` only |
| **Mechanical edit** | Haiku 5.5 if sites are uniform; Sonnet 5.5 if they vary | low | Maximum: exact before/after, do-not-touch list, skip-and-list rule | ≤300 tok + diff |
| **Extract / classify / convert** | Haiku 5.5 | low | Maximum: schema, enum values, tie-breaks | Exact schema, nothing else |
| **Bounded implementation** | Sonnet 5.5 | low | High: the interface, the file to imitate, the invariant, plus the verification line (`low` skips checks) | ≤400 tok + diff |
| **Research / synthesis** | Haiku 5.5 at medium for lookup-shaped questions (give it today's date); Sonnet 5.5 at low when the synthesis matters | medium / low | High: exact questions, named sources, no open browsing | ≤800 tok, conclusion first |
| **Codebase archaeology** | Sonnet 5.5 | medium | High: name the entry points; ask for the path, not the map | ≤800 tok |
| **Deep debugging** | — | — | — | **Don't.** Split it: cheap locator to narrow the surface, then one expensive call |
| **Design / architecture / trade-off call** | — | — | — | **Don't.** Cheap architecture is the most expensive line item you will ever ship |
| **Verifier / critic** | Haiku 5.5 for mechanical, per-item rubrics | high | Maximum: rubric as a file, pass/fail per item | ≤300 tok verdict |
| **Long-horizon migration** | — | — | — | **Don't** as one agent. Pay once for the plan, then run economy codemods per slice |
| **Frontier / everyone-is-stuck** | — | — | — | **Don't.** Nothing cheap cracks what nothing else could |

### `swift`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Haiku 5.5 | low | Maximum | ≤300 tok; partition across N parallel |
| **Mechanical edit** | Sonnet 5.5 | low | Maximum | ≤300 tok + diff; partition by directory |
| **Extract / classify / convert** | Haiku 5.5 | low | Maximum | Exact schema, nothing else |
| **Bounded implementation** | Sonnet 5.5 | low | High: interface plus a file to imitate, plus the verification line | ≤400 tok + diff |
| **Research / synthesis** | Sonnet 5.5 | low | High: three named sources, question shaped as a list or a yes/no | ≤500 tok, conclusion first |
| **Codebase archaeology** | Sonnet 5.5 | medium | High: entry points named, tool budget ~15 calls | ≤600 tok |
| **Deep debugging** | Opus 5.5 | low | Low on method, high on situation: repro, ruled-out, invariant | ≤500 tok. Buying recognition, not search; Opus 5.5 at `low` out-finds Opus 5 at `high` |
| **Design / architecture / trade-off call** | Opus 5.5 | low | Low: constraints only | ≤400 tok: recommendation first, runners-up one line each |
| **Verifier / critic** | Opus 5.5 for judgment; Sonnet 5.5 for the routine pass | low | Maximum: rubric as a file | ≤300 tok verdict |
| **Long-horizon migration** | Opus 5.5 for the plan, then parallel Sonnet 5.5 codemods at low | medium | Medium: the invariant and the slice boundaries | Plan ≤500 tok; slices report ≤300 tok each |
| **Frontier / everyone-is-stuck** | Fable 5.1 | low–medium | Low | Whatever it takes. Fable's `low` is competitive with Opus and Sonnet on cost per task |

### `premium`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Sonnet 5.5 | medium | Medium: the question and what counts as a use; ask for indirect and dynamic references | ≤400 tok, `file:line` + one clause |
| **Mechanical edit** | Sonnet 5.5 | medium | Medium: the intent behind the pattern, plus the do-not-touch list | ≤400 tok + diff; ambiguous sites listed, never guessed |
| **Extract / classify / convert** | Sonnet 5.5 | low | High: schema and tie-breaks, plus "flag anything the schema can't hold" | Schema + a short exceptions list |
| **Bounded implementation** | Opus 5.5 | medium | Medium: objective, constraints, the invariant; not the how | ≤800 tok + diff |
| **Research / synthesis** | Opus 5.5 | high | Low–medium: the question, sources of truth, what "answered" means | 1,000–2,000 tok, conclusion first |
| **Codebase archaeology** | Opus 5.5 | high | Low: the question and the entry points; let it wander | 1,000–1,500 tok |
| **Deep debugging** | Opus 5.5 | xhigh | Low: symptom, repro, ruled-out. Do not prescribe the search | ≤1,000 tok: cause, evidence, fix |
| **Design / architecture / trade-off call** | Opus 5.5 | xhigh | Low: constraints and non-negotiables, not a preferred answer | Structured: options, trade-offs, recommendation |
| **Verifier / critic** | Opus 5.5 | high | Medium: rubric as a reference file | ≤500 tok verdict + violations |
| **Long-horizon migration** | Opus 5.5; Fable 5.1 if it has stalled | xhigh | Medium: the invariant, checkpoint cadence, where to write notes | Notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5.1 | high | Low: full problem, everything already tried | Whatever it takes |

### `no-limits`

Set `max_tokens` around 64k on every row here (API).

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Sonnet 5.5 | high | Low: the question; let it decide what counts | ≤600 tok + a confidence note. Or two locators with different strategies, reconciled |
| **Mechanical edit** | Opus 5.5 | medium | Low–medium: the intent; let it flag where the pattern is wrong | ≤600 tok + diff + sites the pattern should not touch |
| **Extract / classify / convert** | Opus 5.5 | low–medium | Medium: schema plus "tell me where the schema is wrong" | Schema + exceptions + a critique of the schema |
| **Bounded implementation** | Opus 5.5 | high | Low: objective and constraints; pair it with a separate verifier | ≤800 tok + diff + verifier verdict |
| **Research / synthesis** | Opus 5.5; Fable 5.1 if the synthesis *is* the job | xhigh | Low | ≤2,000 tok + a sources appendix |
| **Codebase archaeology** | Opus 5.5 | xhigh | Low | 1,500 tok to you + a written map committed to a file |
| **Deep debugging** | Opus 5.5 at max, with Fable 5.1 on the same problem in parallel as an independent attempt | max / high | Minimum: the situation exhaustively, the method not at all | ≤1,000 tok each: cause, evidence, fix; reconcile |
| **Design / architecture / trade-off call** | Fable 5.1 | high–xhigh | Minimum | Structured; then a separate Opus 5.5 verifier against the constraints |
| **Verifier / critic** | Two Opus 5.5 verifiers: one at low–medium for precision, one at max for recall on hard cases | low–medium + max | Medium | ≤500 tok each. They miss different things; reconcile |
| **Long-horizon migration** | Fable 5.1, or Opus 5.5 orchestrating async subagents | xhigh / high | Medium: the invariant, checkpoint every N files, the notes file | Notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5.1 | max | Minimum | Whatever it takes |

## When cheap can't buy it

`cost: cheap` with `quality: top-notch` is the one combination that cannot be honoured. Cost wins the tier — you cannot spend what you do not have — and the quality deficit is paid back where it can be: maximum precision on the handoff, and effort spent wherever the cheaper tier still converts it into accuracy.

Four archetypes have no cheap configuration at all: **deep debugging**, **design / architecture**, **long-horizon migration**, **frontier**. Each is bottlenecked on judgment, which is the one thing precision cannot supply. Say so instead of shipping a degraded spawn. Then pick one:

- **Decompose.** Most expensive tasks are a cheap search wrapped around one expensive decision. Fan out locators at `economy`, narrow the surface, then spend once on the decision with everything already found. This is the single best cost lever available.
- **Shrink the task.** One subsystem instead of the codebase. One hypothesis instead of the space.
- **Renegotiate.** Say what the task needs and what it would cost. A refused spawn is cheaper than a confident wrong answer, and question 3 in `SKILL.md` already prices that.

Escalation respects the ceiling too. If a result comes back thin and you are already at the cost ceiling, the next move is not a quiet upgrade — it is decomposing, or asking for more budget.

## Recording it

State the resolution before spawning, strategy first, so the derivation can be corrected and not just the outcome:

```
→ [fast · normal · cheap] → haiku-5.5 · low effort · maximum precision · return ≤300 tok   (locator fan-out ×6)
→ [normal · top-notch · richie-rich] → opus-5.5 · max effort + fable-5.1 · high in parallel · minimum precision · return ≤1000 tok each   (deep debugging)
```

In a persisted agent definition, record the strategy as intent alongside the resolved values — see `agent-defs.md`:

```yaml
strategy: { speed: fast, quality: normal, cost: cheap }
```
