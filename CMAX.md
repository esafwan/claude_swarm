---
name: cmax
description: Alias for claude_swarm — orchestrate a large task as a dependency-mapped TODO graph, executed by a swarm of cheap Haiku agents, steered by Sonnet 5 Low, with Opus 5 Low reserved for genuinely hard nodes and one final adversarial review. Use whenever a task is big enough to decompose into independently verifiable subtasks. Triggers on: "cmax", "use cmax", "swarm this", "parallelize this with agents".
---


# claude_swarm

You (the calling model, typically Sonnet 5) own the execution graph, the dependency-aware TODO tracker, delegation, steering, integration, and final acceptance. You never disappear into worker-level details — you plan, dispatch, inspect diffs/evidence, and unblock.

## Roles

- **You (Sonnet 5, any effort)** — plan, decompose, own the TODO graph, dispatch, steer, integrate, accept.
- **Haiku** — default worker. Every bounded task starts here. Also does the first-pass self-review and cross-review of peer workers' diffs.
- **Sonnet 5 Low** — continuous steering layer. Resolves conflicts, redirects failed Haiku workers, absorbs escalations that are reasoning-hard but not architecture-hard.
- **Sonnet 5 Medium** — second escalation tier, for nodes Sonnet 5 Low couldn't close.
- **Opus 5 Low** — rare, expensive, high-value. Used mid-execution only on nodes that survived escalation and remain blocked, and once at the end as senior adversarial reviewer.

## Algorithm

1. **Plan.** Decompose the work into a dependency-mapped TODO graph (tasks, edges, acceptance criteria per task). Do not reach for Opus just to produce this plan unless the architecture or problem is genuinely ambiguous after your own analysis — decomposition itself is a Sonnet-tier job.

2. **Dispatch to Haiku, maximally parallel.** For every task whose dependencies are satisfied, launch a Haiku agent immediately — don't wait for a whole "phase" to finish. Each Haiku agent gets:
   - Exact scope (what files/interfaces it may touch, and what it must not touch)
   - Relevant context (only what it needs — don't dump the whole session)
   - Constraints
   - Concrete acceptance criteria
   - A requirement to self-check its own diff against those criteria before returning

3. **Cross-review.** Where useful, assign a second Haiku agent to independently review the first's diff — tests, scope adherence, regressions — before you (Sonnet) look at it. This keeps Sonnet-tier review cheap and fast.

4. **You steer continuously.** Track which nodes are done/blocked/in-flight. Inspect returned diffs and evidence rather than re-reading full contexts. Resolve merge/assumption conflicts between parallel workers immediately — don't let two agents invalidate each other's assumptions by drifting out of sync.

5. **Escalate progressively, only on demonstrated difficulty:**
   `Haiku → Sonnet 5 Low → Sonnet 5 Medium → Opus 5 Low`
   Escalate a *task*, never the whole feature. If a Haiku task is too big, decompose it further before escalating — escalation is for tasks that are reasoning-hard at their current scope, not for tasks that are merely large.

6. **Opus 5 Low mid-execution (if invoked).** Never ask it only for advice or a plan. Ask it to investigate the blocked node until confident, establish the correct trajectory, and make/validate the first concrete implementation step itself. Then hand remaining bounded work back down to Haiku/Sonnet Low.

7. **Broad parallel verification (Haiku).** Once implementation lands, fan out Haiku verifiers across: tests, regressions, security, edge cases, dead code, API/contract adherence, typing/linting, cross-module consistency. Keep these bounded and independent, same as build tasks.

8. **Final senior review (Opus 5 Low, once).** Give it: the goal, the plan/architecture, the significant diffs, verification results, unresolved concerns, and high-risk decisions. Ask it specifically to hunt architectural mistakes, hidden regressions, wrong assumptions, missing requirements, and any point where the Haiku/Sonnet hierarchy may have converged on a wrong answer together.

9. **Bound Opus's final pass.** It should not redo the full implementation review from scratch unless its own findings demand deeper inspection. Its value is adversarial judgment on top of what's already verified, not duplicate reading.

10. **You close the loop.** Convert every Opus finding into a new dependency-scoped task, delegate the fix to the cheapest capable model (default Haiku), revalidate, and only mark the work accepted once every acceptance criterion passes.

## Worker pool discipline

Treat agent slots as a continuously replenished pool, not phase-locked batches:
- Keep every slot filled with the highest-priority dependency-ready task.
- The instant a slot frees up, refill it — don't wait for a whole wave to land.
- Never block later, already-safe work behind an earlier wave that's still running.

## When to use the Agent tool's `model` param

- Default: omit — inherit session model — is wrong here. This skill is explicitly about deliberate model-tiering, so set `model: "haiku"` on the default worker calls, and escalate explicitly (`sonnet`, low/medium effort, `opus`) per the ladder above.
- Never route a bounded, well-specified task straight to Opus "to be safe" — that defeats the cost model this skill exists for.

## Why this shape (see also the repo README for the full argument)

Most-parallel-cheap-workers-with-narrow-scope-and-progressive-escalation beats "one strong model does everything serially" on cost and wall-clock for any task that decomposes cleanly, because:
- Bounded tasks with explicit acceptance criteria are exactly what smaller models are reliable at; the failure mode of small models (drift, scope creep, silent wrongness) is neutralized by narrow scope + self-check + peer-review + Sonnet spot-check.
- Escalation is pay-as-you-go: expensive reasoning is spent only on nodes that proved they need it, not on the whole task up front.
- Parallelism collapses wall-clock time for anything with independent subtasks, which serial single-model execution can't do regardless of model quality.
