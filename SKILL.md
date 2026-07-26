---
name: subagent-spawner
description: Choose the model, effort level, and prompt shape for a subagent before spawning it. Use when delegating work to a Task/subagent, fanning out parallel agents, deciding between Opus 5 / Sonnet 5 / Haiku 4.5 / Fable 5, writing or tuning a .claude/agents definition, or when a subagent came back shallow, over-thought, off-target, or too verbose. Also use when the constraint is stated as speed, quality, or cost — "do this fast", "latency matters", "keep it cheap", "stay in budget", "make it top-notch", "get this right", "money is no object" — and needs resolving into a concrete model and effort.
---

# Subagent spawner

Three dials, set together:

- **Model** — how *capable*. Frozen weights: what it knows, and how well it tolerates ambiguity.
- **Effort** — how *thorough*. How many files it reads, tools it calls, hypotheses it checks, and steps it takes before reporting back. Not just thinking time.
- **Prompt** — everything the subagent cannot cheaply rediscover, plus the exact shape of what comes back.

Two things set them: the **archetype** (what kind of task this is) and the **strategy** (`speed` · `quality` · `cost` — the constraints you are actually under). Strategy is upstream of the dials, not a fourth one. Same task under a deadline, a tight budget, or a production outage resolves differently — `references/strategy.md`.

They trade against each other. Opus at low effort is five minutes with an expert: real recognition, shallow reading. Sonnet at high effort is a strong generalist given the whole afternoon: thorough on *your* specifics, weaker on "I've seen this exact trap before." Fable even at low effort is the specialist who glances and spots what nobody else would.

The prompt closes the gap: **the less capable the model, the more prescriptive the handoff must be.** Capable models handle ambiguity and resent being over-constrained; smaller models need execution spelled out.

## Procedure

1. Read the strategy off the request — stated speed, quality, or cost constraints. Unstated defaults to `normal · normal · moderate`.
2. Answer the four questions below to fix the archetype.
3. Pick model + effort — `references/routing.md` for the archetype, `references/strategy.md` if the strategy is anything but default.
4. Write the handoff with the six-block contract — `references/prompt-contract.md`.
5. Spawn. If this shape of delegation recurs, persist it as an agent definition — `references/agent-defs.md`.

State the decision in one line before spawning, strategy first, so the derivation can be corrected and not just the outcome:
`→ [normal · top-notch · moderate] → sonnet-5 · medium effort · medium precision · return ≤400 tokens (locator, indirect refs matter)`

The bracket is the strategy you were given; everything after it is what that resolved to. Stating both means a wrong spawn can be traced to a misread constraint rather than re-argued from scratch.

## The four questions

| Question | If yes |
|---|---|
| 1. Could a competent stranger get this wrong *even with all the context*? | **Model up** |
| 2. Is the search space open-ended — unknown number of files, queries, or tool calls? | **Effort up** |
| 3. Is a confidently-wrong answer expensive to catch downstream? | **Effort up**; model up if the error would be one of judgment |
| 4. Am I running many of these at once, or is latency the binding constraint? | That is **`speed: fast`** — effort down, precision up, return tighter. Resolve it in `references/strategy.md` rather than adjusting by feel |

Question 1 separates *didn't know enough* from *didn't try hard enough* — the same split that governs escalation after a bad result. Questions 1–3 are about the task and hold regardless of budget; question 4 is a constraint, and the strategy knobs own it.

## Defaults

- **Strategy unstated → `normal · normal · moderate`.** That is the `standard` profile: the archetype table in `references/routing.md` exactly as written. Only reach for `strategy.md` when a real constraint was stated.
- **Unsure → Sonnet 5 at the model default effort (high).** Model defaults are tuned to what most people would want to spend. Reach for the dials deliberately, not reflexively.
- **Scoped, mechanical subagent → drop effort to `low`.** Narrow delegation is exactly where low effort is meant to live: fewer tool calls, straight to action, terse confirmation.
- **Exploratory subagent (repeated tool calls, wide search, long-horizon) → raise effort to `xhigh`.** This is the case where effort earns its cost.

These pull in opposite directions on purpose. "Subagent" is not one workload. Decide which kind you are spawning before setting effort.

## Before touching either dial

If a subagent underperforms, the first suspect is the handoff, not the settings. Check in this order:

1. **Context** — did it have the paths, IDs, prior decisions, and ruled-out dead ends?
2. **Scope** — was the task one coherent objective, or three stapled together?
3. **Tools** — did it have what it needed, and nothing ambiguous or overlapping?
4. **Conflicts** — do your prompt, the agent definition, and CLAUDE.md contradict each other? Conflicting instructions burn reasoning before any work starts.
5. **Strategy** — does the budget match the task? A cheap strategy on a judgment-bound archetype underperforms by design, not by accident. Surface that; don't quietly resolve it.

Only after those are clean: raise effort if it *skipped* something (didn't read the file, didn't run the tests, bailed partway). Raise the model if it *tried* and was still wrong.

Raising effort to compensate for a vague prompt is paying tokens to fix a context bug. Raising it past a stated cost ceiling is a decision to make out loud, not silently.

## What the subagent hands back

A subagent's value is a clean context window that returns a *distilled* result — not a transcript. It may burn tens of thousands of tokens internally and should still return a tight summary. Always state the return budget and format; effort does not reliably control response length, so prompt for it explicitly.

Rough budgets: locator ≤300 tokens · verification verdict ≤500 · research synthesis 1,000–2,000 · implementation report ≤800 plus the diff.
