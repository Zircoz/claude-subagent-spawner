# subagent-spawner

A Claude Code skill for deciding **how** to delegate before you delegate: which model, what effort level, and how precisely to write the handoff prompt — resolved from the kind of task it is and the constraints you're under.

Spawning a subagent is three decisions, not one. Most of the cost of getting it wrong shows up as a subagent that read three files when it needed thirty, or one that burned Opus tokens on a rename, or one still running at an effort level tuned for last generation's models. This skill makes the decision explicit and consistent, and it takes your actual constraints as input: say *fast*, *cheap*, or *top-notch* and it resolves that into a concrete spawn.

```
→ [normal · top-notch · richie-rich] → opus-5.5 · max + fable-5.1 · high in parallel · minimum precision · return ≤1000 tok each   (deep debugging)
→ [fast · normal · cheap] → haiku-5.5 · low · maximum precision · return ≤300 tok                                               (locator, ×6 parallel)
```

## Install

```bash
git clone https://github.com/Zircoz/claude-subagent-spawner ~/.claude/skills/subagent-spawner
```

Project-scoped instead:

```bash
git clone https://github.com/Zircoz/claude-subagent-spawner .claude/skills/subagent-spawner
```

Or install via Vercel's skill registry:

```bash
npx skills add Zircoz/claude-subagent-spawner
```

Restart Claude Code, or run `/skills` to confirm it's loaded. It triggers on its own when you're about to spawn a subagent; invoke it directly with `/subagent-spawner`.

## What's in it

| File | Contents |
|---|---|
| `SKILL.md` | The entry point. Four classification questions, defaults, and the "fix the prompt before touching the dials" rule. |
| `references/routing.md` | Model characters and prices, effort levels with per-model notes and known quirks, a previous-generation mapping, an archetype table (locator → frontier), and the escalation heuristic. |
| `references/strategy.md` | The speed/quality/cost knobs, how contradictory combinations resolve, and an archetype × profile table for every non-default strategy. |
| `references/prompt-contract.md` | Six-block handoff template, the precision dial, lines that fix known model quirks, return-budget table, anti-patterns, worked examples. |
| `references/agent-defs.md` | Frontmatter for `.claude/agents/*.md`, five ready-made definitions, fan-out patterns. |

Loaded progressively — `SKILL.md` is short on purpose, and the references are pulled in only when the decision needs them.

## The core idea

**Model** is how capable. **Effort** is how thorough. **Precision** is how much of the method you specify — and it runs *inverse* to model capability. Over-specify Opus and you fence it out of the insight you spawned it for; under-specify Haiku and it wanders.

Two things set those three: the **archetype** (what kind of task this is) and the **strategy** — `speed`, `quality`, `cost`. Strategy is upstream of the dials, not a fourth one. Precedence is **cost ceiling > quality floor > speed lever**: speed buys down effort rather than the model tier, spending more without asking for more buys nothing, and cost is the only knob that can tell you a task isn't affordable rather than quietly shipping a worse one.

When a subagent disappoints, the diagnostic is: did it not *know* enough, or not *try* hard enough? Knowledge → bigger model. Effort → higher effort. But check the handoff first; raising effort to compensate for a vague prompt is paying tokens to fix a context bug.

## Models covered

Haiku 5.5, Sonnet 5.5, Opus 5.5, and Fable 5.1, the current generation as of October 2026. The previous generation (Haiku 4.5, Sonnet 5, Opus 5, Fable 5) is mapped by role in `references/routing.md`, including Haiku 4.5's lack of an effort parameter.

What changed with the 5.5 generation, and why the tables moved:

- **Effort shifted down a level.** Opus 5.5 at `medium` beats Opus 5 at `high`, and Sonnet 5.5 at `medium` beats Sonnet 5 at its best for under a tenth of the cost per task. The tables now start lower and reserve `xhigh`/`max` for measured gains.
- **Haiku became a real subagent tier.** Haiku 5.5 takes effort levels, follows instructions well, and costs 1/20th of Sonnet, so economy and fan-out work moved onto it. Mind its price cliff for prompts over 100K tokens.
- **Opus is only 2× Sonnet now**, so `premium` reaches for it sooner.
- **Fable 5.1's niche narrowed.** Opus 5.5 matches or beats it on Anthropic's published benchmarks at 40% of the price. Fable is for when Opus has failed, for multi-hour autonomous runs, and for an independent second opinion.
- **Verifiers default to `medium`.** In CodeRabbit's review benchmarks, lower-effort Opus 5.5 was the most precise reviewer, and higher effort found different bugs rather than strictly more of them.

## Sources

Built from Anthropic's published guidance:

- [Effort](https://platform.claude.com/docs/en/build-with-claude/effort) — Claude Platform docs
- [Choosing a Claude model and effort level in Claude Code](https://claude.com/blog/claude-model-and-effort-level-in-claude-code) — Lydia Hallie
- [The new rules of context engineering for Claude 5 generation models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) — Thariq Shihipar
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Applied AI
- [Introducing Claude Opus 5.5](https://www.anthropic.com/claude-opus-5-5), [Claude Sonnet 5.5](https://www.anthropic.com/claude-sonnet-5-5), [Claude Haiku 5.5](https://www.anthropic.com/claude-haiku-5-5), [Claude Fable 5.1](https://www.anthropic.com/claude-fable-and-mythos-5-1) — Anthropic launch posts and benchmark tables
- [Model configuration](https://code.claude.com/docs/en/model-config) and [Create custom subagents](https://code.claude.com/docs/en/sub-agents) — Claude Code docs (aliases, effort defaults, frontmatter)

And from independent code-review benchmarks:

- [Claude Opus 5.5 Code Review Benchmarks](https://www.coderabbit.ai/blog/opus-5-5-model-review), [Sonnet 5.5 vs Opus 5.5](https://www.coderabbit.ai/blog/sonnet-5-5-model-review), [Fable 5.1 Review](https://www.coderabbit.ai/blog/fable-5-1-model-review) — CodeRabbit

Agent frontmatter accepts `effort` in current Claude Code; older CLIs and harnesses may not, so `references/agent-defs.md` includes a prompt-steering fallback. Vendor benchmark numbers are self-reported; the tables use them for direction, not as precise rankings. The `strategy:` frontmatter key is a convention of this skill, not something the CLI reads — it records why a `model` and `effort` were chosen, and those two fields still have to carry the resolved values.

## License

MIT
