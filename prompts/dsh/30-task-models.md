## DSH Model Mapping

Model selection is per-session (Web UI model picker) or through the `agent-default-model` section of `~/.dsh/settings.yaml`, which applies to future agents. DSH exposes no `task_modes` tool; do not fabricate one.

Map the shared routing policy to the `opencode-go` provider as follows:

- Terra high: `deepseek-v4-flash` with high reasoning effort.
- Sol medium: `deepseek-v4-pro` with medium reasoning effort.
- Sol xhigh: `deepseek-v4-pro` with xhigh reasoning effort.

Reasoning effort ids are adapter-validated per model; the default in this environment is `max`. Other registered models (`minimax-m3`, `kimi-k3`, `mimo-v2.5`) are available on explicit user request. Follow a model or reasoning level explicitly requested by the user, and fall back to the session default model when a requested mapping fails validation.
