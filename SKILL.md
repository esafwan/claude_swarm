---
name: claude_swarm
description: Orchestrate a large task as a dependency-mapped TODO graph, executed by a swarm of cheap Haiku agents, steered by Sonnet 5 Low, with Opus 5 Low reserved for genuinely hard nodes and one final adversarial review. Use whenever a task is big enough to decompose into independently verifiable subtasks — multi-file features, refactors, migrations, bulk audits, test-suite buildouts. Triggers on: "swarm this", "use claude_swarm", "parallelize this with agents", "break this into tasks and farm it out".
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

## Filesystem isolation — mandatory before any parallel dispatch

Step 1 of the Algorithm ("decompose into a TODO graph") is incomplete without this: a
dependency edge is not just "task B needs task A's output," it is also "task B and task A
touch the same file." **Before dispatching anything in parallel, classify every pair of
ready tasks by file-scope overlap:**

1. **List the files/interfaces each task is expected to touch** as part of writing its
   scope (Algorithm step 2 already requires this — reuse it for this check, don't skip it
   because it feels redundant).
2. **Disjoint scopes → dispatch normally**, same shared working directory, no isolation
   needed. This is the common case and most of this skill's cost model depends on it
   staying cheap — don't over-isolate.
3. **Overlapping scopes (especially anything touching shared/global files — a design
   system's shared component library, a shared config, a shared schema/types file) →
   you MUST do one of:**
   - **Serialize** the overlapping tasks (run them one after another, not concurrently) if
     the overlap is small and the tasks are cheap, or
   - **Isolate** each overlapping task in its own git worktree (pass `isolation: "worktree"`
     to the `Agent` tool call) and merge the results back yourself once each returns —
     this is the default choice once more than ~2 tasks share risky overlap, or the shared
     file is large/central enough that a silent last-writer-wins clobber would be expensive
     to detect.
   - Never dispatch overlapping-scope tasks concurrently onto the same shared working
     directory and rely on each worker "being careful" or on a later diff review to catch
     collisions. Two agents editing, `git add`-ing, and committing in the same working tree
     at the same time can interleave in ways a post-hoc diff review will not reliably catch
     — this is not a theoretical risk, it is the normal failure mode of shared-checkout
     concurrency.
4. **A task that runs its own internal claude_swarm** (i.e. a sub-agent you dispatched is
   itself decomposing further and spawning its own Haiku workers) inherits this same
   obligation one level down — tell it explicitly, in its dispatch prompt, to run this same
   overlap classification before its own parallel fan-out, not just at your top level.
5. When in doubt about whether two tasks overlap (e.g. both touch "the design system" in
   some vague sense but you haven't confirmed the exact files), **default to isolating** —
   the cost of an unnecessary worktree is small; the cost of a missed collision on a shared
   checkout is a silently corrupted or half-clobbered file that may not surface until much
   later.

## Commit traceability — tag every commit with its task ID

When multiple tasks land commits into the same shared checkout (the common case when you
chose "serialize" over "isolate" in the previous section, or whenever several tasks finish
close together), give every task's dispatch prompt this instruction: **append a trailer
line at the very end of its commit message** (after the body, on its own line, still plain
human-readable text) naming the task:

```
Track-Item: <task-id-or-number>
```

Use whatever ID scheme your TODO graph already uses (a number, a short slug — anything
stable and unique within this run). This costs nothing and pays for itself the moment you
need to reconcile: `git log --grep "Track-Item: 7"` instantly answers "which commit(s) came
from task 7," without you having to remember timing, diff through unrelated commits, or ask
a completed (and possibly gone) worker to remind you. It also lets a task that runs its own
internal commit (e.g. after amending to fix a review finding) keep the same trailer, so the
final `git log` reads as a task-indexed history of the whole run, not an anonymous stream of
"fix" commits. Tell each dispatched task about this requirement explicitly — it will not
happen by default, and a task that already committed before receiving the instruction should
`git commit --amend` to add the trailer rather than leave it untagged.

## When to use the Agent tool's `model` param

- Default: omit — inherit session model — is wrong here. This skill is explicitly about deliberate model-tiering, so set `model: "haiku"` on the default worker calls, and escalate explicitly (`sonnet`, low/medium effort, `opus`) per the ladder above.
- Never route a bounded, well-specified task straight to Opus "to be safe" — that defeats the cost model this skill exists for.

## Why this shape (see also the repo README for the full argument)

Most-parallel-cheap-workers-with-narrow-scope-and-progressive-escalation beats "one strong model does everything serially" on cost and wall-clock for any task that decomposes cleanly, because:
- Bounded tasks with explicit acceptance criteria are exactly what smaller models are reliable at; the failure mode of small models (drift, scope creep, silent wrongness) is neutralized by narrow scope + self-check + peer-review + Sonnet spot-check.
- Escalation is pay-as-you-go: expensive reasoning is spent only on nodes that proved they need it, not on the whole task up front.
- Parallelism collapses wall-clock time for anything with independent subtasks, which serial single-model execution can't do regardless of model quality.
