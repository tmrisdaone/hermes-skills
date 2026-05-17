# Hermes Companion Tool Discovery Mechanism

This reference documents the agent-finding logic used by Hermes companion tool bootstrap scripts (e.g., Hermes WebUI's `bootstrap.py`). When a companion tool fails with "Python environment cannot import both WebUI dependencies and Hermes Agent," use this to diagnose.

## How Bootstrap Finds the Agent Dir

The `discover_agent_dir()` function checks candidates in order:

1. **`HERMES_WEBUI_AGENT_DIR` env var** — explicit override, highest priority
2. **`~/.hermes/hermes-agent/`** — standard install location (inside HERMES_HOME)
3. **`../hermes-agent`** — sibling directory (when companion repo is cloned next to agent)
4. **`~/.hermes/hermes-agent/`** (absolute form) — fallback
5. **`~/hermes-agent/`** — home directory clone
6. **CLI shebang parsing** — reads the `hermes` binary's first line (`#!/path/to/venv/bin/python`), walks up parent dirs looking for `run_agent.py`. Last resort.

Validation: a candidate must exist AND contain `run_agent.py`.

## How Bootstrap Finds the Python Executable

The `discover_launcher_python(agent_dir)` function checks:

1. **`HERMES_WEBUI_PYTHON` env var** — explicit override
2. **Agent venv** — if agent_dir found, checks `agent_dir/venv/bin/python` then `agent_dir/.venv/bin/python`
3. **Repo's own venv** — `REPO_ROOT/.venv/bin/python`
4. **System Python** — `shutil.which("python3")` or `sys.executable`

## The Import Check

`ensure_python_has_webui_deps()` runs a subprocess that verifies both yaml AND the agent can be imported:

```python
script = "import yaml\nfrom run_agent import AIAgent\n"
```

The agent dir is prepended to `PYTHONPATH` so `run_agent` resolves correctly.

If the check fails:
1. It tries agent_dir venv candidates
2. Falls back to creating a local `.venv` in the repo with `pip install -r requirements.txt`
3. Retries the check with the new venv
4. Fails with a clear error message if neither works

## Common Failure: Minimal Agent Venv

A development or partial agent install may have only `pip`, `setuptools`, `wheel`, and `pyyaml` in its venv. The `from run_agent import AIAgent` import will fail because the agent depends on packages like `requests`, `httpx`, `rich`, `python-dotenv`, `jinja2`, `pydantic`, `prompt_toolkit`, etc.

**Fix:** Install the agent's core dependencies into its venv:

```bash
~/hermes-agent/venv/bin/python -m pip install \
  requests httpx rich python-dotenv jinja2 pydantic \
  prompt_toolkit tenacity ruamel.yaml croniter fire pyyaml
```

Or install from the agent's `pyproject.toml`:
```bash
cd ~/hermes-agent && ~/hermes-agent/venv/bin/python -m pip install -e .
```

## Key Env Vars Summary

| Variable | Purpose | Default |
|---|---|---|
| `HERMES_WEBUI_AGENT_DIR` | Agent checkout path | auto-discovered |
| `HERMES_WEBUI_PYTHON` | Python executable | auto-discovered |
| `HERMES_WEBUI_HOST` | Bind address | 127.0.0.1 |
| `HERMES_WEBUI_PORT` | Port | 8787 |
| `HERMES_WEBUI_STATE_DIR` | State/sessions directory | ~/.hermes/webui |
| `HERMES_WEBUI_PASSWORD` | Auth password | (unset = no auth) |
