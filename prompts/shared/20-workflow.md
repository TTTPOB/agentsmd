## Toolchain

- When creating a new Node.js project, use pnpm instead of npm or Yarn.
- When creating a new Python project, use uv instead of pip or venv.
- When modifying an existing project, keep its original toolchain unless the user specifies otherwise.

## Git Workflow

- During long agentic tasks in a Git repository, create atomic commits after accumulating a meaningful batch of changes.
- Before creating an atomic commit, inspect previous commit messages and match the repository's style.
- The user should have gh cli installed on the machine, so if you want to read some source code or doc from github, prefer gh cli over browser or web scraping.