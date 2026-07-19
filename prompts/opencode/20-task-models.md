## OpenCode Model Mapping

When delegating work with the `task` tool, map the shared routing policy as follows:

- Terra high: use `openai/gpt-5.6-terra` with variant `high`.
- Sol medium: use `openai/gpt-5.6-sol` with variant `medium`.
- Sol maximum: use `openai/gpt-5.6-sol` with variant `max`.
- Do not call `task_models` for these known mappings. Call it when the user requests another provider, when you need to discover variants, or when a known mapping fails validation.
