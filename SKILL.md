---
name: long-horizon-resume
description: Use only when explicitly invoked as $long-horizon-resume for a user-requested long-running Codex task that may cross usage/reset windows and must resume from durable checkpoints instead of restarting. Do not use implicitly for ordinary long tasks.
---

# Long Horizon Resume

Run a user-authorized long task as a resumable, idempotent workflow that can survive Codex usage-window exhaustion, app restarts, and delayed continuation runs.

## Activation boundary

This skill is explicit-only. Do not apply it merely because a task is large, slow, expensive, or likely to hit a usage limit. Apply it only when the user explicitly invokes `$long-horizon-resume` (or explicitly asks to use this installed skill).

The skill does not bypass, extend, or alter any quota. Its purpose is to preserve state and resume safely after execution becomes available again.

## Core invariants

1. Persist task state on disk before substantial work begins.
2. Treat the persisted state file, not conversational memory, as the authoritative recovery record.
3. Split work into bounded, independently resumable stages and work units.
4. Make every continuation idempotent: completed and validated work is reused, not regenerated.
5. Checkpoint before and after each major stage, after important decisions, and before starting an external long-running process.
6. Never depend on a final "save state" action after a quota error; assume interruption can occur without warning.
7. Preserve the user's original scope, constraints, tool route, and validation requirements across continuation runs.
8. Do not create duplicate background jobs or overlapping continuation workers.

## Start-up procedure

When invoked for a task:

1. Identify the project/workspace root and the user's final objective.
2. Read any existing `codex_task_state.md` in that root.
   - If it belongs to the same objective and is not COMPLETE, resume it.
   - If it is COMPLETE, do no continuation work unless the user explicitly starts a new task.
   - If it belongs to another task, do not overwrite it. Create a task-specific state file such as `codex_task_state_<short-id>.md` and record that path.
3. If no state exists, create one using `references/state-template.md`.
4. Decompose the task into the smallest sensible stages that have observable completion criteria. Prefer stages that can normally be completed and checkpointed within roughly 20-30 minutes of agent work. Long external computations may exceed this if their process/log state is persisted.
5. Record all non-negotiable constraints and tool choices under `INVARIANTS` before executing them.
6. Mark the first actionable stage `IN_PROGRESS`, write the exact `NEXT_ACTION`, save the state file, then begin work.

## Continuation automation

If the current Codex surface exposes a thread/task automation or scheduled continuation facility, create at most one continuation automation for this task.

The automation exists only to re-enter this same workflow. It must not redefine the task.

Preferred continuation instruction:

> Resume the task governed by the persisted Codex task-state file in this project. Read that file first. If OVERALL_STATUS is COMPLETE, do nothing. Otherwise continue only from the first incomplete or interrupted stage, preserve completed validated work, avoid duplicate long-running processes, validate outputs before marking stages DONE, update the state file, and stop when the current work unit is safely checkpointed.

Scheduling policy:

- If a reliable usage-window reset time is known from the product, schedule the next attempt shortly after that time.
- Otherwise, use a conservative recurring cadence no faster than once per hour.
- Do not create multiple overlapping continuation automations for the same task.
- Record the automation identifier/name and cadence in the state file when available.
- When the task reaches COMPLETE, disable/cancel the continuation automation if the surface supports doing so. If cancellation is unavailable, the automation must immediately exit whenever it sees COMPLETE.

If no automation facility is available, set `CONTINUATION_MODE: manual` and write the exact resume command/prompt into the state file. Do not claim that automatic cross-window continuation is available.

## Work-unit protocol

For each stage or work unit:

1. Read the state file first.
2. Re-establish only the context needed for the next action; do not reread large unchanged files without reason.
3. Verify existing outputs before regenerating them.
4. Before modifying an important artifact, record what is about to change and the expected validation.
5. Execute the smallest complete unit of work.
6. Validate the result using observable evidence appropriate to the task (tests, rendering, file inspection, numerical checks, etc.). A successful command exit code alone is insufficient when the artifact itself can be wrong.
7. Update:
   - stage status;
   - files created/modified;
   - validation evidence;
   - decisions/invariants learned;
   - exact next action.
8. Save the state file before moving on.

A continuation run must never restart from the original prompt unless the persisted state is unusable or the user explicitly requests a restart.

## Handling external long-running processes

Before launching a computation, conversion, build, or other process that can outlive the current model turn, persist:

- command or job description;
- working directory;
- PID/job ID if available;
- stdout/stderr or log path;
- expected output paths;
- start time;
- how to determine whether it is still running;
- how to validate success;
- whether restarting is safe.

On continuation:

1. Check whether the existing job is still alive or has produced final outputs.
2. If alive, do not launch a duplicate. Poll infrequently and checkpoint the observation.
3. If finished, validate outputs before marking the stage DONE.
4. If dead/failed, inspect logs and decide whether restart is safe. Record the failure and restart rationale.

## Interruption and quota exhaustion

Assume quota exhaustion can terminate execution immediately.

Therefore:

- never leave the only copy of critical state in chat context;
- do not hold multiple essential steps only in memory before checkpointing;
- prefer atomic/small edits over a large uncheckpointed rewrite;
- checkpoint immediately after expensive or hard-to-reproduce work;
- do not delete valid intermediate outputs merely because the final stage is unfinished.

If execution becomes unavailable, leave the project in a recoverable state. The next automation/manual invocation must read the state file and continue from `NEXT_ACTION`.

## Idempotence and conflict rules

On every resume:

- `DONE` + validated: reuse it.
- `IN_PROGRESS` + expected output exists: inspect/validate before rerunning.
- `IN_PROGRESS` + external job alive: wait; do not duplicate.
- `FAILED`: inspect recorded evidence before retrying.
- project changed externally: identify the conflict before continuing; do not silently overwrite newer user changes.

Do not rebuild a validated artifact just to make the continuation look clean.

## Efficiency rules

- Avoid unnecessary subagents; use them only when parallel work materially reduces total work and their results can be checkpointed independently.
- Reuse prior extraction, parsed data, intermediate renders, and verified decisions.
- Batch related operations when this does not increase recovery risk.
- Keep status polling infrequent.
- Avoid unrelated web/literature research unless the task itself requires it.
- Prefer deterministic scripts or existing project tooling for repetitive operations.

## Completion criteria

Mark `OVERALL_STATUS: COMPLETE` only when:

1. all required stages are DONE;
2. required validation has passed;
3. requested final artifacts/results exist at recorded paths;
4. unresolved blockers are empty or explicitly accepted by the user;
5. `NEXT_ACTION` is `None`;
6. the final state file contains a concise completion summary.

Then cancel/disable the continuation automation when possible. Never continue modifying the task after COMPLETE unless the user explicitly reopens it.
