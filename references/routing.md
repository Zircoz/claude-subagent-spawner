# Routing: model and effort

## The models

| Model | $/MTok in · out | Character | Spawn a subagent on it when |
|---|---|---|---|
| **Haiku 5.5** | 0.10 · 0.50 (prompts ≤100K; 0.50 · 2.50 above) | Fast, very cheap, strong instruction-follower | The task is closed and well specified: search, extraction, classification, uniform edits, rubric checks. Built to run as a subagent under a Sonnet or Opus lead. Weak on complex multi-step agentic coding. |
| **Sonnet 5.5** | 2 · 10 | Strong generalist, fastest Sonnet yet | Default. Well-scoped everyday work: implementation, bug fixes, terminal-heavy agentic coding (it beats Opus 5.5 on Terminal-Bench 4.0), the every-PR review pass. |
| **Opus 5.5** | 4 · 20 | Expert | The task is ambiguous, open-ended, or needs sustained judgment: subtle bugs, architecture, high-risk review, long-horizon migrations, orchestrating other agents. Only 2× Sonnet now, so reach for it whenever judgment is the bottleneck. |
| **Fable 5.1** | 10 · 50 | Specialist, long-distance runner | Opus 5.5 has already tried and failed, or the job is a multi-hour autonomous run with many async subagents. It no longer leads on published benchmarks; its edge is long-horizon persistence, and being a *different* model gives you an independent second opinion. |

Price ratios matter more than they used to: Haiku : Sonnet : Opus : Fable ≈ **1 : 20 : 40 : 100** on input. Dropping from Sonnet to Haiku saves 95%. Moving from Sonnet up to Opus costs only 2×.

Model selection swaps which frozen weights answer. Context steers the answer; it does not add knowledge that isn't in the weights. If a model is confidently wrong about a library that postdates its training, no amount of effort fixes that. Put the docs in the prompt or move up a tier.

## The effort levels

All four current models take `low` · `medium` · `high` · `xhigh` · `max`.

| Effort | Behavior | Use for a subagent that… |
|---|---|---|
| `low` | Fewest tokens. Combines operations into fewer tool calls, acts without preamble, terse confirmation. More likely to skip verification or stop early on long tasks (see quirks below). | …has a closed, fully specified job: locators, single-file edits, classification, extraction, format conversion. |
| `medium` | Balanced. **The default on Opus 5.5 and Haiku 5.5, and on Sonnet 5.5 inside Claude Code.** | …does bounded multi-step work with a known shape: routine implementation, scoped refactors, agentic coding against a clear spec. |
| `high` | Complex reasoning and agentic work. Default on Fable 5.1, and on Sonnet 5.5 via the API. | …must explore: debugging, codebase archaeology, multi-source research, design. |
| `xhigh` | Extended, for long-horizon work. On Opus 5.5 it thinks noticeably more per turn than Opus 5 did at the same level. | …is hard and you've seen `high` fall short: the deepest debugging, multi-hour migrations, judgment calls with expensive failure modes. |
| `max` | No constraint on spend. | …is a genuinely frontier problem. It often doesn't beat `xhigh` (Sonnet 5.5 scores *lower* at `max` than at `xhigh` on FrontierCode). Reserve it. |

Effort is a **behavioral signal, not a token cap**. At low effort a model still thinks on hard problems, just less. The only hard cap is `max_tokens`, which truncates mid-stream. Prefer prompt-level "keep it to N" guidance over a wall.

### Effort names are not portable

The 5.5 generation shifted the scale down. A level on one model is not the same amount of work as that level on another:

- **Opus 5.5 `medium` beats Opus 5 `high`** on coding and knowledge work, and its `low` comes close to Opus 5 `high` at much lower cost. At its lowest setting it caught 72% of known bugs in Deloitte's test; Opus 5 at `high` caught 56%.
- **Sonnet 5.5 `medium` beats Sonnet 5 at its best** on agentic coding, at under a tenth of the cost per task. Its levels were recalibrated, so a Sonnet 5 setting does not carry over.
- **Fable 5.1 at `low`** is competitive on cost per task with Opus and Sonnet while performing better.

So the old reflex of "exploration means `xhigh`" is now usually one level too high. Start each archetype one level lower than you would have on the 5.0 generation, and move up only when a subagent *skips* work.

