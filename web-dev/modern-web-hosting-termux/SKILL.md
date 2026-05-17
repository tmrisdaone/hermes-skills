---
name: modern-web-hosting-termux
description: Workflow for creating high-visual modern websites and hosting them (locally and publicly) in a Termux environment.
---

# Modern Website Generation & Hosting (Termux)

Workflow for creating high-visual, modern single-page websites and hosting them within a Termux environment.

## Core Design Standards
- **Visuals:** Prioritize "modern" trends: Glassmorphism (`backdrop-filter`), animated CSS gradients, floating background blobs, and Inter/system-sans typography.
- **Architecture:** Use semantic HTML5. Keep CSS separate or embedded via base64 for standalone python hosting.
- **Responsiveness:** Always use `@media` queries for mobile-first or adaptive layouts.

## Hosting Strategies
### 1. Directory Hosting (Standard)
Use `python3 -m http.server <port>` for rapid testing of directories containing `index.html`.

### 2. Standalone "Single-File" Hosting (Portable)
To avoid directory mapping issues or to create a portable host:
- Encode HTML/CSS assets as `base64` within a Python script.
- Use `http.server.SimpleHTTPRequestHandler` to serve these blocks as raw bytes on request.
- This bypasses filesystem path errors and allows the site to be hosted by a single `.py` file.

### 3. Flask-Based Hosting (Integrated Backend)
For projects requiring dynamic content or API endpoints:
- Install Flask: `pip install flask`
- Run a combined backend/frontend server using the `app.py` pattern demonstrated below.
- Ensure dependencies are installed before launching.

#### Sample Flask Integration Pattern:
```python
from flask import Flask, send_from_directory
import os

app = Flask(__name__)

@app.route('/')
def index():
    return send_from_directory('.', 'index.html')

@app.route('/<path:path>')
def static_files(path):
    return send_from_directory('.', path)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.getenv('PORT', 5000)))
```

## Public Exposure (Tunnels)
When `localhost` is insufficient, use tunnels to expose the port to the internet.

### Preferred: Cloudflare Tunnel (trycloudflare.com)
```
cloudflared tunnel --url http://localhost:<port> > /tmp/cloudflared.log 2>&1
# Wait ~8 seconds then read the log for the public URL:
grep -o 'https://.*\.trycloudflare\.com' /tmp/cloudflared.log
```
- **Availability:** `pkg install cloudflared` — system-level, works reliably in Termux.
- **Output:** Cloudflared writes to stderr. Redirect both stdout+stderr to a file, then grep for the trycloudflare.com URL from the log.
- **Pitfall — no output on startup:** Cloudflared can take 5–10 seconds to connect. Don't check immediately — wait 8s then read the log file.
- **DNS Fix:** If API lookups fail (`connection refused` on `[::1]:53`), try `nameserver 8.8.8.8` in `$PREFIX/etc/resolv.conf` or use `http://127.0.0.1:PORT` instead of `localhost`.
- **Pitfall — 1033 error:** The tunnel is active but local server unreachable. Use `127.0.0.1` instead of `localhost` in the tunnel command.

### Fallback: BrowserSync (live-reload + optional tunnel)
```
# Local only (live-reload):
browser-sync start --server --files "*.html, *.css"

# With public tunnel (may not work in Termux — see pitfall below):
browser-sync start --server --tunnel --no-open
```
- **Pitfall — `--tunnel` hangs in Termux:** The `--tunnel` option (uses localtunnel.me) often hangs with zero output on Android/Termux. The process starts but never produces a URL. **Do not rely on BrowserSync tunnels in Termux.** Use Cloudflare Tunnel instead.
- **Live-reload does work:** BrowserSync without `--tunnel` works perfectly for hot-reload development.

### Last-Resort Tunnels
- **SSH tunnels:** `ssh -R 80:localhost:<port> -o StrictHostKeyChecking=no -N localhost.run` or `ssh -R 80:localhost:<port> a.pinggy.io`
- **Pitfall:** Some mobile networks block port 22. Try VPN (Cloudflare WARP, Proton) to bypass.
- **Local network fallback:** Use the device's local IP (`ip addr show wlan0` | grep inet) on the same WiFi.

### Platform Pitfall (NPM Tunnels)
Many NPM-based tunnel packages (`localtunnel`, `instatunnel`, `browser-sync --tunnel`) fail in Termux. They either check `os: 'linux'` and reject `os: 'android'`, or use pre-compiled binaries incompatible with Termux. **Prefer `pkg install cloudflared` over NPM tunnels.**

### Verification
```
# Local server check:
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/
# Public URL check:
curl -s -o /dev/null -w "%{http_code}" https://your-tunnel.trycloudflare.com/
```

## Converting Flask/Jinja2 Templates to Static HTML
When moving from a dynamic Flask prototype to a statically-deployable site:

1. **Extract the core CSS** from the Flask app's base template — this becomes your inline `<style>` block.
2. **Materialize each template** by replacing Jinja2 syntax with static equivalents:
   - `{% extends "base.html" %}` → remove (embed base HTML directly)
   - `{% block title %}...{% endblock %}` → extract title text
   - `{% block content %}...{% endblock %}` → extract body content
   - `{{ url_for('page') }}` → replace with `page.html`
   - `{{ url_for('static', filename='...') }}` → replace with direct path or inline
3. **Build a shared template function** in your generation script that wraps each page's body content with the common `<header>`, `<footer>`, background divs, particles, and toast system.
4. **Keep consistent navigation** — all pages should point to each other via relative links (`about.html`, `support.html`, etc.), not hash anchors.
5. **Add interaction handlers** per page (e.g., contact form submit → toast notification).
6. **Serve with `python3 -m http.server <port>`** — no Flask needed.

## Multi-Page Architecture Conventions
- **Single source of truth for CSS:** Keep one inline `<style>` block (or one `.css` file) used by all pages.
- **Navigation:** Header uses `<a href="about.html">` not `<a href="#">`. Footer links go to actual `.html` files.
- **Back-to-top button:** Include on every sub-page using a shared `<button class="back-to-top">` with matching JS.
- **Toast system:** Include `toast-container` div and `showToast()` JS function in every page's common footer.
- **Particles/background:** Include the same animated background divs (`bg-grid`, `bg-glow-*`, `bg-particles`) on every page for visual consistency.

### Reference: Multi-Page Static Conversion
See `references/multi-page-static-conversion.md` for a complete walkthrough of converting Flask/Jinja2 templates to standalone static HTML pages, including the exact replacement table and generation script pattern.

## Full-Stack Integration Workflow
When moving from a static site to a functional store:
- **Backend Setup:** Use a dedicated `/backend` directory with a `package.json` to manage dependencies (`express`, `cors`, `body-parser`).
- **State Management:** Avoid simple JS arrays for complex apps. Implement a centralized `Store` object with a Subscription/Observer pattern to automatically sync the UI with the data state.
- **Dynamic Rendering:** Fetch product data from the API (`GET /api/products`) and render the grid dynamically via JavaScript rather than hard-coding HTML.
- **Process Orchestration:** Run the Backend and Frontend (BrowserSync) as simultaneous background processes. Use `pkill -f` to clear orphaned processes if port conflicts occur.
- **Cache Busting:** When CSS doesn't apply despite being present, use versioning in the link tag (e.g., `style.css?v=1.1`) to force the browser to bypass cache.
