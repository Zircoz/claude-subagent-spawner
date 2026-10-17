# The handoff contract

A subagent starts cold. Everything it needs must be in the prompt or reachable from it — and nothing else should be, because every token you send spends attention that the actual work needs.

Target: **the smallest set of high-signal tokens that reliably produces the outcome.** Minimal is not the same as short. Under-specifying and then paying effort to recover is worse than a fuller prompt.

## Six blocks

Drop any block that doesn't apply. Keep the order.

```
1. OBJECTIVE   One sentence. The outcome, not the procedure.
2. CONTEXT     Only what it cannot cheaply rediscover.
3. BOUNDARIES  Out of scope, off limits, read-only vs. write.
4. RETURN      Exact shape and token budget.
5. STOP        When to quit and report instead of pushing on.
6. TOOLS       Which ones, and roughly how many calls are reasonable.
```

**1. Objective.** State the result you want to hold in your hand. "Find why the retry loop double-charges" beats "investigate the retry loop." Never staple two objectives together — spawn twice.

**2. Context.** The high-value items, in rough priority: absolute file paths and identifiers · decisions already made and why · hypotheses already ruled out (this is the single biggest waste-saver) · the invariant that must survive · anything discovered this session that isn't written down anywhere.

Do not paste conversation transcripts, whole files it can open itself, or background it can infer from the repo. Prefer *pointers* to *payloads*: `see src/billing/retry.ts` and let it read, rather than inlining 400 lines. Folder structure, naming, and timestamps are themselves signal the subagent can use.

**3. Boundaries.** What not to touch, what not to install, whether it may write files or only read, whether it may commit. Boundaries are cheap and prevent the expensive class of subagent failure.

**4. Return.** Both shape and budget. The subagent may burn 50k tokens internally; you want 500 back.

```
Return exactly:
- VERDICT: pass | fail
- VIOLATIONS: file:line — one line each, max 8
- Nothing else. Under 400 tokens.
```

**5. Stop conditions.** "If you can't reproduce it in 15 minutes of tool calls, come back with what you ruled out." Without this, a high-effort subagent will keep going, and you get an expensive nothing instead of a cheap negative result.

**6. Tools.** Name the ones it needs. An ambiguous or overlapping toolset is a real failure mode — if you can't say which tool applies in a given situation, it can't either.

## The precision dial

Precision is the inverse of model capability. Match it or you lose on both ends: an under-specified Haiku wanders, an over-specified Opus is boxed out of the insight you spawned it for.

Precision is also where a cost ceiling gets paid back. When budget caps the model tier, the work moves into the handoff: the globs, the schema, the tie-breaks, the stop condition. It is the one lever that gets *cheaper* as you lean on it harder.

| | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 | Fable 5.1 |
|---|---|---|---|---|
| **Objective** | Imperative, one action, plus success criteria | Outcome + success criteria | Outcome + why it matters | The problem, honestly stated, and who it's for |
| **Method** | Prescribe the steps | Suggest an approach | Say nothing about method | Say nothing about method; step lists *reduce* its quality |
| **Examples** | Give 1–2 canonical ones | One if the format is unusual | Usually skip; they narrow the search space | Skip |
| **Output** | Rigid schema | Schema or clear shape | Shape + budget | Budget |
| **Judgment calls** | Enumerate and resolve them in advance | Name the tie-breaker | Hand it the constraints, let it decide | Hand it the constraints |
| **Boundaries** | Explicit stop condition and tool budget | Explicit stop condition | Off-limits list | Off-limits list, spelled out; it takes adjacent actions otherwise |

Haiku 5.5 follows instructions far better than Haiku 4.5 did, so it tolerates outcome-plus-criteria phrasing. It still pays to resolve judgment calls for it in advance, though: precision is where its 20× price advantage gets earned back.

For capable models, replace examples with a well-designed interface: expressive parameters, self-describing field names, an enum whose values imply their use. A tool signature teaches more than three worked examples, and doesn't fence in the exploration.

**Prefer references over descriptions.** A failing test is a better spec than a paragraph about correctness. An HTML mockup beats a description of the design. A rubric file beats "make it good." Point the subagent at a file; don't paraphrase it.

## Lines that fix known quirks

Each of these patches a failure documented for the current models. Add one only when its condition applies, because every line spends attention.

