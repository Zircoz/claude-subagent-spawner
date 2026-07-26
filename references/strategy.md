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
| fast | normal | richie-rich | `swift` — same; the budget is not the binding constraint |
| normal | normal | cheap | `economy` |
| fast | normal | cheap | `economy` — `cheap` already floors both dials; `fast` adds nothing |
| normal | top-notch | moderate | `premium` |
| fast | top-notch | moderate | `premium`, one effort level down |
| normal | top-notch | richie-rich | `no-limits` |
| fast | top-notch | richie-rich | `no-limits`, effort capped at low–medium — tier stays, deliberation goes |
| normal | top-notch | cheap | **contested** → `economy` + maximum precision. See *When cheap can't buy it* |
| fast | top-notch | cheap | **contested** → same, plus a tighter return contract |

Strategy resolves to an effort level, and effort is per-request — changing it mid-run invalidates prompt caching. Fix the strategy for a subagent's whole run.

## The five profiles

**`economy`** — Sonnet 5 is the tier ceiling, `medium` the effort ceiling. The saved capability is paid back in precision: name the globs, the schema, the stop condition, the skip-and-list rule. Cheap models do not wander when there is nowhere to wander. Four archetypes have no economy configuration.

**`swift`** — Tier holds, effort drops, precision rises, return contracts hard. Most perceived latency is output length and tool-call count, not model choice, so cap both explicitly. This is also the fan-out profile: breadth comes from parallelism, not from any one agent's depth.

**`standard`** — The archetype table in `routing.md`, unchanged. Not reproduced here.

**`premium`** — Opus 5 floor wherever judgment is the bottleneck; effort at or one above baseline. Note that the base table is *already* premium for the hard archetypes — deep debugging, design, migration, and frontier are unchanged. `quality: top-notch` mostly changes the easy half of the table, where the default was chosen for thrift.

**`no-limits`** — Top tier, `xhigh`–`max`, and the thing an unlimited budget actually buys: **redundancy**. Two agents with different strategies reconciled, or a worker paired with an independent verifier, beats one agent at a higher number. Thinking cannot be disabled at `xhigh` or `max` — set `max_tokens` around 64k on every row in this profile so it has room to think across tool calls.

## Archetype × profile

Copy the cell. `standard` lives in `routing.md`.

### `economy`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Haiku 4.5 | fixed | Maximum — globs, directories, stop condition | ≤300 tok, `file:line` only |
| **Mechanical edit** | Haiku 4.5 if sites are uniform; Sonnet 5 if they vary | fixed / low | Maximum — exact before/after, do-not-touch list, skip-and-list rule | ≤300 tok + diff |
| **Extract / classify / convert** | Haiku 4.5 | fixed | Maximum — schema, enum values, tie-breaks | Exact schema, nothing else |
| **Bounded implementation** | Sonnet 5 | low | High — the interface, the file to imitate, the invariant | ≤400 tok + diff |
| **Research / synthesis** | Sonnet 5 | medium | High — exact questions, named sources, no open browsing | ≤800 tok, conclusion first |
| **Codebase archaeology** | Sonnet 5 | medium | High — name the entry points; ask for the path, not the map | ≤800 tok |
| **Deep debugging** | — | — | — | **Don't.** Split it: cheap locator to narrow the surface, then one expensive call |
| **Design / architecture / trade-off call** | — | — | — | **Don't.** Cheap architecture is the most expensive line item you will ever ship |
| **Verifier / critic** | Sonnet 5 | low | Maximum — rubric as a file, pass/fail per item | ≤300 tok verdict |
| **Long-horizon migration** | — | — | — | **Don't** as one agent. Pay once for the plan, then run economy codemods per slice |
| **Frontier / everyone-is-stuck** | — | — | — | **Don't.** Nothing cheap cracks what nothing else could |

