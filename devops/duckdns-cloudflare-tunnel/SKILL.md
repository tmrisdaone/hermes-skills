---
name: duckdns-cloudflare-tunnel
description: Set up a public web tunnel using DuckDNS + Cloudflare Tunnel (trycloudflare) on Android/Termux
---

# DuckDNS + Cloudflare Tunnel Workflow

Use when the user wants to make a local web server publicly accessible via their DuckDNS domain.

## Steps

### 1. Update DuckDNS with current IP
```bash
DOMAIN="<duckdns-subdomain>"
TOKEN="<token>"
curl -sL "https://duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip="
```
Verify: `host <domain>.duckdns.org`

### 2. Set up auto-update cron (every 5 min)
Create script at `~/.hermes/scripts/duckdns-update.sh`:
```bash
#!/data/data/com.termux/files/usr/bin/bash
DOMAIN="<domain>"
TOKEN="<token>"
curl -sL "https://duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip=" > /dev/null
echo "$(date): DuckDNS updated" >> ~/duckdns.log
```
Then register via `cronjob` tool with `no_agent=true`:
```yaml
action: create
name: duckdns-update
schedule: "every 5 minutes"
script: duckdns-update.sh
no_agent: true
```

### 3. Start local server + Cloudflare tunnel
```bash
# Background: start Python HTTP server
cd <site-dir> && python3 -m http.server 8080 &

# Background: start Cloudflare tunnel
cloudflared tunnel --url http://localhost:8080 > ~/cloudflared.log 2>&1 &
```

### 4. Get tunnel URL
```bash
grep "trycloudflare" ~/cloudflared.log | grep -oP 'https://[^\s]+'
```

### Cloudflare Tunnel Login (Interactive)

When a permanent named tunnel is needed (not ephemeral trycloudflare):

1. **Start login in background**, redirect output to a file:
   ```bash
   cloudflared tunnel login > /path/to/cf-login.log 2>&1
   ```

2. **Wait ~4 seconds**, then read the log for the auth URL:
   ```bash
   cat /path/to/cf-login.log
   # Extracts a URL like:
   # https://dash.cloudflare.com/argotunnel?aud=&callback=https%3A%2F%2F...
   ```

3. **Present the URL to the user** — they must open it in their browser, log into Cloudflare, and click "Authorize". The process blocks until this completes, and the cert is saved to `~/.cloudflared/cert.pem`.

4. **Verify**: `ls -la ~/.cloudflared/cert.pem`

### Pitfall — Login Timed Out
If the user doesn't authorize quickly enough, the login times out. Re-run `cloudflared tunnel login` — each invocation generates a unique callback URL.

### Pitfall — Multiple Login Sessions
Running successive `cloudflared tunnel login` commands before the previous one completes or is killed will conflict. Always `pkill -f "cloudflared tunnel login"` first.

## Named Tunnel Setup (Post-Login)

### 1. Create the tunnel
```bash
cloudflared tunnel create <tunnel-name>
# Creates: ~/.cloudflared/<tunnel-id>.json
# Outputs: "Created tunnel <tunnel-name> with id <tunnel-id>"
```

### 2. Configure ingress rules
Create `~/.cloudflared/config.yml`:
```yaml
tunnel: <tunnel-id>
credentials-file: /data/data/com.termux/files/home/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: yourdomain.com
    service: http://localhost:8080
  - service: http_status:404
```
The last rule (catch-all 404) is required — Cloudflare will reject the config without it.

### 3. Route DNS
```bash
# If a DNS record already exists, use --overwrite-dns:
cloudflared tunnel route dns --overwrite-dns <tunnel-name> yourdomain.com
# Without --overwrite-dns, fails with: "An A, AAAA, or CNAME record with that host already exists"
# Creates a CNAME: yourdomain.com → <tunnel-id>.cfargotunnel.com
```

### 4. Start the tunnel
```bash
cloudflared tunnel run <tunnel-name> > ~/cf-tunnel.log 2>&1
# Wait ~8s for connections. Check with:
tail -5 ~/cf-tunnel.log
# Should show: "Registered tunnel connection" from Cloudflare edge locations
```

## Important: CGNAT + DuckDNS Limitation

The user's Android phone is behind carrier NAT — DuckDNS domain is registered but **not directly reachable** from the internet. The Cloudflare tunnel (trycloudflare.com) is the actual access point. For proper DuckDNS domain access, need a VPS or Cloudflare login for named tunnel + Worker.

### ⚠️ 530 Origin Error Diagnosis

If you get **HTTP 530** when hitting Cloudflare's edge IPs with the DuckDNS domain:

```bash
# Test multiple Cloudflare anycast IPs
for ip in 104.16.0.1 104.16.1.1 172.64.0.1; do
  curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 \
    -H "Host: caloriesistmr.duckdns.org" "http://$ip/" 2>/dev/null
done
```

**530 Origin Unreachable** = Cloudflare's edge IS receiving the request but the origin (tunnel) can't be reached. Common causes:

| Cause | How to verify | Fix |
|---|---|---|
| Domain is **Pending** (nameservers not changed) | Check Cloudflare Dashboard — "Waiting for nameserver propagation" | Cannot fix with DuckDNS; need a real domain or VPS |
| Tunnel not running | `cloudflared tunnel list` shows 0 connections | Start the tunnel |
| Ingress rule not matching | `tail -f ~/cf-tunnel.log` shows no incoming requests | Check config.yml hostname matches exactly |
| DNS record is DNS-only (gray cloud) not proxied (orange) | Check Cloudflare DNS | `cloudflared tunnel route dns` auto-creates proxied records |

The **real constraint**: Cloudflare Tunnel DNS routing requires the domain to be **Active** (nameservers changed) on Cloudflare. DuckDNS subdomains (`*.duckdns.org`) can never change nameservers, so Cloudflare will never proxy traffic for them. This is a fundamental architectural limitation, not a configuration issue.

### What DOES Work with Named Tunnels
- **Quick Tunnel** (trycloudflare.com, no login required) — always works for testing
- **Named Tunnel with your own domain** (bought from a registrar, $1/yr) — full Cloudflare proxy + tunnel
- **DuckDNS + trycloudflare** — DuckDNS provides the domain name, trycloudflare provides the transport; they are complementary, not interchangeable

### cfargotunnel.com Endpoint
The tunnel's CNAME target (`<tunnel-id>.cfargotunnel.com`) resolves to an **IPv6 ULA address** (`fd10:*`) — this is **not publicly routable**. It's designed to be used exclusively behind Cloudflare's proxy. Direct access via curl will fail with "Could not connect to server".

## Reference Files
- `references/530-origin-error-resolution.md` — Detailed diagnostic steps, root cause analysis, and resolution options for the HTTP 530 error when pairing DuckDNS with Cloudflare Tunnel

## User Communication Style for This Environment
- Execute immediately — don't narrate what you're about to do
- When one approach fails, chain to the next without being asked
- Present interactive auth URLs as clean, tappable links
- After completing a step, state what's done and what's needed next concisely
- Respect hard constraints (e.g., "must be duckdns") — work within them, don't explain why they're difficult