**Always state effort explicitly.** Defaults differ by model and by surface: Sonnet 5.5 defaults to `high` on the API but `medium` in Claude Code. An omitted effort level is a silent, surface-dependent choice.

### Per-model notes and known quirks

- **Haiku 5.5**: the first Haiku that takes effort. Use `low` for high-volume chat-like and lookup tasks, `medium` (default) for most work, and `high` when strict instruction following matters. Above `high`, its thinking and replies get long, so compare against Sonnet 5.5 before settling there.
  - *Price cliff:* prompts over 100K tokens bill at 5× the rate. A subagent's prompt grows as its conversation accumulates, so a long-running Haiku agent can cross the line mid-run. Keep Haiku jobs short-lived, or compact early.
  - *Early stopping:* at `low` with a long system prompt, it sometimes hands the task back before finishing.
  - *Skipped checks:* at `low` and `medium` it sometimes reports code "done" without running tests.
  - *Empty replies:* at `xhigh` it occasionally ends with an empty reply.
  - *Search:* it searches less than it should unless given today's date and told which facts change.
  - It has no server-side refusal fallback.
- **Sonnet 5.5**: start at `medium` for agentic coding and multistep tool use, and `low` for chat, classification, extraction, and search.
  - *Low effort:* it may skip verifying a change, or stop to check in before finishing.
  - *Max effort:* it over-delegates. It reaches for fan-out skills that split work across many subagents, which caused timeouts and out-of-scope edits in Cognition's runs. Don't run it at `max`.
  - *Literal reading:* it follows "minimize tool calls" literally, so don't discourage tool use in its prompt.
- **Opus 5.5**: defaults to `medium`, one level below Opus 5. Start at `medium`, test the neighbors, and reserve `xhigh` and `max` for measured gains.
  - Thinking cannot be disabled; effort is the only control.
  - At a given level it thinks more than Opus 5 did, so lower effort before you prompt it to "think less."
  - It delegates to subagents well and is less likely than recent models to take hard-to-reverse actions.
  - Fast mode (2.5× output speed at 2× price, Claude API only) buys speed without cutting effort.
- **Fable 5.1**: defaults to `high`. Hard tasks can run 15+ minutes in a single request, so plan for long turns and prefer async check-ins over blocking.
  - At high effort on routine work it over-gathers context; step down to `medium` or `low` where quality holds.
  - Prescriptive step-by-step prompts *reduce* its output quality.
  - It sometimes takes adjacent actions nobody asked for, so state boundaries explicitly.
- **Effort does not shorten prose.** Ask for length in the prompt.
- **Caching:** changing the top-level effort between requests invalidates the prompt cache. On the API, a per-message effort change (beta) avoids that on all four current models. In Claude Code, pick one level per subagent run.

### Previous generation (still served)

If your harness pins older models (some cloud providers resolve the `opus`/`sonnet`/`haiku` aliases to older versions), route by *role* and fix the effort:

| Role | Previous gen | Difference that matters |
|---|---|---|
| Haiku slot | Haiku 4.5 | **Takes no effort parameter.** Control it through prompt precision and an explicit tool budget instead. 200K context. It is 10× the price of Haiku 5.5. |
| Sonnet slot | Sonnet 5 | Defaults to `high`. Start one level *higher* than the 5.5 table says. |
| Opus slot | Opus 5 | Defaults to `high`. Start one level higher than the 5.5 table says. It costs more per token than Opus 5.5. |
| Fable slot | Fable 5 | Its runs often go long until the harness cuts them off. Give it hard stop conditions. |

## Strategy

The table below answers "what kind of task is this." It does not answer "what am I willing to spend." Three knobs cover that: `speed` (`fast`/`normal`), `quality` (`top-notch`/`normal`), and `cost` (`cheap`/`moderate`/`richie-rich`). They resolve with the archetype into the same four values.

**Precedence: cost ceiling > quality floor > speed lever.** Speed buys down effort, never the model tier. Cost is the only knob that can lower the tier, and the only one that can refuse a task outright. Full resolution tables are in `strategy.md`.

## Archetypes

Copy the row that matches. `precision` is how prescriptive the handoff should be.

This is the `standard` profile, the default strategy `normal · normal · moderate`. For any other strategy, resolve here first, then take the matching cell from `strategy.md`.

