# Codex Long-Horizon Task State

SKILL: long-horizon-resume
TASK_ID: <short stable id>
OVERALL_STATUS: ACTIVE
CONTINUATION_MODE: <automation|manual>
AUTOMATION: <name/id/cadence or None>
PROJECT_ROOT: <absolute or project-relative path>
LAST_CHECKPOINT: <timestamp>
CURRENT_STAGE: <stage id>
NEXT_ACTION: <one exact next action>

## Objective

<Preserve the user's objective without broadening it.>

## Invariants

- <required tool route>
- <scope constraint>
- <format/validation requirement>
- <other decisions that must survive continuation>

## Stages

| ID | Stage | Status | Evidence / Outputs | Next |
|---|---|---|---|---|
| S0 | <stage> | TODO | | |
| S1 | <stage> | TODO | | |

Allowed stage states: `TODO`, `IN_PROGRESS`, `DONE`, `FAILED`, `BLOCKED`.

## Files and artifacts

| Path | Purpose | Status | Validation |
|---|---|---|---|

## Running processes

| Job/PID | Command or job | Working dir | Log | Started | State | Safe resume action |
|---|---|---|---|---|---|---|

## Decisions and mappings

<Record notation mappings, design decisions, tool-version constraints, or other facts that must not be rediscovered or silently changed.>

## Validation record

| Stage | Check | Result | Evidence |
|---|---|---|---|

## Blockers / failures

<Record exact failure, relevant logs, and whether a retry is safe.>

## Resume instruction

Read this file first. Continue only from the first incomplete/interrupted stage. Preserve DONE validated work. Check for existing long-running jobs before starting duplicates. Update this file before and after each major work unit.

## Completion summary

<Fill only when OVERALL_STATUS becomes COMPLETE.>
