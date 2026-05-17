# 530 Origin Error — Diagnosis & Resolution

## Context
User hosts a static shopping site on Android/Termux behind CGNAT:
- DuckDNS domain: `caloriesistmr.duckdns.org`
- Local server: Python `http.server` on `localhost:8080`
- Tunnel: Cloudflare named tunnel `txrao-site`
- User added the DuckDNS domain to Cloudflare (Pending state)

## Error
All requests to Cloudflare edge IPs return **HTTP 530**:
```http
HTTP/1.1 530 <none>
Content-Type: text/plain; charset=UTF-8
Content-Length: 16
```

## Diagnostic Steps Performed

### 1. Local Server Verified
```bash
curl http://localhost:8080/   # → 200 OK
```

### 2. Tunnel Connected
```bash
cloudflared tunnel info txrao-site
# → 4 connections: mia01, mia02, mia04, mia08
```

### 3. DNS Routing Added
```bash
cloudflared tunnel route dns --overwrite-dns txrao-site caloriesistmr.duckdns.org
# → "Added CNAME caloriesistmr.duckdns.org"
```

### 4. DuckDNS Updated to Cloudflare IP
```bash
curl -sL "https://duckdns.org/update?domains=caloriesistmr&token=xxx&ip=104.16.0.1"
# → OK
host caloriesistmr.duckdns.org @8.8.8.8
# → 104.16.0.1
```

### 5. Multiple Cloudflare IPs Tested
| IP | Result |
|---|---|
| 104.16.0.1 | 530 |
| 104.16.1.1 | 530 |
| 104.16.2.1 | 530 |
| 172.64.0.1 | 000 (connection refused) |
| 172.64.35.1 | 530 |

### 6. Tunnel's cfargotunnel.com Endpoint Checked
```bash
host 88a4fcfd-85b9-4089-8a55-ef49c01c41f2.cfargotunnel.com
# → IPv6 ULA: fd10:aec2:5dae::
# Not publicly reachable — designed for Cloudflare-proxied traffic only
```

## Root Cause
The domain `caloriesistmr.duckdns.org` was added to Cloudflare but is **stuck in "Pending Nameserver Update"** state because DuckDNS controls the `duckdns.org` zone — users cannot change nameservers. Cloudflare **only proxies traffic** for domains that are **Active** (nameservers verified).

## Resolution Options

### Option 1: Buy a Real Domain ($1/yr)
- Purchase `caloriesistmr.xyz` (or similar) from Namecheap/Porkbun
- Add to Cloudflare → change nameservers → activate
- Reuse the existing named tunnel with the new domain

### Option 2: Use Ephemeral trycloudflare (Free, Works Now)
```bash
cloudflared tunnel --url http://localhost:8080
# Provides a public URL immediately, no login needed
```

### Option 3: Wait (Unlikely to Help)
Once a domain is added to Cloudflare in pending state, it will never activate without nameserver changes. This is not a timing issue.

## Key Learning
Cloudflare Tunnel's DNS routing (`cloudflared tunnel route dns`) creates DNS records in Cloudflare, but they **only take effect** when Cloudflare is the authoritative DNS server. For domains on external DNS providers (like DuckDNS), Cloudflare cannot proxy traffic — even though the DNS record exists and the tunnel is connected.

The **530 error** is Cloudflare's way of saying "I can see this request, but I can't complete it because the domain setup is incomplete."
