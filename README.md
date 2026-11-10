# claude_swarm

A Claude Code skill that turns a large task into a dependency-mapped TODO graph and
executes it with a swarm of cheap `Haiku` agents, steered continuously by `Sonnet 5 Low`,
with `Opus 5 Low` reserved for genuinely hard nodes and one final adversarial review pass.

Install: drop `SKILL.md` into `~/.claude/skills/claude_swarm/` (or your project's
`.claude/skills/claude_swarm/`), then invoke it in Claude Code with
`/claude_swarm` or by asking to "swarm this" / "parallelize this with agents".

## The core pattern

```
You (plan + own the graph)
  -> Haiku swarm executes bounded tasks + peer-reviews each other
  -> Sonnet 5 Low unblocks difficult branches
  -> Haiku verifies broadly (tests, regressions, security, edge cases, contracts)
  -> Opus 5 Low performs one high-value adversarial review at the end
  -> You convert findings into new tasks and close the loop
```

Escalation ladder per task node, used only on demonstrated difficulty:
`Haiku -> Sonnet 5 Low -> Sonnet 5 Medium -> Opus 5 Low`.

## Why I think this is faster and cheaper (and roughly as good, or better)

This is my honest read, not a marketing claim — treat the numbers as directional, not benchmarked.

**Cheaper, mechanically.** Haiku is roughly an order of magnitude cheaper per token than
Sonnet, and multiple orders cheaper than Opus. If a task decomposes into N independently
verifiable subtasks, and even 80% of them are within a small model's competence when scoped
tightly (explicit files, explicit acceptance criteria, narrow diff), you've moved 80% of the
token spend to the cheapest tier. Escalation is pay-as-you-go: expensive reasoning is bought
only for the fraction of nodes that prove they need it, never for the whole task up front —
which is the opposite of "run everything on the strongest model just in case."

**Faster, mechanically.** Wall-clock for a serial single-model run is the sum of every
subtask's latency. Wall-clock for a replenished worker pool is close to the critical path
through the dependency graph, because independent branches run concurrently and a freed
slot is refilled immediately rather than waiting for a whole phase to close. For any task
with real parallelism in its dependency graph — most multi-file features, migrations, and
audits — this is a large constant-factor win regardless of which model does the work.

**Why quality doesn't collapse.** The usual worry with cheap models is silent drift or
wrong-but-plausible output. This pattern targets that directly, not by hoping Haiku is smart
enough, but by constraining the conditions under which it has to be:
- Tasks are scoped small and bounded, with explicit acceptance criteria — exactly the regime
  where smaller models are reliable, since they don't have to hold a large ambiguous context.
- Every worker self-checks against its own acceptance criteria before returning.
- A second Haiku independently reviews the first's diff before it reaches the steering layer.
- The steering layer (Sonnet Low) inspects diffs and evidence, not raw transcripts, so a
  cheap mistake gets caught structurally rather than relying on the same model that made it
  to notice.
- The most expensive model in the system (Opus) is spent once, at the end, on adversarial
  review against real diffs and real test results — the highest-leverage place to spend it,
  since by then most of the surface area has already been narrowed down for it.

**Where this doesn't help.** Tasks that are one indivisible reasoning problem (no dependency
graph, no parallel subtasks, genuinely ambiguous architecture) get no benefit from swarming —
decompose-and-parallelize only pays off when the task actually decomposes. The skill says
this explicitly: don't reach for Opus to produce the plan unless the architecture is
genuinely ambiguous, and don't escalate a whole feature just because one node in it is hard.

## Adaptive workflow

Use the lightest workflow that fits the risk. Small, clear changes get focused inspection and targeted checks. Multi-file features get repository reconnaissance, a dependency graph, bounded workers, and evidence-based review. Architecture-heavy or high-impact work gets design and trade-off review before implementation. Runtime checks complement automated tests for user-visible or integration behavior when the environment supports it.

## Benchmarking against alternatives

The cost and speed arguments above are architectural expectations, not measured results. Use [`BENCHMARK.md`](BENCHMARK.md) to compare this workflow with alternatives such as pstack on the same representative tasks. Record workflow and model versions, token usage or cost, elapsed and human time, verification results, and review findings. Do not publish expected savings as measured results.

## Practices incorporated

The workflow combines cost-aware delegated execution with risk-based design and verification, adapting selected ideas from [pstack](https://github.com/backnotprop/pstack) and [Matt Pocock's engineering skills](https://github.com/mattpocock/skills). It uses outcome clarification for material ambiguities, vertical-slice task graphs, test-first behavior checks at public seams, reproduce-first bug fixing, separate spec and standards reviews, and runtime checks alongside automated tests. Cost and speed remain unmeasured until comparable runs are completed.

## Files

- `SKILL.md` — the skill definition Claude Code loads.
- `CMAX.md` — the `cmax` alias, kept in sync with `SKILL.md`.
- `BENCHMARK.md` — a repeatable comparison protocol.
