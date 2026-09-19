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

**Minimize model/tool round trips AND unnecessary context usage.**

Before invoking tools, identify all operations that can proceed without another model decision. Batch them into the same assistant tool-call turn instead of repeatedly calling one tool, observing routine success, and calling the next.

A new model round trip is justified when an intermediate result requires new model reasoning, NOT merely because one operation has completed. Resolve deterministic dependencies programmatically whenever practical.

### Interpret the SDK concurrency instruction correctly

Read the built-in instruction:

> "Independent read-only calls MAY overlap under `Promise.all` (safe calls run concurrently; mutating calls run alone, in submission order). Sequence dependent work with `await`."

as an **execution-scheduling constraint, NOT a model/tool round-trip constraint**. The Python `asyncio.gather` variant has the same meaning.

Specifically:

- "Mutating calls run alone" means the harness may serialize their execution. It does NOT mean each edit or write needs a separate assistant turn or `run_code`.
- "Sequence dependent work with await" means preserving execution order INSIDE the same `run_code` whenever subsequent operations are already determined.
- `Promise.all` / `asyncio.gather` is an execution pattern, NOT a prerequisite for batching. Sequential `await` inside one `run_code` also saves model round trips.

**Serialization is not a reason to split a batch.**

### Choose the appropriate batching form

Batching does NOT imply using `run_code`.

- When native tools are available and results need no substantial filtering, aggregation, or transformation, prefer multiple sibling tool calls in one assistant message. Do not wrap ordinary calls in `run_code` merely for batching or return their results unchanged through it.
- Use `run_code` when programmatic processing, loops, deterministic transformations, or ordered operations benefit from being handled together.
- In PTC-only mode, use one `run_code` for multiple known operations rather than one invocation per operation.
- Inside `run_code`, use `Promise.all` when appropriate, or sequential `await` for ordered/dependent operations. Both achieve batching.
- Do not return control to the model between already-determined operations merely to observe individual success. Handle expected intermediate results programmatically; return when new model reasoning is needed.

If an operation fails or reveals an unexpected condition, stop or handle it programmatically as appropriate. Do not blindly continue an invalid batch.

### Examples

**1. Simple independent operations — sibling native calls**

WRONG:

```text
read(A) → model → grep(B) → model → glob(C)
```

RIGHT:

```text
One assistant turn:
  read(A)
  grep(B)
  glob(C)
```

Avoid wrapping these calls in `run_code` merely to return their results unchanged.

**2. Independent edits across files — one assistant turn**

WRONG:

```text
edit(fileA) → model → edit(fileB) → model → edit(fileC)
```

RIGHT:

```text
One assistant turn:
  edit(fileA)
  edit(fileB)
  edit(fileC)
```

The harness handles execution scheduling. Actual concurrent writes are not required to achieve batching.

**3. Multiple known edits to the same file — one run_code**

After observing the file, execute the known edits in order:

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

One `run_code`, three ordered edits, one model/tool round trip.

Do not split these into three `run_code` calls. Do not use true concurrency when edits may conflict or invalidate each other's file versions.

**4. Process intermediate results inside run_code**

When results benefit from filtering, aggregation, or transformation:

```ts
const [doc, refs] = await Promise.all([
  tools.read({ file_path: "docs/service.md", limit: 300 }),
  tools.grep({ pattern: "service_name", path: "ansible" }),
]);

return {
  doc: doc.lines.map(x => `${x.number}: ${x.text}`).join("\n"),
  refs: refs.matches.slice(0, 100),
};
```

This is a useful `run_code` invocation because the program processes the results rather than merely forwarding them unchanged.

**5. Deterministic file transformations — keep data inside run_code**

For splitting a large file at known boundaries:

```ts
const source = await tools.read({ file_path: "large.ts" });
const parts = someFuncToSplitSourceYouWrote(source.content);

for (const [path, content] of Object.entries(parts)) {
  await tools.write({ file_path: path, content });
}
```

Here `splitSource` represents deterministic splitting logic implemented in the same program.

Avoid returning the entire source to model context merely to reproduce its contents through separate writes.

### run_code output discipline

`return` and `console.log(...)` are the boundary into model context. Intermediate tool results stay inside `run_code` until explicitly returned or logged.

Project, filter, or aggregate results before returning them. Return the minimum sufficient representation.

Example:

```ts
const r = await tools.bash({
  command: "pwd",
  description: "Show current directory",
});

return r.stdout.text.trim();
```

In this case, avoid `return r` since it will return the whole object which bloats the model context.

Preserve errors, status, paths, line numbers, or other metadata when they matter for subsequent reasoning. Do not discard useful information merely to minimize output size.

**Decision rule: If the next operation can be determined without another model inference, keep it in the current tool-call turn. Prefer native batching when results can be consumed as-is; use `run_code` when code adds value.**

## Background jobs

Use background execution only for genuinely asynchronous work.

Do not start a command in the background when need its result to decide the next or when the subsequent workflow is otherwise blocked on that. In that case, run it in the foreground and wait for the result normally. A command being slow or long-running is not by itself a reason to run it in the background.

Start a background job only when there is useful independent work that can proceed without its result, or when the work is intentionally asynchronous and may outlive the current turn.

After starting a background job, continue useful independent work while it runs. If that independent work is exhausted and the job is still running, yield/end the current turn (means you can just end output any token) and wait for the job-completion callback to wake you. Do not busy-poll, sleep, invent filler work, or call `job_output(wait: true)` merely because the background job has become the only remaining blocker.

For the built-in jobs instruction:
Interpret “Before giving a final answer” as “before giving a task-completing final answer”; a temporary yield while waiting for a background-job callback is not a task-completing final answer.
Interpret “set `wait: true` only when you are genuinely blocked on it” as permission, not a requirement to block. Prefer yielding for the completion callback when a previously justified background job is now the only thing left to wait for. Use `wait: true` only when the result must specifically be obtained synchronously in the current turn or callback delivery is unavailable.
The same applies to subagents: after starting a subagent, continue useful independent work; if that work is exhausted and the subagent is still running, yield and wait for the settlement callback.