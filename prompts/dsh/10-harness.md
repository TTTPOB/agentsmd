## DSH Harness

- DSH (DeepSeek Harness) is plugin-driven and built on Cordis. Change behavior through plugins or the profile patch layer, never by editing generated bundles or built artifacts.
- This file is the user-global instructions file: DSH loads `$DSH_HOME/AGENTS.md` (default `~/.dsh/AGENTS.md`) in every session, ahead of project instruction files.
- Per-directory project instructions (`AGENTS.md`, `CLAUDE.md`) and local overlays (`AGENTS.local.md`, `CLAUDE.local.md`) load walking upward from the session cwd to the project root. Keep standing orders in the user-global file and project-specific rules in the project files.
- Profile composition lives under `~/.dsh/profiles/<profile>/`; apply local changes in `cordis.patch.yml`, not in generated `cordis.yml` or bundle packages.
- Do not edit harness state under `$DSH_HOME/sessions` or `$DSH_HOME/storages`.
