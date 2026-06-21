# Quickstart

The fastest path is to let the `@onboarding` agent do it. Prefer the terminal?
The CLI path is right below. Both produce the same build. Designed for the
**Pro Dev with AI** persona — see [PERSONAS.md](PERSONAS.md).

## Fastest path — let the agent set it up

Clone the repo, open it in GitHub Copilot Chat or Claude Code, and paste:

```text
Set up agent-homebase for my project. Ask me what you need, recommend a
profile, run init.py, deploy the resolved files, and verify it works.
```

The agent interviews you (~5 questions), picks a profile, fills the config,
runs the build, deploys the files, and verifies — no YAML editing, no copy
commands. It self-removes once setup is confirmed.

## CLI path

Designed for the **Pro Dev with AI** persona — see [PERSONAS.md](PERSONAS.md).
Get a working build into your project in three commands.

### Prerequisites

- Python 3.12+
- A virtual environment tool: `uv` is recommended on Linux/WSL; `venv` also works
- Git

### Steps

1. **Clone and enter the repo:**
   ```bash
   git clone https://github.com/j78f88/agent-homebase.git
   cd agent-homebase
   ```

2. **Create an environment and install dependencies:**

   Linux/WSL with `uv`:
   ```bash
   uv venv --python python3.12 .venv
   source .venv/bin/activate
   uv pip install -r requirements-dev.txt
   ```

   Standard Python:
   ```bash
   python -m venv .venv
   source .venv/bin/activate   # Windows PowerShell: .venv\Scripts\Activate.ps1
   python -m pip install -r requirements-dev.txt
   ```

3. **Pick a profile** that matches your stack:
   ```bash
   cp profiles/python-api.config.yml config/my-project.config.yml
   # Other options: react-web-app, monorepo-fullstack
   ```

   Windows PowerShell:
   ```powershell
   Copy-Item profiles\python-api.config.yml config\my-project.config.yml
   ```

4. **Fill in `config/my-project.config.yml`:** repo name, project language,
   framework, coverage targets, commands, and deployment URLs. Comments mark
   values that should be project-specific.

5. **Build** the resolved artifacts:
   ```bash
   python init.py --config config/my-project.config.yml
   ```

6. **Verify the build:**
   ```bash
   python -m pytest tests/ -q
   ```

## What you got

- `resolved/skills/` — copy into your project's `.github/agents/`
- `resolved/instructions/` — copy into `.github/instructions/`
- `resolved/agents/` — copy into `.github/agents/`

For a first project bootstrap, copy the starter docs you need from `starters/`
into the target project.

## Known build output

The example config may print warnings for optional external security commands
such as `syft`, `trivy`, and `checkov` because they are not in the default
command whitelist. That is expected for the sample config; a `✓ Security
validation passed` line means the build continued successfully.

## Next

Re-run `python init.py` any time you change the config or the source under
`skills/`, `instructions/`, or `agents/`. Never hand-edit anything in
`resolved/` — it is regenerated every build.
