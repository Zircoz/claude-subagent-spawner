# subagent-spawner

A Claude Code skill for deciding **how** to delegate before you delegate: which model, what effort level, and how precisely to write the handoff prompt.

Spawning a subagent is three decisions, not one. Most of the cost of getting it wrong shows up as a subagent that read three files when it needed thirty, or one that burned Opus tokens on a rename. This skill makes the decision explicit and consistent.

```
→ opus-5 · xhigh effort · minimum precision · return ≤1000 tok   (deep debugging)
→ haiku-4.5 · fixed · maximum precision · return ≤300 tok        (locator, ×6 parallel)
```

## Install

```bash
git clone https://github.com/<you>/subagent-spawner ~/.claude/skills/subagent-spawner
```

Project-scoped instead:

```bash
git clone https://github.com/<you>/subagent-spawner .claude/skills/subagent-spawner
```

Restart Claude Code, or run `/skills` to confirm it's loaded. It triggers on its own when you're about to spawn a subagent; invoke it directly with `/subagent-spawner`.

## What's in it

| File | Contents |
|---|---|
| `SKILL.md` | The entry point. Four classification questions, defaults, and the "fix the prompt before touching the dials" rule. |
| `references/routing.md` | Model characters, effort levels with per-model notes, an archetype table (locator → frontier), and the escalation heuristic. |
| `references/prompt-contract.md` | Six-block handoff template, the precision dial, return-budget table, anti-patterns, worked examples. |
| `references/agent-defs.md` | Frontmatter for `.claude/agents/*.md`, five ready-made definitions, fan-out patterns. |

Loaded progressively — `SKILL.md` is short on purpose, and the references are pulled in only when the decision needs them.

## The core idea

**Model** is how capable. **Effort** is how thorough. **Precision** is how much of the method you specify — and it runs *inverse* to model capability. Over-specify Opus and you fence it out of the insight you spawned it for; under-specify Haiku and it wanders.

When a subagent disappoints, the diagnostic is: did it not *know* enough, or not *try* hard enough? Knowledge → bigger model. Effort → higher effort. But check the handoff first; raising effort to compensate for a vague prompt is paying tokens to fix a context bug.

## Models covered

Opus 5, Sonnet 5, Haiku 4.5, Fable 5. Note that Haiku 4.5 doesn't take an effort parameter — the skill handles that case.

## Sources

Built from Anthropic's published guidance:

- [Effort](https://platform.claude.com/docs/en/build-with-claude/effort) — Claude Platform docs
- [Choosing a Claude model and effort level in Claude Code](https://claude.com/blog/claude-model-and-effort-level-in-claude-code) — Lydia Hallie
- [The new rules of context engineering for Claude 5 generation models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) — Thariq Shihipar
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic Applied AI

Effort-parameter placement in agent frontmatter has shifted across CLI versions; verify against your own `/agents` before relying on it. `references/agent-defs.md` includes a prompt-steering fallback.

## License

MIT
