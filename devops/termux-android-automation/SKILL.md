---
name: termux-android-automation
description: Guidelines for controlling Android hardware via Termux and exposing local services to the internet.
---

# Termux Android Automation & Tunneling

### 🎙️ Voice AI Assistant Architecture (Wake-Word Pattern)
To implement a non-blocking, wake-word activated voice agent:
1. **Client Loop:** A background Python script that continuously calls `termux-speech-to-text`.
2. **Wake-Word logic:** Local string matching (e.g., `if "hermes" in text:`) to filter noise.
3. **API Bridge:** A lightweight server (FastAPI/Flask) acting as a mailbox. The Client posts triggers here; the Agent monitors this queue.
4. **TTS Feedback:** Immediate feedback via `termux-tts-speak` to confirm the wake-word was heard before the LLM processes the request.

**Pitfall:** When installing Chromium for headless automation in Termux, use `pkg install chromium`. Note that the executable binary may be named `chromium-browser` rather than `chromium`.


### Common Commands
- **Speech to Text:** `termux-speech-to-text` (Returns user speech as text)
- **Text to Speech:** `termux-tts-speak "message"`
- **Notifications:** `termux-toast "message"` or `termux-notification`
- **URLs:** `termux-open-url <url>`
- **Screenshots:** `termux-screenshot <path>` (Requires specific Android permissions)

### 🛠 Troubleshooting and Pitfalls
- **NPM Installation Failures:** Avoid installing `termux-api` via `npm install`. This often triggers native compilation errors (e.g., `better-sqlite3` failing due to missing `android_ndk_path`). Always use the system package manager: `pkg install termux-api`.
- **Permission Requirements:** Ensure the **Termux:API Android App** is installed from the same source as Termux (F-Droid/GitHub) and granted all system permissions in Android Settings.
When exposing `localhost` services (e.g., port 8080) to the internet, use tunnels to bypass NAT/Firewalls.

### Tool Comparison & Selection
| Tool | Method | Best For | Pitfalls |
|---|---|---|---|
| **Cloudflare Tunnel** | `cloudflared tunnel --url` | High stability, HTTPS | Often fails in Termux due to IPv6 DNS loopback errors (`[::1]:53`) |
| **LocalTunnel** | `lt --port` | Quick setup | May crash on Android due to `unsupported platform` check in `openurl` dependency |
| **Bore / Pinggy** | SSH Reverse Tunnel | Bypassing HTTP APIs | May be blocked by network ISP on Port 22 (SSH) |

### 🛠 Troubleshooting DNS/Tunnel Failures
If tunnels fail with `connection refused` or `lookup failed` while `ping google.com` works:
1. **Force IPv4 DNS:** Try setting nameservers explicitly: `echo "nameserver 8.8.8.8" > $PREFIX/etc/resolv.conf`.
2. **SSH Fallback:** If HTTP-based tunnels fail, try an SSH tunnel (like `ssh -R 80:localhost:8080 a.pinggy.io`).
3. **Local IP:** As a fallback for testing, use the device's local IP found via `ip addr show wlan0`.

### 🖥️ Mobile Desktop (HackLab) Environment
For managing full X11 desktop environments (like XFCE4) within Termux:
- **Startup:** Use dedicated scripts (e.g., `start-hacklab.sh`) to initialize the audio server and X11 display server before launching the window manager (`startxfce4`).
- **Execution:** Since desktop environments are blocking processes, they **must** be run in the background or a separate terminal session to allow the agent to remain responsive.
- **Verification:** The user must switch to the **Termux-X11** app to interact with the graphical interface.
- **Interruption:** Use corresponding shutdown scripts (e.g., `stop-hacklab.sh`) to clean up sessions and kill X server processes.

**Pitfall:** Running XFCE4 in the foreground will cause the agent's tool-call to time out or hang until the session is ended. Always use background execution (`background: true` in tool calls or `nohup` / `&` in shell).

### 🛠 Troubleshooting & Tips
- **Wake-Word Implementation:** To create a \"Wake-Word\" listener, use a loop with `termux-speech-to-text`, convert the output to lowercase, and use a conditional check for the trigger word (e.g., `\"hermes\"`) before relaying the command to an API or script.
- **Rish Usage:** If moving tools like `rish` from `/sdcard/Download` to home, ensure you `chmod +x` the binary. Launch apps using `rish -c \\\"am start -n <package>/<activity>\\\"`.
- **UI Interaction:** Use `rish -c \\\"input tap <x> <y>\\\"` for screen interactions. Note that coordinates vary by device resolution; verify with a screenshot or a coordinate-picker app.
4. **Action:** Integration with `phone_control.sh` (Deep Links) for app-specific actions.
For complex UI tasks that cannot be handled by deep-links (e.g., navigating a specific app menu):
- Use `uiautomator dump` to get the UI hierarchy XML.
- Use `screencap` for visual state.
- Use `input tap <x> <y>`, `input swipe`, and `input text` for interactions.
- **Root Requirement:** Most `input` and `screencap` commands require root (`su -c`) or ADB/Shizuku access to function on non-rooted devices.
