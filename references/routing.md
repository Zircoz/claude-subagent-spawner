# Routing: model and effort

## The models

| Model | Character | Spawn a subagent on it when |
|---|---|---|
| **Haiku 4.5** | Fast, cheap, literal | The task is fully specified and mechanical, and you are running many in parallel. Give it no room to interpret. |
| **Sonnet 5** | Strong generalist | Default. Anything routine-to-moderate, and anything where thoroughness over *your* specifics matters more than pattern recognition. |
| **Opus 5** | Expert | The task is ambiguous, the domain is unfamiliar, the bug is subtle, or the decision has architectural consequences. |
| **Fable 5** | Specialist | Nothing else has cracked it, or the job is long-horizon and multi-step enough that lesser models grind and stall. Costs the most per token — save it. |

Model selection swaps which frozen weights answer. Context steers the answer; it does not add knowledge that isn't in the weights. If Opus is confidently wrong about a library that postdates it, no amount of effort fixes that — put the docs in the prompt or move up.

## The effort levels

| Effort | Behavior | Use for a subagent that… |
|---|---|---|
| `low` | Fewest tokens. Combines operations into fewer tool calls, acts without preamble, terse confirmation. Prefers asking you over figuring it out alone. | …has a closed, fully-specified job. Locators, single-file edits, classification, extraction, format conversion. |
| `medium` | Balanced. The cost-saving step down from default. | …does bounded multi-step work with a known shape. Routine implementation, scoped refactors, summarization over known inputs. |
| `high` | Default. Complex reasoning, agentic work. | …you have no strong reason to move off default. Most subagents. |
| `xhigh` | Extended, for long-horizon work — repeated tool calling, detailed search, knowledge-base traversal. Expect meaningfully more tokens than `high`. | …must explore: deep debugging, wide codebase archaeology, multi-source research, migrations running over 30 minutes. |
| `max` | No constraint on spend. | …is a genuinely frontier problem. On most workloads it adds real cost for small gains, and can cause overthinking on structured-output tasks. Reserve it. |

Effort is a **behavioral signal, not a token cap**. At low effort a model still thinks on hard problems — just less. The only hard cap is `max_tokens`, which truncates mid-stream; prefer prompt-level "keep it to N" guidance over a wall.

### Per-model notes

- **Haiku 4.5 does not take an effort parameter.** Control it through prompt precision and an explicit tool budget instead. Treat it as fixed-effort.
- **Sonnet 5** — defaults to `high`. `medium` is roughly Sonnet 4.6 at high, and is the honest default for most agentic subagents. `low` for high-volume or latency-bound fan-out.
- **Opus 5** — defaults to `high`. Step to `xhigh` for demanding coding and agentic work; use `low`/`medium` liberally as the primary cost and latency control wherever quality holds. Thinking cannot be disabled at `xhigh` or `max`. Set a large `max_tokens` (start ~64k) at those levels so it has room to think and act across tool calls.
- **Fable 5** — defaults to `high`; effort is the primary intelligence/latency/cost dial. Its `medium` and `low` still often beat `xhigh` on prior-generation models, so start lower than instinct suggests.
- **Effort does not shorten prose.** On Opus 5 especially, changing effort does not reliably shorten the visible response. Ask for length in the prompt.
- Effort is per-request and reshapes the rendered prompt, so **changing it mid-conversation invalidates prompt caching**. Pick one level for a subagent's whole run.

## Strategy

The table below answers "what kind of task is this." It does not answer "what am I willing to spend." Three knobs cover that — `speed` (`fast`/`normal`), `quality` (`top-notch`/`normal`), `cost` (`cheap`/`moderate`/`richie-rich`) — and they resolve with the archetype into the same four values.

**Precedence: cost ceiling > quality floor > speed lever.** Speed buys down effort, never the model tier. Cost is the only knob that can lower the tier, and the only one that can refuse a task outright. Full resolution tables in `strategy.md`.

## Archetypes

Copy the row that matches. `precision` is how prescriptive the handoff should be.

This is the `standard` profile — the default strategy, `normal · normal · moderate`. For any other strategy, resolve here first, then take the matching cell from `strategy.md`.

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator** — "find every place X is defined/used" | Haiku 4.5 or Sonnet 5 | low | High — name the globs, the directories, the stop condition | ≤300 tok, file:line list only |
| **Mechanical edit** — rename, codemod, apply a stated pattern per file | Sonnet 5 | low–medium | High — exact before/after, explicit do-not-touch list | ≤300 tok + diff |
| **Extract / classify / convert** — pull fields, tag items, reformat | Haiku 4.5 | n/a | Very high — schema, enum values, tie-break rules | Exact schema, nothing else |
| **Bounded implementation** — build a described component against a known interface | Sonnet 5 | medium–high | Medium — objective and constraints, let it choose the how | ≤800 tok + diff |
| **Research / synthesis** — multi-source, web or docs | Sonnet 5 (Opus 5 if the synthesis is the hard part) | xhigh | Low–medium — the question, sources of truth, what "answered" means | 1,000–2,000 tok |
| **Codebase archaeology** — "how does this subsystem actually work" | Sonnet 5 | xhigh | Low — the question and entry points; let it wander | 1,000–1,500 tok |
| **Deep debugging** — reproduce, isolate, diagnose a subtle failure | Opus 5 | xhigh | Low — symptom, repro, ruled-out hypotheses. Do not prescribe the search | ≤1,000 tok: cause, evidence, fix |
| **Design / architecture / trade-off call** | Opus 5 | high–xhigh | Low — constraints and non-negotiables, not a preferred answer | Structured: options, trade-offs, recommendation |
| **Verifier / critic** — check work against a rubric | Opus 5, or Sonnet 5 for mechanical checks | high | Medium — hand it the rubric as a reference file, not as prose | ≤500 tok verdict + specific violations |
| **Long-horizon migration** (>30 min, many files) | Opus 5; Fable 5 if it has stalled | xhigh | Medium — the invariant that must hold, checkpoint cadence, where to write notes | Progress notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5 | high | Low — full problem, everything already tried | Whatever it takes |

Parallel fan-out over the same archetype is `speed: fast` under cost pressure, and resolves through the `swift` or `economy` tables in `strategy.md` rather than by a separate rule: drop one model tier or one effort level, and tighten the return contract. N subagents returning 2,000 tokens each is how an orchestrator drowns.

## Escalating after a bad result

```
Bad subagent result
      │
      ├─ Was the handoff missing context, or self-contradictory?
      │     → fix the prompt. Do not touch the dials.
      │
      ├─ Did it SKIP work — unread file, unrun test, abandoned partway,
      │  came back asking instead of finding out?
      │     → effort up one level
      │
      └─ Did it clearly try, have everything it needed, and still get it wrong?
            → model up one tier
```

Symmetric down-shift: if a model has been doing routine work correctly for a while, drop a tier. If it is padding — over-verifying settled things, restating the plan — drop effort. Overthinking degrades results; it isn't free thoroughness.

Escalation respects a stated cost ceiling. If you are already at it, "effort up one level" is not available — decompose the task instead (cheap search, one expensive decision) or say what more budget would buy. Spending past a ceiling silently is the failure this skill exists to prevent.

Do not respond to a shallow result on a hard problem by prompting around it ("think carefully!") while leaving effort low. Raise effort. If latency forces low effort, *then* add targeted guidance like "this involves multistep reasoning; think before responding" and pair `low` with an explicit checklist when the task has multiple sections.
