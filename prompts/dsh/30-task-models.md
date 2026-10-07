## DSH Model Mapping

Model selection is per-session (Web UI model picker) or through the `agent-default-model` section of `~/.dsh/settings.yaml`, which applies to future agents. DSH exposes no `task_modes` tool; do not fabricate one.

Map the shared routing policy tiers to the `klaude-openai` provider as follows:
currently gpt-6.1-sol is the best model, use low for cheap task, medium for mid task, and high for more complex task

Reasoning effort ids are adapter-validated per model; the `klaude-openai` provider accepts `off`, `low`, `medium`, `high`, `xhigh`, and `max`.