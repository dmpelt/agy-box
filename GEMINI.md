# Container Runtime & Tooling Rules

## Execution Environment
- You are running as `root` inside an isolated, headless Ubuntu Linux container (`agy-sandbox`).
- **System vs. Workspace Boundary:**
  - **Container OS (`/` and `/root`):** Isolated from the host and persistent across sessions for this project. You have full `root` permissions to install system packages, CLI tools, and runtimes without asking.
  - **Project Workspace (`/workspace`):** Bind-mounted directly to the user's host filesystem. File edits, deletions, and Git commits in `/workspace` immediately affect the host. Never run destructive Git commands (`git push --force`, `git reset --hard`) without explicit user confirmation.

## Tool & Package Hierarchy
When solving a problem or running a task, strictly follow this priority:
1. **System Utilities (`apt-get`):** For standalone Linux CLI utilities (`jq`, `ripgrep`, `fd-find`, `librsvg2-bin`, `poppler-utils`, `graphviz`, `ffmpeg`), always prefer distro packages first. Check if the tool is installed (`command -v <tool>`); if missing, install non-interactively:
   `DEBIAN_FRONTEND=noninteractive apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends <pkg>`
2. **Project Dependencies & Environments (`pixi`):** Always prefer `pixi` over raw `pip`, `venv`, `conda`, or `npm` when managing project environments, language runtimes (Python, Node.js, Rust, C/C++), and libraries in `/workspace`:
   - **Keep Environments Outside `/workspace`:** Ensure Pixi is configured to store environments inside the container filesystem so `/workspace` and host checkpoints stay clean:
     `pixi config set --global detached-environments /root/.pixi/envs`
   - **Adding Packages:** Prefer conda-forge packages first (`pixi add <pkg>`), falling back to PyPI when needed (`pixi add --pypi <pkg>`).
   - **Running Commands:** Always execute project scripts, tests, and tools inside the Pixi environment using `pixi run <command>`.
   - **One-Off Global Tools:** If you need a standalone CLI tool from conda-forge that is not a dependency of the user's project, install it globally into `/root/.pixi` via `pixi global install <pkg>`.
3. **Custom Scripts (Last Resort):** Only write bespoke one-off scripts in `/tmp/` if no standard CLI utility or library already exists for the job.

## Output Verification & Quality Loops
Always verify your deliverables before concluding a task; never assume generated code or assets work without empirical testing:
- **Visual Assets & Diagrams (SVG, HTML, PDF, Charts):** Render visual outputs to PNG in `/tmp/` (e.g., `rsvg-convert -o /tmp/preview.png input.svg` via `librsvg2-bin`, or `pdftoppm -png -r 150 input.pdf /tmp/page` via `poppler-utils`) and inspect the rendered output to verify layout, font rendering, clipping, and text legibility.
- **Code & Data Deliverables:** Always run generated code or tests (`pixi run pytest`, etc.) to verify exit codes and data accuracy before reporting success.

## Host Portability & Documentation
- **Manifests Stay in `/workspace`:** Keep `pixi.toml` (or `pyproject.toml`) and `pixi.lock` committed in `/workspace` so the user can reproduce the exact environment on the host with `pixi install`.
- **System Packages:** Whenever the project itself requires OS-level `apt-get` packages or system libraries to build or run, explicitly document the exact package names and install commands in your response (or the project's `README.md`) so the setup can be reproduced on the host system.

## Shell & Filesystem Hygiene
- **Non-Interactive Execution:** Never trigger interactive prompts, watch modes, or pagers (`less`). Always use `PAGER=cat`, `GIT_PAGER=cat`, and `CI=true`, and pass non-interactive flags (`-y`, `--yes`, `--no-watch`).
- **Scratch Files:** Store temporary renders, downloads, and test logs in `/tmp/`, leaving `/workspace` clean for project deliverables.
- **Headless Only:** Run all browser automation headless (`--headless --no-sandbox --disable-gpu`).
- **Clean Daemons:** Kill background dev servers when testing finishes (`fuser -k <port>/tcp || true`).
