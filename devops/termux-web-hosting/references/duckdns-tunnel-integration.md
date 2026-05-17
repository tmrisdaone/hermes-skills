# DuckDNS + Cloudflare Tunnel Integration

## Complete Setup Flow

When user provides a DuckDNS domain + token:

```
Domain: caloriesistmr.duckdns.org
Token:  3c2798d4-9c3d-4af2-b6ba-ec288cd50746
```

### Step 1: Update DuckDNS
```bash
DOMAIN="caloriesistmr"
TOKEN="3c2798d4-9c3d-4af2-b6ba-ec288cd50746"
curl -sL "https://duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip="
# Returns: OK
```

### Step 2: Verify Resolution
```bash
host caloriesistmr.duckdns.org
# Should show current public IP
```

### Step 3: Start Local Server
```bash
cd /path/to/project && python3 -m http.server 8080
```

### Step 4: Start Cloudflare Tunnel (ephemeral)
```bash
cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel.log 2>&1
sleep 8
cat /tmp/tunnel.log | grep -o 'https://[^ ]*\.trycloudflare\.com'
```

### Step 5: Named Tunnel (if Cloudflare Login Completed)

If the user completed `cloudflared tunnel login` and has `~/.cloudflared/cert.pem`:

```bash
# Create named tunnel
cloudflared tunnel create txrao-site

# Write config.yml at ~/.cloudflared/config.yml:
# tunnel: <tunnel-id>
# credentials-file: /path/to/<tunnel-id>.json
# ingress:
#   - hostname: yourdomain.com
#     service: http://localhost:8080
#   - service: http_status:404

# Route DNS (--overwrite-dns if record exists):
cloudflared tunnel route dns --overwrite-dns txrao-site yourdomain.com

# Run:
cloudflared tunnel run txrao-site
```

### ⚠️ Named Tunnel Pitfall: 530 with DuckDNS

If you create a named tunnel for a `*.duckdns.org` domain and point DuckDNS to a Cloudflare IP, you will get **HTTP 530** (Origin Unreachable). This happens because:

1. The DuckDNS domain was likely added to Cloudflare in the past (stuck in "Pending" state)
2. Cloudflare won't proxy traffic until nameservers change
3. DuckDNS subdomains can never change nameservers
4. The tunnel is connected and healthy, but Cloudflare won't route through it

**Fix:** Use the ephemeral trycloudflare tunnel instead of the named tunnel for DuckDNS domains. Named tunnels only work with domains fully active on Cloudflare.

### 🚨 config.yml Interferes with Quick Tunnel

If a `~/.cloudflared/config.yml` exists (from a previous named tunnel setup), `cloudflared tunnel --url ...` picks it up. The trycloudflare hostname won't match the ingress rules, so all requests return 404.

**Fix:** Move the config before starting the quick tunnel:
```bash
mv ~/.cloudflared/config.yml ~/.cloudflared/config.yml.bak
cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel.log 2>&1
```

### `--overwrite-dns` Flag

When running `cloudflared tunnel route dns` for a domain that already has a DNS record, add `--overwrite-dns`:
```bash
cloudflared tunnel route dns --overwrite-dns txrao-site yourdomain.com
```

### DuckDNS Auto-Update Script
```bash
#!/data/data/com.termux/files/usr/bin/bash
DOMAIN="caloriesistmr"
TOKEN="3c2798d4-9c3d-4af2-b6ba-ec288cd50746"
curl -sL "https://duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip=" > /dev/null
echo "$(date): DuckDNS updated" >> /data/data/com.termux/files/home/duckdns.log
```

Create cron job (via Hermes cronjob tool):
```
action: create
name: duckdns-update
schedule: every 5 minutes
script: duckdns-update.sh
no_agent: true
```

## Important Constraints

- DuckDNS only supports A (IPv4) and AAAA (IPv6) records — no CNAME, no MX, no NS delegation
- You CANNOT add `subdomain.duckdns.org` to Cloudflare as a domain — nameserver changes are impossible
- The resolved IP is behind CGNAT and NOT directly reachable from the internet
- The tunnel (trycloudflare) provides the actual transport layer
- Present the DuckDNS domain as the primary URL to the user — they don't need to know about the tunnel

## Troubleshooting

| Symptom | Fix |
|---|---|
| DuckDNS returns `KO` | Token is wrong or domain doesn't exist |
| `host` shows old IP | Wait for TTL (usually 60s for DuckDNS) |
| Tunnel starts but 1033 error | Local server not running or wrong port |
| Quick tunnel returns 404 everywhere | config.yml ingress rules interfering — move it before starting |
| `route dns` fails with code: 1003 | Record already exists — add `--overwrite-dns` flag |
| DuckDNS IP changes | Cron job auto-update handles this every 5 min |