**Verification:** for Sonnet 5.5 or Haiku 5.5 at `low`/`medium` doing code changes. At those levels they sometimes report "done" without running anything.
```
When you change code that can be run, built, or type-checked, run a real check
that exercises the change before reporting it done: the project's tests,
type-checker, or build. A syntax-only check, or a check that failed to start,
does not count. If no real check can run, say which one you didn't run and why.
```

**Finish the job:** for Haiku 5.5 at `low` under a long system prompt, or any autonomous run that keeps stopping to ask.
```
Keep working until everything asked for is done; stop to ask only when you
can't go on without input or before a risky step. When the work is done and
checked, stop and report. Don't add features or refactors that weren't asked for.
```

**Search grounding:** for Haiku 5.5 (and anything at `low`) with a search tool.
```
The current date is <date>. Your training data ends well before it. Versions,
prices, rules, and anything "latest" may have changed, so search for those
even when you feel sure.
```

**Don't discourage tools:** for Sonnet 5.5, which follows "minimize tool calls" literally. Cap tool use with a budget ("~15 calls"), not a mood ("only when strictly necessary").

**Boundaries and grounded claims:** for Fable 5.1 above all, and for any long autonomous run.
```
Before reporting progress, audit each claim against a tool result from this
session. If something is not verified, say so. Don't take actions outside the
task: no extra branches, no drafts, no cleanup of adjacent code.
```

**Give the reason:** for Fable 5.1. It does better when it knows what the output enables. Add one line: "This is for <who>; they need it to <do what>."

## Verbosity, both directions

*Inbound* (what you write): high-signal only. Cut restated background, throat-clearing, and defensive caveats. Say each thing once — repetition across your prompt, the agent definition, and CLAUDE.md creates conflicts the subagent has to resolve before it can start.

*Outbound* (what it returns): always budgeted, because effort won't do it for you. Give a token ceiling and a shape.

| Subagent kind | Ask for |
|---|---|
| Locator / search | Bare list of `file:line` + one-clause note. ≤300 tok. |
| Extraction / classification | The schema, nothing around it. |
| Verification | Verdict first, then violations, then nothing. ≤500 tok. |
| Research | Findings with sources; conclusion first. 1,000–2,000 tok. |
| Implementation | What changed and why, plus anything surprising. ≤800 tok + diff. |
| Debugging | Root cause, the evidence, the fix. Ruled-out branches in one line each. |

Ask for "conclusion first, evidence second" as a rule — it keeps the useful part above the point where you stop reading.

## Anti-patterns

- **Transcript dumping.** Pasting the conversation so far. Distill it or point at files.
- **Two objectives, one agent.** The returned summary will be a compromise of both.
- **Contradicting yourself across layers.** Prompt says "document thoroughly," CLAUDE.md says "never write comments." Reconcile before spawning.
- **Rules where judgment belongs.** Hardcoded if-else guidance is brittle and often wrong at the edges. Modern models handle "match the surrounding code's conventions" better than a list of conventions.
- **Vagueness disguised as trust.** The opposite failure: "improve the auth flow" with no constraints and no definition of done.
- **No return contract.** You get a wall of prose and re-read the work you delegated to avoid reading.
- **Asking a verifier to also fix things.** Then it grades its own work. Separate agents.
- **Fan-out without a merge plan.** Decide how you'll reconcile eight results before you spawn eight agents.

## Worked examples

**Locator: Haiku 5.5, low effort, maximum precision**

```
Find every call site of `chargeCard` in this repo.

Search src/ and packages/ only. Ignore tests, fixtures, and node_modules.
Include re-exports and dynamic references (string keys, wrappers).

Return: a list of `path:line — enclosing function name`. Nothing else, under 300 tokens.
If you find more than 40, stop and report the count plus the first 40.

Tools: Grep, Glob, Read.
```

**Debugging: Opus 5.5, high effort, minimum precision**

```
Customers on the annual plan are charged twice when a payment retry
overlaps a plan change. Repro: scripts/repro_double_charge.sh (fails ~1 in 3).

Already ruled out: the Stripe idempotency key (verified unique per attempt),
and the webhook handler (logs show a single inbound event).
Billing state machine lives in src/billing/; the invariant that must hold is
one `charges` row per (subscription_id, period).

Find the root cause. Read and run whatever you need; do not change any code.
If you can't isolate it, come back with the narrowed hypothesis space.

Return under 1,000 tokens: root cause, the evidence that proves it,
the smallest fix. One line per branch you ruled out.
```

The difference isn't length. It's that the second one specifies the *situation* exhaustively and the *method* not at all.
