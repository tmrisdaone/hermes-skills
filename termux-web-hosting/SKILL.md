---
name: termux-web-hosting
description: Guidelines for deploying and exposing local websites from within a Termux environment, including asset embedding and tunnel troubleshooting.
---

# Termux-Based Web Hosting & Tunnels

Guidelines for deploying and exposing local websites from within a Termux environment.

## Core Workflow
1. **Server Setup:** Use a standalone Python script (using `http.server` or `socketserver`) that embeds assets as base64 strings to avoid filesystem path / permission issues in shared environments.
2. **Port Mapping:** Default to port 8080 for local development.
3. **Public Exposure:** 
    - Attempt `cloudflared` (Cloudflare Tunnel) as a first choice for stability.
    - Use `localhost.run` or `Pinggy` as SSH-based fallbacks.

## Pitfalls & Fixes

### 🚨 config.yml Ingress Rules Break Quick (trycloudflare) Tunnels

When you have a `~/.cloudflared/config.yml` with ingress rules from a previous named tunnel, even a **quick tunnel** (`cloudflared tunnel --url ...`) picks up those rules. The config.yml applies globally — it is NOT scoped to named tunnels.

**What happens:** The trycloudflare hostname (e.g. `foo-bar.trycloudflare.com`) won't match your ingress rules (which specify your custom domain), so it falls through to the `http_status:404` default — returning **404 on every request**.

**Fix:** Move config.yml before starting a quick tunnel:
```bash
mv ~/.cloudflared/config.yml ~/.cloudflared/config.yml.bak
cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel.log 2>&1
mv ~/.cloudflared/config.yml.bak ~/.cloudflared/config.yml   # restore later
```

### Running Named + Quick Tunnels Simultaneously

Both can run at the same time using different cloudflared processes. Useful when:
- **Named tunnel** routes your custom domain (may return 530 until Cloudflare activates)
- **Quick tunnel** provides immediate trycloudflare URL access

**Caveat:** Move config.yml (see above) when running the quick tunnel alongside.

### `--overwrite-dns` for Existing Records

When routing a domain to a named tunnel and a DNS record already exists:
```bash
cloudflared tunnel route dns --overwrite-dns txrao-site yourdomain.com
```
Without `--overwrite-dns`, this fails with `code: 1003` — "An A, AAAA, or CNAME record with that host already exists."

### The IPv6 DNS Bug:** Termux often attempts to resolve DNS via `[::1]:53` (IPv6 loopback), causing a "connection refused" error for tools like Cloudflare.
    - *Fix:* Set the nameserver explicitly in `/etc/resolv.conf` to `8.8.8.8` or use the `CLOUDFLARED_DNS_SERVER` env var.
- **Android Platform Checks:** Many Node.js-based tunnels (like `localtunnel` and `instatunnel`) explicitly block the `android` platform or use pre-compiled binaries incompatible with Termux's file system. Do not suggest these; prioritize `cloudflared` which is available via `pkg install cloudflared`.
- **Cloudflare Tunnel Output Redirection:** Cloudflared writes its public URL to stderr. Always redirect both stdout+stderr to a file, then grep for the URL:
    ```
    cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel.log 2>&1
    sleep 8
    grep -o 'https://.*\.trycloudflare\.com' /tmp/tunnel.log
    ```
- **Cloudflare Tunnel Start Latency:** Don't check the log immediately. The tunnel takes 5–10 seconds to register. Wait 8 seconds before reading.
- **Cloudflare Tunnel 1033 Error:** This indicates the tunnel is active but the local server is unreachable.
    - *Fix:* Use `http://127.0.0.1:PORT` instead of `http://localhost:PORT` in the tunnel command to bypass potential DNS resolution issues on Android.
- **Cloudflare Tunnel 530 Error (Origin Unreachable):** This means Cloudflare's edge received the request but can't reach the origin through the tunnel. Common causes:
    - Domain is in "Pending" state (nameservers not changed) — Cloudflare won't proxy. **Cannot fix with DuckDNS subdomains.**
    - Tunnel isn't running — check `cloudflared tunnel list` for 0 connections.
    - Ingress rule doesn't match — verify `hostname:` in config.yml matches exactly.
    - DNS record is DNS-only (gray cloud) not proxied (orange cloud) — `cloudflared tunnel route dns` auto-creates proxied records.
    - *See `references/duckdns-tunnel-integration.md` for full diagnosis.*
