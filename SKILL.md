---
name: subagent-spawner
description: Choose the model, effort level, and prompt shape for a subagent before spawning it. Use when delegating work to a Task/subagent, fanning out parallel agents, deciding between Opus 5 / Sonnet 5 / Haiku 4.5 / Fable 5, writing or tuning a .claude/agents definition, or when a subagent came back shallow, over-thought, off-target, or too verbose.
---

# Subagent spawner

Three dials, set together:

- **Model** — how *capable*. Frozen weights: what it knows, and how well it tolerates ambiguity.
- **Effort** — how *thorough*. How many files it reads, tools it calls, hypotheses it checks, and steps it takes before reporting back. Not just thinking time.
- **Prompt** — everything the subagent cannot cheaply rediscover, plus the exact shape of what comes back.

They trade against each other. Opus at low effort is five minutes with an expert: real recognition, shallow reading. Sonnet at high effort is a strong generalist given the whole afternoon: thorough on *your* specifics, weaker on "I've seen this exact trap before." Fable even at low effort is the specialist who glances and spots what nobody else would.

The prompt closes the gap: **the less capable the model, the more prescriptive the handoff must be.** Capable models handle ambiguity and resent being over-constrained; smaller models need execution spelled out.

## Strategy (optional shortcut)

Before the four questions, you can pick a **strategy** — a named preset that sets the starting model and effort based on what you care about most.

| Strategy | Speed | Quality | Cost | → Model | → Effort |
|---|---|---|---|---|---|
| **blitz** | fast | normal | cheap | Haiku 4.5 | n/a (fixed) |
| **balanced** | normal | normal | moderate | Sonnet 5 | medium |
| **thorough** | normal | normal | moderate | Sonnet 5 | xhigh |
| **premium** | normal | top-notch | richie-rich | Opus 5 | xhigh |
| **frontier** | normal | top-notch | richie-rich | Fable 5 | high |
| **quick-premium** | fast | top-notch | moderate | Opus 5 | low |

Pick by expressing what you care about across three dimensions: **speed** (`fast` / `normal`), **quality** (`top-notch` / `normal`), **cost** (`cheap` / `moderate` / `richie-rich`). If your signals conflict (e.g., top-notch quality + cheap cost), resolve by priority: **quality > speed > cost**, and note the conflict in the decision trace.

The strategy sets the starting dials. The four questions below refine them — they can push the model up or effort up from wherever the strategy landed.

Full strategy details and conflict examples → `references/routing.md`.

## Procedure

1. Pick a strategy (optional — skip to step 2 if you know the dials directly).
2. Answer the four questions below to refine.
3. Pick model + effort — `references/routing.md`.
4. Write the handoff with the six-block contract — `references/prompt-contract.md`.
5. Spawn. If this shape of delegation recurs, persist it as an agent definition — `references/agent-defs.md`.

State the decision in one line before spawning, so it can be corrected:
`→ [balanced] sonnet-5 · medium effort · precise instructions · return ≤400 tokens (locator fan-out ×4)`

## The four questions

| Question | If yes |
|---|---|
| 1. Could a competent stranger get this wrong *even with all the context*? | **Model up** |
| 2. Is the search space open-ended — unknown number of files, queries, or tool calls? | **Effort up** |
| 3. Is a confidently-wrong answer expensive to catch downstream? | **Effort up**; model up if the error would be one of judgment |
| 4. Am I running many of these at once, or is latency the binding constraint? | **Both down**, precision up |

Question 1 separates *didn't know enough* from *didn't try hard enough* — the same split that governs escalation after a bad result.

## Defaults

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

Only after those are clean: raise effort if it *skipped* something (didn't read the file, didn't run the tests, bailed partway). Raise the model if it *tried* and was still wrong.

Raising effort to compensate for a vague prompt is paying tokens to fix a context bug.

## What the subagent hands back

A subagent's value is a clean context window that returns a *distilled* result — not a transcript. It may burn tens of thousands of tokens internally and should still return a tight summary. Always state the return budget and format; effort does not reliably control response length, so prompt for it explicitly.

Rough budgets: locator ≤300 tokens · verification verdict ≤500 · research synthesis 1,000–2,000 · implementation report ≤800 plus the diff.
