---
name: termux-deployment
description: Guidelines for deploying and hosting web services and agents within the Termux environment, focusing on networking and hardware API integration.
---

# Termux Deployment & Hosting

This skill governs the deployment of web servers, AI agents, and hardware-integrated tools specifically within the Termux Android environment.

## 🌐 Networking & Tunneling
Termux lives behind Android's NAT/Firewall, making standard `localhost` inaccessible from the outside world.

### 1. Local Network Access
For devices on the same Wi-Fi, use the internal IP.
- **Command:** `ip addr show wlan0`
- **Access:** `http://<internal-ip>:<port>`

### 2. Public Tunnels (The "Tunneling Hierarchy")
When `localhost` must be public, prioritize tools in this order based on reliability in Termux:

1. **Cloudflare Tunnel (`cloudflared`):** Best for production-like URLs. 
   - *Pitfall:* Frequently fails due to IPv6 DNS issues (`[::1]:53` connection refused). 
   - *Fix:* Force IPv4 DNS by updating `/etc/resolv.conf` or using `--url http://127.0.0.1:port` instead of `localhost`.
2. **Pinggy / Localhost.run:** SSH-based tunnels.
   - *Command:* `ssh -R 80:localhost:port a.pinggy.io`
   - *Pros:* No installation required. 
   - *Cons:* May be blocked by corporate/school firewalls on port 22.
3. **Ngrok:** Reliable but requires account/token setup.

## 📱 Hardware API Integration (Termux:API)
To access Android hardware (Camera, SMS, TTS, STT), the `termux-api` package is not enough.

### 🛠️ The Critical Two-Step Setup
1. **CLI Package:** `pkg install termux-api` (The bridge scripts).
2. **Companion App:** The **Termux:API Android App** must be installed from the same source (F-Droid/GitHub) as the main Termux app.

### Common API Tools
- **Voice:** `termux-tts-speak "text"` (Output) $\rightarrow$ `termux-speech-to-text` (Input).
- **Notifications:** `termux-toast "message"`.
- **Screenshots:** `termux-screenshot <path>` (Requires high-level Android permissions granted manually in App Info).

## 🐍 Python Hosting Patterns
When hosting "Single-File" sites or agents, avoid directory-based servers (`http.server`) if the file structure is volatile.

### The "Embedded Asset" Pattern
For maximum portability and to avoid `FileNotFoundError` in complex paths, embed HTML/CSS as base64 strings inside the Python server script.

- **Benefit:** Zero external dependencies; the script is the website.

## 🧩 Hermes Companion Tool Installation

When installing tools that extend Hermes Agent (Hermes WebUI, dashboards, bridges, etc.), these tools typically need to import from the agent's codebase and use its venv.

### Discovery Mechanism Pattern

Companion tool bootstrap scripts usually auto-discover the agent via this priority chain:

1. **Specific env var** — e.g. `HERMES_WEBUI_AGENT_DIR`, `HERMES_WEBUI_PYTHON`
2. **`~/.hermes/hermes-agent/`** — standard install location
3. **Sibling directory** — `../hermes-agent` relative to the companion repo
4. **CLI shebang parsing** — follows the `hermes` binary's shebang to find `run_agent.py`

### Key Environment Variables

| Variable | Purpose |
|---|---|
| `HERMES_WEBUI_AGENT_DIR` | Point to the Hermes agent checkout directory |
| `HERMES_WEBUI_PYTHON` | Override Python executable (use agent venv python) |
| `HERMES_WEBUI_HOST` / `HERMES_WEBUI_PORT` | Bind address and port (default: 127.0.0.1:8787) |
| `HERMES_WEBUI_STATE_DIR` | Session/state storage (default: `~/.hermes/webui`) |

### Dependency Fix Pattern

Minimal/development agent venvs may lack packages the companion tool needs (the bootstrap checks `import yaml + from run_agent import AIAgent`). Fix by installing the missing deps into the agent's venv:

```bash
# Identify missing imports from the error traceback:
~/hermes-agent/venv/bin/python -c "import yaml; from run_agent import AIAgent"
# Install missing packages:
~/hermes-agent/venv/bin/python -m pip install <missing-package>
```

Common missing packages on a minimal venv: `requests`, `httpx`, `python-dotenv`, `rich`, `jinja2`, `pydantic`, `pyyaml`.

### Verification

- Check the health endpoint: `curl -s http://127.0.0.1:<port>/health`
- Check logs at `~/.hermes/webui/bootstrap-<port>.log`

For Hermes WebUI specifically, manage with `ctl.sh` from the repo root:
```bash
./ctl.sh status     # PID, uptime, bound host/port, log path
./ctl.sh logs       # tail ~/.hermes/webui.log
./ctl.sh restart    # graceful restart
./ctl.sh stop       # terminate daemon
```

See the companion tool discovery reference (`references/companion-tool-discovery.md`) for the detailed agent-finding logic used by bootstrap scripts.

## ⚠️ Pitfalls & Lessons
- **IPv6 DNS:** Termux often attempts DNS lookups via IPv6 loopback, causing tunnel failures. Always prefer `127.0.0.1` over `localhost` in config strings.
- **Platform Checks & Binary Compatibility:** Some Node.js/Python libraries (e.g., `onnxruntime-node`) have explicit platform checks or lack pre-compiled binaries for Android ARM64, causing "Unsupported Platform" errors. 
  - **The Proot Solution:** When a tool requires a standard Linux environment (glibc/binutils) that Termux's bionic libc doesn't provide, wrap the execution in a `proot-distro` login:
    `proot-distro login <distro> -- bash -c "command_here"`
- **Setup Persistence:** Installing heavy suites (like Ubuntu + build-essential) via proot can timeout due to the volume of packages. Separate the `apt update` from the `install` phase to ensure progress is saved.
- **Permissions:** API commands will fail silently or return "command not found" if the companion app is missing or permissions were not granted in Android's App Info settings.
