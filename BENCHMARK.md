# Workflow benchmark protocol

This repository's cost and speed claims are design expectations until measured. Use this protocol before presenting them as results or deciding whether to replace this workflow with another stack such as pstack.

## Compare fairly

1. Pick representative tasks before running either workflow: one multi-file feature, one architecture-heavy change, and one debugging or refactoring task. Keep prompts, repositories, starting commits, acceptance criteria, and available tools identical.
2. Pin and record each workflow version and model configuration. If exact model parity is impossible, record the difference and avoid attributing all outcome differences to the workflow.
3. Run each task from a clean checkout. Do not transfer implementation notes or outputs between runs. Repeat runs when practical; report the number of runs and spread, not just the best result.
4. Have reviewers who do not know which workflow produced each diff evaluate the result against the same acceptance criteria.

## Record per run

- Task identifier, task type, repository and starting commit
- Workflow name and revision
- Model names/versions, effort settings, concurrency, and relevant tool availability
- Elapsed wall-clock time and human intervention time
- Token usage and estimated cost, with pricing source/date and any excluded usage
- Acceptance criteria met and checks run, including exact test/lint/build commands
- Runtime evidence for user-visible or integration behavior, where available
- Independent review findings: confirmed defects, severity, and whether each was fixed
- Final outcome, unresolved risks, and anything that made the run incomparable

## Interpret results

Report quality, cost, and time separately. A faster run that misses requirements is not a win; a cheaper run that needs substantial human repair should include that repair effort. Do not compare token counts across different tokenizers as if they were directly equivalent. Explain missing telemetry, failed runs, model/configuration changes, and sample-size limits. Do not infer a general advantage from a single task or publish expected savings as measured results.
