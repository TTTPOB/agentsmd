# Agent prompt fragments

Platform support: Linux only.

This repository keeps shared and tool-specific agent instructions as numbered Markdown fragments. The renderer combines tracked fragments with device-specific fragments in the ignored `local/` directory.

## Install

Clone the private repository and run the installer:

```bash
git clone git@github.com:TTTPOB/agentsmd.git
cd agentsmd
./scripts/install
```

The installer detects the current Bash or Zsh shell, creates the ignored device directories, installs the `prompts-render` and `prompts-update` aliases, and renders all templates. Select a shell explicitly or configure both:

```bash
./scripts/install bash
./scripts/install zsh
./scripts/install all
```

Restart the shell or source the updated `~/.bashrc` or `~/.zshrc` after installation. The installer maintains one marked configuration block, so repeated runs update the existing aliases.

## Layout

```text
prompts/shared/*.md       Shared by Codex and OpenCode
prompts/codex/*.md        Codex-specific instructions
prompts/opencode/*.md     OpenCode-specific instructions
templates/*.template      Composition and output definitions
scripts/render-prompts    Builds both global instruction files
local/                    Ignored device-specific fragments
```

The renderer processes templates in filename order. When a template includes a directory, it loads that directory's Markdown fragments in filename order. Use numeric prefixes such as `10-`, `20-`, and `30-` to make fragment order explicit.

Device-specific fragments live under `$AGENT_PROMPTS_HOME/local`:

```text
local/
├── shared.d/             Loaded by Codex and OpenCode
├── codex.d/              Loaded only by Codex
└── opencode.d/           Loaded only by OpenCode
```

The script uses its repository root when `AGENT_PROMPTS_HOME` is unset. Git ignores the complete `local/` directory.

## Templates

Each template declares one output file and the content to render:

```md
---
output: $HOME/.codex/AGENTS.md
---
@prompts/shared
@prompts/codex
@?local/shared.d
@?local/codex.d
```

Template directives have the following behavior:

- `output:` sets the rendered file path.
- `@file.md` includes one file.
- `@directory` includes non-empty `*.md` files from one directory in filename order.
- `@?path` marks a file or directory as optional.
- Other lines are copied to the output as literal text.

Relative include and output paths resolve from `AGENT_PROMPTS_HOME`. Output paths support `$HOME`, `${HOME}`, `$XDG_CONFIG_HOME`, `${XDG_CONFIG_HOME}`, and `~/`. When `XDG_CONFIG_HOME` is unset, the renderer uses `$HOME/.config`. It rejects other variables instead of evaluating shell expressions.

Add another `templates/*.template` file to create another output. The renderer discovers it without script changes.

## Render prompts

```bash
./scripts/render-prompts
```

The included templates write:

```text
~/.codex/AGENTS.md
~/.config/opencode/AGENTS.md
```

Check whether existing outputs match the fragments without changing them:

```bash
./scripts/render-prompts --check
```

Restart OpenCode after rendering because a running session keeps the instructions it loaded at startup.

## Bash aliases

Add the following lines to `~/.bashrc`, replacing the path with this repository's location:

```bash
export AGENT_PROMPTS_HOME="$HOME/path/to/agentsmd"
alias prompts-render='"$AGENT_PROMPTS_HOME/scripts/render-prompts"'
alias prompts-update='git -C "$AGENT_PROMPTS_HOME" pull --ff-only && "$AGENT_PROMPTS_HOME/scripts/render-prompts"'
```

Reload the shell configuration:

```bash
source ~/.bashrc
```

## Zsh aliases

Add the same definitions to `~/.zshrc`:

```zsh
export AGENT_PROMPTS_HOME="$HOME/path/to/agentsmd"
alias prompts-render='"$AGENT_PROMPTS_HOME/scripts/render-prompts"'
alias prompts-update='git -C "$AGENT_PROMPTS_HOME" pull --ff-only && "$AGENT_PROMPTS_HOME/scripts/render-prompts"'
```

Reload the shell configuration:

```zsh
source ~/.zshrc
```

`prompts-update` requires this directory to be a Git repository with an upstream branch. `prompts-render` works without Git.

## Device-specific prompts

Copy the example once, then edit it for the current machine:

```bash
mkdir -p "$AGENT_PROMPTS_HOME/local/shared.d"
cp "$AGENT_PROMPTS_HOME/examples/device.md" \
  "$AGENT_PROMPTS_HOME/local/shared.d/10-device.md"
```

Keep machine paths, hardware constraints, local model availability, and locally installed tooling in these directories. Do not commit them to this repository.
