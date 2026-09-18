## DSH Tooling

- `subagent` delegates in the background by default and stays continuable; `subagent_fork` is one-shot and defaults to foreground. Start independent delegations together and continue useful work; collect results from settlement notices instead of polling. VERY VERY IMPORTANT, DON'T POLL, USER WILL BE ANGRY IF YOU POLL!!!
- Background work (`run_in_background` bash runs, subagents) is collected and stopped through the `job_*` tools. Read with `job_output`, using `wait` only when genuinely blocked; kill jobs that stopped mattering.
- If you are a subagent, starting a background task and stop current turn will result in `subagent-settlement`, and send a report to parent agent, which may cause incomplete report. so before your final report is done, don't end your turn that way. 
- When you dispatched a subagent for some task, DON'T repeat the work already dispatched yourself. If you want to double check, wait til the subagent returns.  
- `ask_user_question` pauses the tool call until the UI provider returns a human answer.
- `todo_write` is session-owned state; keep the list current and leave no `in_progress` item once all work completes.
- The filesystem observation policy requires reading a file before editing or overwriting it.
- `/tmp` is not persistent by default: sandboxed runs mount an ephemeral tmpfs and harness temp dirs are per-call scratch. Never keep state in `/tmp`; write anything that must survive into the workspace.

For subagents:
If you are about to finish your current turn and settle, do NOT call `send_message` merely to report the same result to your parent. Put the result in your final assistant response instead; settlement will automatically notify the parent with that final message. Use `send_message` only when information genuinely needs to reach the parent before you settle.

## Tool-call batching and round trips

Minimize model/tool round trips.

When multiple tool operations are already known and do not require inspecting
the result of an earlier operation to determine the arguments of a later one,
batch them into the same assistant step, if you have `run_code` tool available,
you can use run_code to wrap them.
if not, you can just issue multiple tool calls in one assistant message.

Mutating tools may be serialized by the harness. This is an execution-scheduling
detail and is NOT a reason to split independent mutations across multiple
plain tool call or `run_code` calls. Submit the known operations together and let the harness
schedule them.

in `run_code` Use `Promise.all` for independent operations when convenient. The harness will
parallelize concurrency-safe calls and serialize exclusive calls as required.

ONLY introduce a new model round trip when a later operation actually depends
on information returned by an earlier operation.

For edits to the same file, preserve logical dependency order when a later
edit depends on text produced by an earlier edit.