### `swift`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Haiku 4.5 | fixed | Maximum | ≤300 tok; partition across N parallel |
| **Mechanical edit** | Sonnet 5 | low | Maximum | ≤300 tok + diff; partition by directory |
| **Extract / classify / convert** | Haiku 4.5 | fixed | Maximum | Exact schema, nothing else |
| **Bounded implementation** | Sonnet 5 | low | High — interface plus a file to imitate | ≤400 tok + diff |
| **Research / synthesis** | Sonnet 5 | low–medium | High — three named sources, question shaped as a list or a yes/no | ≤500 tok, conclusion first |
| **Codebase archaeology** | Sonnet 5 | medium | High — entry points named, tool budget ~15 calls | ≤600 tok |
| **Deep debugging** | Opus 5 | low | Low on method, high on situation — repro, ruled-out, invariant | ≤500 tok. Buying recognition, not search |
| **Design / architecture / trade-off call** | Opus 5 | low–medium | Low — constraints only | ≤400 tok: recommendation first, runners-up one line each |
| **Verifier / critic** | Sonnet 5 | low | Maximum — rubric as a file | ≤300 tok verdict |
| **Long-horizon migration** | Opus 5 for the plan, then parallel economy codemods | medium | Medium — the invariant and the slice boundaries | Plan ≤500 tok; slices report ≤300 tok each |
| **Frontier / everyone-is-stuck** | Fable 5 | low–medium | Low | Whatever it takes. Fable's `low` often beats prior-generation `xhigh` |

### `premium`

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Sonnet 5 | medium | Medium — the question and what counts as a use; ask for indirect and dynamic references | ≤400 tok, `file:line` + one clause |
| **Mechanical edit** | Sonnet 5 | medium | Medium — the intent behind the pattern, plus the do-not-touch list | ≤400 tok + diff; ambiguous sites listed, never guessed |
| **Extract / classify / convert** | Sonnet 5 | low–medium | High — schema and tie-breaks, plus "flag anything the schema can't hold" | Schema + a short exceptions list |
| **Bounded implementation** | Opus 5 | high | Medium — objective, constraints, the invariant; not the how | ≤800 tok + diff |
| **Research / synthesis** | Opus 5 | xhigh | Low–medium — the question, sources of truth, what "answered" means | 1,000–2,000 tok, conclusion first |
| **Codebase archaeology** | Opus 5 | xhigh | Low — the question and the entry points; let it wander | 1,000–1,500 tok |
| **Deep debugging** | Opus 5 | xhigh | Low — symptom, repro, ruled-out. Do not prescribe the search | ≤1,000 tok: cause, evidence, fix |
| **Design / architecture / trade-off call** | Opus 5 | xhigh | Low — constraints and non-negotiables, not a preferred answer | Structured: options, trade-offs, recommendation |
| **Verifier / critic** | Opus 5 | high | Medium — rubric as a reference file | ≤500 tok verdict + violations |
| **Long-horizon migration** | Opus 5; Fable 5 if it has stalled | xhigh | Medium — the invariant, checkpoint cadence, where to write notes | Notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5 | high | Low — full problem, everything already tried | Whatever it takes |

### `no-limits`

Set `max_tokens` ~64k on every row here.

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** | Sonnet 5 | xhigh | Low — the question; let it decide what counts | ≤600 tok + a confidence note. Or two locators with different strategies, reconciled |
| **Mechanical edit** | Opus 5 | high | Low–medium — the intent; let it flag where the pattern is wrong | ≤600 tok + diff + sites the pattern should not touch |
| **Extract / classify / convert** | Opus 5 | medium | Medium — schema plus "tell me where the schema is wrong" | Schema + exceptions + a critique of the schema |
| **Bounded implementation** | Opus 5 | xhigh | Low — objective and constraints; pair it with a separate verifier | ≤800 tok + diff + verifier verdict |
| **Research / synthesis** | Opus 5; Fable 5 if the synthesis *is* the job | max | Low | ≤2,000 tok + a sources appendix |
| **Codebase archaeology** | Opus 5 | max | Low | 1,500 tok to you + a written map committed to a file |
| **Deep debugging** | Opus 5; Fable 5 if Opus has already failed | max | Minimum — the situation exhaustively, the method not at all | ≤1,000 tok: cause, evidence, fix |
| **Design / architecture / trade-off call** | Fable 5 | high–xhigh | Minimum | Structured; then a separate Opus verifier against the constraints |
| **Verifier / critic** | Opus 5 | xhigh | Medium | ≤500 tok. Two verifiers with different rubrics is the better spend |
| **Long-horizon migration** | Fable 5 | xhigh | Medium — the invariant, checkpoint every N files, the notes file | Notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5 | max | Minimum | Whatever it takes |

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
→ [fast · normal · cheap] → haiku-4.5 · fixed effort · maximum precision · return ≤300 tok   (locator fan-out ×6)
→ [normal · top-notch · richie-rich] → opus-5 · max effort · minimum precision · return ≤1000 tok   (deep debugging)
```

In a persisted agent definition, record the strategy as intent alongside the resolved values — see `agent-defs.md`:

```yaml
strategy: { speed: fast, quality: normal, cost: cheap }
```
