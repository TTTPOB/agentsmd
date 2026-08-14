## OpenCode Model Mapping

When delegating work with the `task` tool, map the shared routing policy tiers as follows:

- cheap: use `openai/gpt-5.6-luna` with variant `xhigh`.
- mid: use `openai/gpt-5.6-sol` with variant `medium`.
- expensive: use `openai/gpt-5.6-sol` with variant `xhigh`.
- Do not call `task_models` for these known mappings. Call it when the user requests another provider, when you need to discover variants, or when a known mapping fails validation.