- **pgvector on Android ARM64:** Compiles from source but fails at runtime with `cannot locate symbol "acos"`. Fix: re-link with `-lm` flag. See `references/pgvector-android.md` for full procedure.
- **Browser-Sync Setup:** For live-reloading in Termux:
    - Install via `npm install -g browser-sync`.
    - Ensure the server is started from the exact directory containing the files to avoid "Cannot GET" errors.
    - Usage: `browser-sync start --server --files \"*.html, *.css\"`
- **Browser-Sync `--tunnel` Failure:** The `--tunnel` option (localtunnel.me) hangs indefinitely with no output in Termux. Do not use it. Stick to cloudflared for public exposure.
- **SSH Port Blocking:** Some mobile networks block port 22. If `ssh` based tunnels fail, suggest the user connects to a VPN (Warp, Proton) to bypass network-level firewalling.

## Cloudflare Login (for Named Tunnels)

When you need a permanent tunnel with your own domain (not ephemeral trycloudflare), you must authenticate:

1. **Start login** (runs in headless Termux, outputs a browser URL):
   ```bash
   cloudflared tunnel login > /path/to/login.log 2>&1
   ```
   Wait ~3s, then read the log to extract the URL:
   ```bash
   cat /path/to/login.log
   # Looks like:
   # https://dash.cloudflare.com/argotunnel?aud=&callback=https%3A%2F%2Flogin.cloudflareaccess.org%2F...
   ```

2. **Present the URL to the user** — they must open it in their browser, log into Cloudflare, select a domain, and authorize. The login process blocks until this completes.

3. **After authorization**, the certificate (`~/.cloudflared/cert.pem`) is saved automatically. The login command exits cleanly.

4. **Create a named tunnel** (once authenticated):
   ```bash
   cloudflared tunnel create my-tunnel-name
   # Saves credentials to ~/.cloudflared/<tunnel-id>.json
   ```

5. **Configure DNS** to point your domain to the tunnel:
   ```bash
   cloudflared tunnel route dns my-tunnel-name yourdomain.com
   ```

6. **Run the named tunnel**:
   ```bash
   cloudflared tunnel run my-tunnel-name
   # Or with a config file pointing to localhost:8080
   ```

### Important Notes
- The login is interactive — the cloudflared process waits for the user to complete authentication in their browser. Run it as a background process and poll the log.
- Each `cloudflared tunnel login` invocation generates a unique callback URL that expires. If the user doesn't complete auth before timeout, re-run.
- After login, `~/.cloudflared/` directory is created with `cert.pem` and `config.yml` (default empty).
- The ephemeral (trycloudflare) tunnel requires NO login — it's a quick way to test without a Cloudflare account.

## DuckDNS Integration (Dynamic DNS)

DuckDNS is a free dynamic DNS service. On Android/Termux you can pair it with Cloudflare Tunnel to get both a clean domain name and actual public access.

### Registering / Updating a DuckDNS Domain

1. User signs in at https://duckdns.org (Google/GitHub/Twitter auth)
2. They get a **token** and choose a **subdomain** (e.g., `myproject`)
3. Update the domain via API:
   ```bash
   curl -sL "https://duckdns.org/update?domains=myproject&token=YOUR_TOKEN&ip="
   ```
   Omitting `ip=` makes DuckDNS auto-detect the requester's IP. Response `OK` means success.

### 🚫 Cloudflare + DuckDNS Nameserver Limitation

If a user adds their `subdomain.duckdns.org` to Cloudflare, Cloudflare will ask them to change nameservers. **This is impossible** — DuckDNS owns the `duckdns.org` zone and users cannot delegate authority.

**Do NOT** suggest adding a DuckDNS subdomain to Cloudflare as a "domain" — it will get stuck in "Pending Nameserver Update" forever.

**What DOES work:**
- Keep DuckDNS as the DNS provider (A record only)
- Use Cloudflare Tunnel in ephemeral (trycloudflare) mode for transport — no Cloudflare account needed
- If the user wants a named tunnel, they must own a real domain (buy from Namecheap/Porkbun for ~$1/yr)

### User Expectation: DuckDNS as the Primary Domain

