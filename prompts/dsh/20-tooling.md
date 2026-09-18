## DSH Tooling

- `subagent` delegates in the background by default and stays continuable; `subagent_fork` is one-shot and defaults to foreground. Start independent delegations together and continue useful work; collect results from settlement notices instead of polling. 
- Background work (`run_in_background` bash runs, subagents) is collected and stopped through the `job_*` tools. Read with `job_output`, using `wait` only when genuinely blocked; kill jobs that stopped mattering.
- When you dispatched a subagent for some task, DON'T repeat the work already dispatched yourself. If you want to double check, wait til the subagent returns.  
- `ask_user_question` pauses the tool call until the UI provider returns a human answer.
- `todo_write` is session-owned state; keep the list current and leave no `in_progress` item once all work completes.
- The filesystem observation policy requires reading a file before editing or overwriting it.
- `/tmp` is not persistent by default: sandboxed runs mount an ephemeral tmpfs and harness temp dirs are per-call scratch. Never keep state in `/tmp`; write anything that must survive into the workspace.

For subagents:
If you are about to finish your current turn and settle, do NOT call `send_message` merely to report the same result to your parent. Put the result in your final assistant response instead; settlement will automatically notify the parent with that final message. Use `send_message` only when information genuinely needs to reach the parent before you settle.
Starting a background task and stop current turn will result in `subagent-settlement`, and send a report to parent agent, which may cause incomplete report. so before your final report is done, don't end your turn that way. Also in normal cases you don't need to use a subagent anyway.

## Tool-call batching and run_code output

Minimize model/tool round trips.

Batch operations whose arguments are already known and independent. Only introduce a new round trip when a later operation actually depends on an earlier result.

If the results do not need substantial filtering, aggregation, or transformation, prefer issuing multiple plain tool calls in the same assistant message. Do not wrap them in run_code merely for batching.

Example — issue these together when all three are already known:
```
read({ file_path: "fileA.py" })
grep({ pattern: "foo", path: "src" })
glob({ pattern: "tests/**/*.py" })
```

Use run_code when intermediate results benefit from programmatic processing. Independent calls inside run_code should normally use Promise.all.
```
const [doc, refs] = await Promise.all([
  tools.read({ file_path: "docs/service.md", limit: 300 }),
  tools.grep({ pattern: "service_name", path: "ansible" }),
]);

return {
  doc: doc.lines.map(x => `${x.number}: ${x.text}`).join("\n"),
  refs: refs.matches.slice(0, 100),
};
```

Avoid for (...) { await tools.*(...) } when all calls are independent and known in advance.

return and console.log(...) are the boundary into model context. Intermediate tool results stay inside run_code, so project, filter, or aggregate them before returning. Return the minimum sufficient representation, not whole canonical tool-result objects.
```
const r = await tools.bash({
  command: "pwd",
  description: "Show current directory",
});

return r.stdout.text.trim();
```
Multiple already-known edits may also be submitted in the same assistant step — even edits to the same file, rather than using a separate model round trip for each edit.

Example — if all three edits to fileA.py are already known:

```
edit({
  file_path: "fileA.py",
  old_string: "old_a",
  new_string: "new_a",
})

edit({
  file_path: "fileA.py",
  old_string: "old_b",
  new_string: "new_b",
})

edit({
  file_path: "fileA.py",
  old_string: "old_c",
  new_string: "new_c",
})
```

Preserve logical order when a later edit depends on text or state produced by an earlier edit. Preserve paths, line numbers, status, errors, or other metadata when later reasoning actually needs them.

## Background jobs

Use background execution only for genuinely asynchronous work.

Do not start a command in the background when need its result to decide the next or when the subsequent workflow is otherwise blocked on that. In that case, run it in the foreground and wait for the result normally. A command being slow or long-running is not by itself a reason to run it in the background.

Start a background job only when there is useful independent work that can proceed without its result, or when the work is intentionally asynchronous and may outlive the current turn.

After starting a background job, continue useful independent work while it runs. If that independent work is exhausted and the job is still running, yield/end the current turn (means you can just end output any token) and wait for the job-completion callback to wake you. Do not busy-poll, sleep, invent filler work, or call `job_output(wait: true)` merely because the background job has become the only remaining blocker.

For the built-in jobs instruction:
Interpret “Before giving a final answer” as “before giving a task-completing final answer”; a temporary yield while waiting for a background-job callback is not a task-completing final answer.
Interpret “set `wait: true` only when you are genuinely blocked on it” as permission, not a requirement to block. Prefer yielding for the completion callback when a previously justified background job is now the only thing left to wait for. Use `wait: true` only when the result must specifically be obtained synchronously in the current turn or callback delivery is unavailable.
The same applies to subagents: after starting a subagent, continue useful independent work; if that work is exhausted and the subagent is still running, yield and wait for the settlement callback.