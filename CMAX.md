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

## Risk-adaptive workflow — use only the depth the task needs

Before decomposing, classify the task by ambiguity, impact, and verification risk. Do not run the full multi-agent workflow mechanically for every request.

- **Small and low-risk** (a localized change with clear requirements and an obvious check): dispatch a single Haiku worker with a narrow scope, run the relevant checks, and summarize evidence. Skip architecture agents and parallel fan-out; the Opus final review may be a brief diff-only pass but is not skipped.
- **Normal feature work** (multiple files or components, clear boundaries): understand the existing code and conventions, map dependencies and acceptance criteria, then use the normal worker and review workflow below.
- **High-ambiguity or high-impact work** (new subsystem, public contract, data migration, security boundary, major architectural choice): first investigate constraints and existing patterns; produce a concise design with interfaces, trade-offs, risks, and open questions. Use an independent architecture challenge or competing proposals only where a real decision remains. Resolve material design questions before parallel implementation.

Escalate based on demonstrated uncertainty or failure, not task size alone. If the request is a prototype or explicitly prioritizes speed, keep the design proportionate and label shortcuts that affect production readiness.

## Understand and design before decomposition

For feature work, first inspect the relevant repository structure, existing implementations, tests, configuration, and recent decisions. Identify constraints, conventions, affected interfaces, and what should remain unchanged. State what you found and what assumptions still need validation.

For architecture-heavy work, define module boundaries, data flow, contracts, failure behavior, and migration or compatibility needs before creating the task graph. Record a short decision note when a choice is consequential or likely to be revisited; do not create ceremony for a routine local change. Compare alternatives with a separate reviewer or parallel proposals only when the trade-offs are material and unresolved.

## Align on the outcome and shape the work

Before implementation, capture the user's intended outcome, key actors or scenarios, constraints, and observable acceptance criteria. If a material ambiguity could change the solution, restate the goal in your own words and ask only the question needed to resolve it; otherwise proceed with a clear, lightweight assumption. Do not turn a clear request into a long interview or make the user approve routine internal planning.

For substantial work, define the test seams and the smallest useful end-to-end behavior before splitting tasks. Prefer existing public interfaces and the highest practical seam. Shape the DAG as **vertical slices** that deliver complete, independently verifiable behavior across affected layers; avoid batches that only create a horizontal layer with nothing demonstrable. Each slice should fit a fresh worker context and state its user-facing result and blocking dependencies. Keep the task graph in the session by default; use project issues or persisted planning files only when the user or project workflow calls for them.

A wide mechanical refactor is an exception to vertical slicing. Use expand–migrate–contract: introduce the compatible new form, migrate callers in bounded batches while keeping checks green, then remove the old form only after all callers have moved. Add an integration checkpoint when intermediate batches cannot remain green independently.

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

10. **You close the loop.** Convert every accepted finding into a dependency-scoped task, delegate the fix to the cheapest capable model (default Haiku), revalidate, and only mark the work accepted once every acceptance criterion passes.

## Test and debugging discipline

For behavior changes, use a red–green–refactor loop where the repository has a practical test seam: write one meaningful failing test for an agreed acceptance behavior, implement only enough to pass it, then simplify without changing behavior. Work one vertical slice at a time instead of writing a large imagined test suite before learning from implementation. Tests should exercise public behavior at the highest practical seam, use the project's domain language and existing glossary where present, and avoid assertions coupled to private implementation or mocks of internal collaborators. If a new test seam is consequential or changes what behavior is being promised, surface that choice before committing to it.

For bugs, reproduce the failure first with a failing test or a minimal repeatable case, trace the cause, then fix it and prove the reproduction passes. If the symptom cannot be reproduced, report that limitation and gather the next most useful evidence rather than claiming a verified fix.

## Review on two axes and simplify

Review changes against both **the intended behavior** (acceptance criteria, originating request or spec) and **the repository's standards and architecture**. Keep the reviews independent when more than one reviewer is warranted: one checks spec coverage and missed cases; another checks maintainability, conventions, and likely regressions. Return findings with evidence and locations, not vague preferences. Apply project conventions first and only make a refactor when it removes real complexity or improves a seam; do not broaden a focused change for aesthetic cleanup.

When an error repeats, address it at the highest durable level that fits the cause: architecture or contracts first, then types or static checks, then tests or CI, then documentation. Choose the smallest effective correction and verify it prevents recurrence. Capture durable learnings in an existing project document or the skill itself when they will guide future work; avoid adding a generic note for a one-off mistake.

## Verification evidence

Automated tests are the repeatable proof of specified behavior. Run relevant tests and static checks, report exact commands and results, and never infer a pass from a worker's summary alone.

Runtime checks complement tests. For user-visible or integration behavior, run the application or a representative end-to-end path when the environment permits. Runtime checks can reveal integration or configuration problems that unit tests miss, but they do not replace focused automated tests. Capture observable evidence when useful (for example, a screenshot, request/response, or log excerpt) and tie it to the acceptance criterion. If runtime verification is unavailable, state what was verified and what remains unverified.

Use an independent reviewer for meaningful or risky changes when a separate model or agent is available. Prefer a different model or context from implementation. Reviewers should inspect the diff and evidence against the plan and report actionable findings with locations. Reconcile disagreements against code, tests, or runtime evidence, fix confirmed findings, and rerun affected checks. The Opus final adversarial review (Algorithm steps 8-9) still applies; this paragraph governs per-node review only.

## Persistent supervision and learning

Keep a compact task ledger with task ID, status, dependencies, owner, changed files, verification evidence, and blockers. Update it as work lands so the user can ask what is ready for review at any point. For work that may cross sessions or needs handoff, persist the ledger and key decisions in a clearly named project-local note; otherwise keep it in the active working context and avoid adding process files to the product repository.

When a recurring failure or review finding reveals a systemic cause, fix the durable source of the problem where practical (a test, type, guard, documented contract, or CI check). Record the cause and prevention in the project's established documentation when future contributors need it; do not treat a one-off correction as a reason to add generic process documentation.

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