Users who set up a DuckDNS domain expect that domain to be **the** URL they share. They do NOT want to hand out a random trycloudflare URL. Always present the solution in terms of their DuckDNS domain — the tunnel is invisible infrastructure.

**How to present it:** "Your site is at myproject.duckdns.org" — the tunnel URL is an implementation detail.

Android phones are almost always behind **Carrier-Grade NAT (CGNAT)**. The IP that DuckDNS resolves to is a shared carrier IP — it is **not directly reachable** from the internet. DuckDNS alone will NOT make the site accessible.

**Correct architecture: DuckDNS + Tunnel**
- DuckDNS provides a **clean domain name** (e.g., `myproject.duckdns.org`)
- Cloudflare Tunnel (trycloudflare) provides **actual public routing**
- They are complementary: DuckDNS for DNS, tunnel for transport

### Auto-Update Cron Job

Phone IPs change over time. Set up a recurring update using Hermes cron:
```bash
# Write a script to ~/.hermes/scripts/duckdns-update.sh:
#!/data/data/com.termux/files/usr/bin/bash
DOMAIN="myproject"
TOKEN="YOUR_TOKEN"
curl -sL "https://duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip=" > /dev/null
echo "$(date): DuckDNS updated" >> /data/data/com.termux/files/home/duckdns.log

# Then create a cron job via the cronjob tool:
#   action: create
#   name: duckdns-update
#   schedule: every 5 minutes
#   script: duckdns-update.sh   (relative to ~/.hermes/scripts/)
#   no_agent: true
```

### DuckDNS Verification
```bash
host myproject.duckdns.org
# Should return the current public IP
```

## Multi-Page Site Generation from Jinja2 Templates

When you have Flask/Jinja2 template files and need to convert them to standalone HTML for static hosting (e.g., with Python http.server):

1. **Read each template** — they use `{% extends "base.html" %}` + `{% block content %}` pattern
2. **Extract the body** between `{% block content %}` and `{% endblock %}`
3. **Replace Jinja2 template refs**:
   - `{{ url_for('page_name') }}` → `page_name.html`
   - `{{ url_for('index') }}` → `index.html`
   - `{{ url_for('shop') }}` → `index.html#products`
4. **Strip extends/block tags** using regex: `re.sub(r"{% extends.*?%}", "", body)`
5. **Wrap each body** with a full HTML shell containing:
   - The same `<style>` block as the main index.html (for consistent theming)
   - The same header with correct nav links pointing to actual .html files
   - The same footer with correct links
   - Same animated background divs (bg-grid, bg-glow-1-3, bg-particles)
   - Same JS (particles init, scroll reveal, header scroll, toast system)
6. **Add a back-to-top button** to every sub-page (it's a small UX touch users expect)
7. **Add page-specific JS** — e.g., contact form handler with toast feedback

### Navigation Wiring Pattern

When wiring up navigation links across pages:
```html
<!-- Header -->
<li><a href="support.html">Support</a></li>

<!-- Footer Company -->
<li><a href="about.html">About</a></li>
<li><a href="careers.html">Careers</a></li>
<li><a href="contact.html">Contact</a></li>

<!-- Footer Legal -->
<li><a href="privacy.html">Privacy</a></li>
<li><a href="terms.html">Terms</a></li>

<!-- Footer Shop (point back to main page anchors) -->
<li><a href="index.html#products">Wearables</a></li>
```

## User Communication Style (this environment)
- **Terse commands expected**: "login into cloudflare", "START it up", "Can u use broswer sync" — just execute, don't explain what you're about to do.
- **Offer alternatives proactively**: If the first tool fails, immediately suggest the fallback without being asked. User said "use browser-sync... that doesnt work u can use cloudflare" — they expect you to chain through options.
- **Respect hard preferences**: When user says "I cant use anything eles must be duckdns" — stop suggesting alternatives and work within their constraint. Don't explain why it's hard, just solve it.
- **Present URLs for interactive steps directly**: When headless auth is needed, paste the full URL clearly so the user can tap it on their phone.
- **Concisely report results**: After completing a step, state what's done and what's needed next. No paragraphs of explanation.
- **Assume the user knows what they want**: Don't debate choices. If they want DuckDNS over trycloudflare, make it work even if it requires creative architecture.

## Verification
- Check if the local server is active: `curl -I http://localhost:8080`
- Verify tunnel connectivity by visiting the provided public URL from an external device.
