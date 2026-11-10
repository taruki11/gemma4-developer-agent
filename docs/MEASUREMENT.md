# Measurement protocol (summary)

The goal of the protocol is to keep a *negative* claim honest: a method only counts as a gain if
the same fixed task set, the same agent budget, and the same independent scoring produce a
higher task-level resolve count.

## Scoring

- Agent phase and scoring phase are separate. The agent only sees a sanitized task record
  (`instance_id`, `repo`, `base_commit`, `problem_statement`, `hints_text`) and a clean snapshot of
  the repository at that base commit. No test patch, no FAIL_TO_PASS list, no reference fix.
- After all agent runs finish, an independent process hands the recorded patches to the official
  `swegemma` verification path (clean baseline + reference "gold" patch + agent patch under the
  same test command) on CPU only.
- A task is **resolved** only when the patched repository passes the required test nodes; a
  non-empty patch, a compiling patch, or a passing local example is not a resolve.

## Controls

Every task set is validated before use:

- clean baseline must **fail** (otherwise the task cannot show a fix);
- the reference gold patch must **pass** (otherwise the scoring environment is broken).

Tasks that fail either control are excluded and reported, never silently dropped.

## Splits and reuse

- The official public development corpus (129 tasks) is the only source of tasks. The hidden
  leaderboard set is different and is never touched by these experiments.
- Tasks used for a prompt/method experiment are exposed forever after; later experiments reuse
  them only as *previously exposed* tasks and say so. Fresh-task claims use tasks that have never
  been used before in any run.
- The six tasks reserved for the first held-out calibration are never used for tuning.

## Comparisons

- One-factor-at-a-time changes only (prompt wording, parser schema, service parameter), identical
  model, tasks, budgets, tool set, and scoring between arms.
- Primary metric: task-level resolved under the fixed budget. Secondary: empty patches, timeouts,
  tool calls, wall time, and error-class counts.
- Because task-level counts are small, results are reported as paired gained/lost with the exact
  binomial / McNemar p-value when possible, and otherwise as descriptive with an explicit
  "insufficient evidence" label. A single success is never presented as causality; a floor of
  0/11 is never presented as proof that a method cannot help.