| Archetype | Model | Effort | Precision | Return budget |
|---|---|---|---|---|
| **Locator**: "find every place X is defined/used" | Haiku 5.5 | low | High: name the globs, the directories, the stop condition | ≤300 tok, file:line list only |
| **Mechanical edit**: rename, codemod, apply a stated pattern per file | Sonnet 5.5; Haiku 5.5 if every site is uniform | low | High: exact before/after, explicit do-not-touch list | ≤300 tok + diff |
| **Extract / classify / convert**: pull fields, tag items, reformat | Haiku 5.5 | low; medium if tie-breaks need judgment | Very high: schema, enum values, tie-break rules | Exact schema, nothing else |
| **Bounded implementation**: build a described component against a known interface | Sonnet 5.5 | medium | Medium: objective and constraints, let it choose the how | ≤800 tok + diff |
| **Research / synthesis**: multi-source, web or docs | Sonnet 5.5; Opus 5.5 if the synthesis is the hard part | high | Low–medium: the question, sources of truth, what "answered" means | 1,000–2,000 tok |
| **Codebase archaeology**: "how does this subsystem actually work" | Sonnet 5.5 | high | Low: the question and entry points; let it wander | 1,000–1,500 tok |
| **Deep debugging**: reproduce, isolate, diagnose a subtle failure | Opus 5.5 | high | Low: symptom, repro, ruled-out hypotheses. Do not prescribe the search | ≤1,000 tok: cause, evidence, fix |
| **Design / architecture / trade-off call** | Opus 5.5 | high | Low: constraints and non-negotiables, not a preferred answer | Structured: options, trade-offs, recommendation |
| **Verifier / critic**: check work against a rubric | Opus 5.5 for judgment or high-risk changes; Sonnet 5.5 for the routine pass | medium | Medium: hand it the rubric as a reference file, not as prose | ≤500 tok verdict + specific violations |
| **Long-horizon migration** (>30 min, many files) | Opus 5.5; Fable 5.1 if it has stalled | high | Medium: the invariant that must hold, checkpoint cadence, where to write notes | Progress notes to a file; ≤500 tok to you |
| **Frontier / everyone-is-stuck** | Fable 5.1, after Opus 5.5 has had a real attempt | high | Low: full problem, everything already tried | Whatever it takes |

Why the verifier sits at `medium`: in CodeRabbit's code-review benchmark, Opus 5.5 at lower effort had the best precision (66.7% on the hard set). Pushing effort up caught more of the hardest bugs but added noise, and the two settings missed *different* bugs. Use `medium` for a clean verdict. Use two verifiers when recall on hard cases matters (see `no-limits`).

Parallel fan-out over the same archetype is `speed: fast` under cost pressure. It resolves through the `swift` or `economy` tables in `strategy.md` rather than through a separate rule: drop one model tier or one effort level, and tighten the return contract. N subagents returning 2,000 tokens each is how an orchestrator drowns. Haiku 5.5 makes fan-out cheap enough that the orchestrator's context, not the bill, is now usually the binding constraint.

## Escalating after a bad result

```
Bad subagent result
      │
      ├─ Was the handoff missing context, or self-contradictory?
      │     → fix the prompt. Do not touch the dials.
      │
      ├─ Did it SKIP work: unread file, unrun test, abandoned partway,
      │  came back asking instead of finding out?
      │     → effort up one level (or add the verification line — see prompt-contract.md)
      │
      └─ Did it clearly try, have everything it needed, and still get it wrong?
            → model up one tier
```

Skipped work at `low` on Haiku 5.5 or Sonnet 5.5 is a known pattern, not a mystery. Try the verification line from `prompt-contract.md` before paying for a higher level.

Symmetric down-shift: if a model has been doing routine work correctly for a while, drop a tier. If it is padding (over-verifying settled things, restating the plan, gathering context the task doesn't need), drop effort. Overthinking degrades results; it isn't free thoroughness.

Escalation respects a stated cost ceiling. If you are already at it, "effort up one level" is not available. Instead, decompose the task (cheap search, one expensive decision) or say what more budget would buy. Spending past a ceiling silently is the failure this skill exists to prevent.

Do not respond to a shallow result on a hard problem by prompting around it ("think carefully!") while leaving effort low. Raise effort. If latency forces low effort, *then* add targeted guidance like "this involves multistep reasoning; think before responding," and pair `low` with an explicit checklist when the task has multiple sections.
