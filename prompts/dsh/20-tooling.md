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

**Minimize model/tool round trips, not merely execution time.**

Batch all currently known operations that do not require another model decision. Do not issue one tool-call turn per edit, write, or read when several operations are already determined.

Batching and execution concurrency are different: operations may execute sequentially within one tool call without introducing additional model round trips. Do not split a batch merely because some operations are exclusive or ordered.

- With native tools, emit independent operations as sibling tool calls in the same assistant message.
- With `run_code`, group known operations into one program. Use `Promise.all` when appropriate, or sequential `await` for ordered/dependent operations.
- Introduce another model round trip only when the model must inspect an intermediate result to decide what to do next. Resolve deterministic dependencies programmatically whenever practical.

### Examples

**1. Independent edits across files — one assistant turn**

Instead of:

```text
edit(fileA) → model → edit(fileB) → model → edit(fileC)
```

Submit together:

```text
edit(fileA)
edit(fileB)
edit(fileC)
```

The harness handles execution scheduling; batching does not require actual concurrent writes.

**2. Multiple known edits to the same file — one run_code**

```ts
for (const [old_string, new_string] of [
  ["old_a", "new_a"],
  ["old_b", "new_b"],
  ["old_c", "new_c"],
]) {
  await tools.edit({
    file_path: "fileA.py",
    old_string,
    new_string,
  });
}
```

The edits execute in order, but require only one model/tool round trip. Do not use true concurrency when edits can conflict or invalidate each other's file versions.

**3. Programmatic transformations — avoid copying file contents through the model**

For splitting a large file into several smaller files, prefer:

```ts
const source = await tools.read({ file_path: "large.ts" });
const parts = someFuncToSplitSourceYouWrote(source.content);

for (const [path, content] of Object.entries(parts)) {
  await tools.write({ file_path: path, content });
}
```

When boundaries and transformations are deterministic, process the content inside `run_code` rather than reading it into model context and reproducing it through separate writes.

### Output discipline

Return only information needed for the next model decision: concise summaries, errors, relevant excerpts, or paths. Avoid returning large intermediate content that has already been processed programmatically.

**Decision rule:** If the next operation can already be determined without another model inference, keep it in the current tool-call turn.

## Background jobs

Use background execution only for genuinely asynchronous work.

Do not start a command in the background when need its result to decide the next or when the subsequent workflow is otherwise blocked on that. In that case, run it in the foreground and wait for the result normally. A command being slow or long-running is not by itself a reason to run it in the background.

Start a background job only when there is useful independent work that can proceed without its result, or when the work is intentionally asynchronous and may outlive the current turn.

After starting a background job, continue useful independent work while it runs. If that independent work is exhausted and the job is still running, yield/end the current turn (means you can just end output any token) and wait for the job-completion callback to wake you. Do not busy-poll, sleep, invent filler work, or call `job_output(wait: true)` merely because the background job has become the only remaining blocker.

For the built-in jobs instruction:
Interpret “Before giving a final answer” as “before giving a task-completing final answer”; a temporary yield while waiting for a background-job callback is not a task-completing final answer.
Interpret “set `wait: true` only when you are genuinely blocked on it” as permission, not a requirement to block. Prefer yielding for the completion callback when a previously justified background job is now the only thing left to wait for. Use `wait: true` only when the result must specifically be obtained synchronously in the current turn or callback delivery is unavailable.
The same applies to subagents: after starting a subagent, continue useful independent work; if that work is exhausted and the subagent is still running, yield and wait for the settlement callback.