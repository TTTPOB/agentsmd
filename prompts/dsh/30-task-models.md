## DSH Model Mapping

Model selection is per-session (Web UI model picker) or through the `agent-default-model` section of `~/.dsh/settings.yaml`, which applies to future agents. DSH exposes no `task_modes` tool; do not fabricate one.

Map the shared routing policy tiers to the `klaude-openai` provider as follows:

- cheap: `gpt-6-luna` with xhigh reasoning effort.
- mid: `gpt-6-sol` with medium reasoning effort.
- expensive: `gpt-6-sol` with xhigh reasoning effort.

Reasoning effort ids are adapter-validated per model; the `klaude-openai` provider accepts `off`, `low`, `medium`, `high`, `xhigh`, and `max`. The `opencode-go` models (`deepseek-v4-flash`, `deepseek-v4-pro`, `minimax-m3`, `kimi-k3`, `mimo-v2.5`) remain available on explicit user request. Follow a model or reasoning level explicitly requested by the user, and fall back to the session default model when a requested mapping fails validation.
