## OpenCode Tooling

- When a PTY tool call uses `notifyOnExit`, wait for its exit notification instead of checking it frequently.
- Only invoke the `grillme` skill when the user explicitly asks to use it. Do not invoke it to clarify incomplete tasks or gather requirements.
- When confusing API behavior or a non-obvious design tradeoff requires workaround code, add a comment explaining the reason.
