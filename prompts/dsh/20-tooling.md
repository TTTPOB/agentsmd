## DSH Tooling

- `subagent` delegates in the background by default and stays continuable; `subagent_fork` is one-shot and defaults to foreground. Start independent delegations together and continue useful work; collect results from settlement notices instead of polling.
- Background work (`run_in_background` bash runs, subagents) is collected and stopped through the `job_*` tools. Read with `job_output`, using `wait` only when genuinely blocked; kill jobs that stopped mattering.
- `ask_user_question` pauses the tool call until the UI provider returns a human answer.
- `exit_plan_mode` is only valid while plan mode is active.
- `todo_write` is session-owned state; keep the list current and leave no `in_progress` item once all work completes.
- The filesystem observation policy requires reading a file before editing or overwriting it.
- `read_image` executes only when the exact routed model declares image input.
- `/tmp` is not persistent by default: sandboxed runs mount an ephemeral tmpfs and harness temp dirs are per-call scratch. Never keep state in `/tmp`; write anything that must survive into the workspace.
