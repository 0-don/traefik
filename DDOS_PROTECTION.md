# DDoS Protection Playbook

## Architecture

```
Client -> Cloudflare Edge (WAF + DDoS L7) -> Kernel (conntrack) -> Traefik (CrowdSec + middlewares) -> Backend
```

Cloudflare blocks at the edge. Kernel conntrack tuning prevents connection table exhaustion. Traefik middlewares isolate services so one overwhelmed backend cannot starve others. CrowdSec catches what slips through. Backend containers have memory limits.

## Quick Reference

### Cloudflare API

```bash
EMAIL="don.cryptus@gmail.com"
KEY="<your-global-api-key>"
SERVER_IP="<your-server-ip>"
ZONE_CG="b48b416ddd4ab255df880eb756464b20"  # coding.global
ZONE_UR="09567baf778ae9946a29a034811323c6"  # unorouter.ai
```

### Check if an attack is happening

```bash
# Cloudflare: requests and threats in last 3 hours
curl -s "https://api.cloudflare.com/client/v4/graphql" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  -H "Content-Type: application/json" \
  --data "{
    \"query\": \"{ viewer { zones(filter: {zoneTag: \\\"$ZONE_CG\\\"}) { httpRequests1hGroups(limit: 3, filter: {datetime_geq: \\\"$(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ)\\\", datetime_leq: \\\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\\\"}, orderBy: [datetime_ASC]) { dimensions { datetime } sum { requests threats responseStatusMap { edgeResponseStatus requests } } } } } }\"
  }" | jq '.data.viewer.zones[0].httpRequests1hGroups[] | {hour: .dimensions.datetime, requests: .sum.requests, threats: .sum.threats}'

# Conntrack table (if near max, ALL sites go down)
ssh don@$SERVER_IP "cat /proc/sys/net/netfilter/nf_conntrack_count && echo '/' && cat /proc/sys/net/netfilter/nf_conntrack_max"

# Traefik CPU (>300% means attack traffic is overwhelming)
ssh don@$SERVER_IP "docker stats --no-stream traefik"
```

### Server side logs

```bash
# Live tail (watch for 499 floods, unusual IPs, repeated paths)
ssh don@$SERVER_IP "docker exec traefik tail -f /var/log/traefik/access.log"

# Top IPs by request count
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | awk '{print \$1}' | sort | uniq -c | sort -rn | head -20"

# Response code distribution
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | awk '{print \$9}' | sort | uniq -c | sort -rn"

# Top paths being hit
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | awk '{print \$7}' | sort | uniq -c | sort -rn | head -20"

# Requests per second (find peak)
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | grep -oP '\[\d{2}/\w{3}/\d{4}:\d{2}:\d{2}:\d{2}' | sort | uniq -c | sort -rn | head -10"

# Check user agents (empty UA = bot)
ssh don@$SERVER_IP 'docker exec traefik cat /var/log/traefik/access.log | grep -c "\"\" \"-\""'

# Unique IPs
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | awk '{print \$1}' | sort -u | wc -l"

# Which backend services are being hit
ssh don@$SERVER_IP "docker exec traefik cat /var/log/traefik/access.log | grep -oP '\"[^\"]*@docker\"' | sort | uniq -c | sort -rn"
```

### CrowdSec

```bash
# View active bans
ssh don@$SERVER_IP "docker exec crowdsec cscli decisions list"

# View alerts
ssh don@$SERVER_IP "docker exec crowdsec cscli alerts list -l 20"

# Inspect a specific alert
ssh don@$SERVER_IP "docker exec crowdsec cscli alerts inspect <ID>"

# Manually ban an IP
ssh don@$SERVER_IP "docker exec crowdsec cscli decisions add --ip <IP> --duration 24h --reason 'manual DDoS ban'"

# Remove a ban (e.g. self-ban or false positive)
ssh don@$SERVER_IP "docker exec crowdsec cscli decisions delete --ip <IP>"
```

## Current Cloudflare Rules

### coding.global (4/5 custom rules used)

| # | Name | Action | Expression |
|---|------|--------|------------|
| 1 | Allow IndexNow | skip | `http.request.uri.path eq "/fa52fd419e4c203cf499dabb0beaa1fe.txt"` |
| 2 | Block DDoS patterns | block | `(http.request.uri.query contains "cb=") or (http.x_forwarded_for contains ",") or (http.user_agent eq "") or (http.request.method eq "HEAD" and not cf.client.bot)` |
| 3 | Challenge high threat score | managed_challenge | `(not cf.client.bot and cf.threat_score gt 10)` |
| 4 | Challenge datacenter traffic | managed_challenge | `(not cf.client.bot and ip.src.asnum in {13238 14061 15169 16276 24940 32934 36352 45102 55286 63949 197540 39351})` |

### unorouter.ai (4/5 custom rules used)

| # | Name | Action | Expression |
|---|------|--------|------------|
| 1 | Skip WAF for API gateway | skip | `(http.host eq "api.unorouter.ai")` |
| 2 | Block DDoS patterns | block | Same as coding.global rule 2 |
| 3 | Challenge high threat score | managed_challenge | Same as coding.global rule 3 |
| 4 | Challenge datacenter traffic | managed_challenge | Same as coding.global rule 4 |

### Rate Limiting (1/1 used, both zones)

100 requests per 10 seconds per IP per colo, action: block.

### Other Settings (both zones)

- Security level: high
- Bot Fight Mode: enabled
- Crawler protection: enabled
- DDoS L7 override: block at low sensitivity (most aggressive)

## How to Update Rules During an Attack

### 1. Identify the attack pattern

Look at the latest logs for what is different:

```bash
# What do the attack requests look like?
ssh don@$SERVER_IP "docker exec traefik tail -200 /var/log/traefik/access.log | grep ' 499 ' | tail -10"
```

Common attack fingerprints to look for:
- Query params (`?cb=`, `?_cb=`, `?ts=`, `?rnd=`, `?cachebypass=`)
- Empty user agent (`"-"`)
- Double IPs in the source (`1.2.3.4,5.6.7.8`)
- Random garbage paths (`/HZvoJf`, `/fuckyyou`)
- Specific HTTP method or version
- All hitting one backend service
- 499 flood (fire and forget pattern)

### 2. Update the block rule

```bash
# Get the ruleset and rule IDs
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_CG/rulesets" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  | jq '.result[] | select(.phase == "http_request_firewall_custom") | .id'

# Get rule IDs within the ruleset
RULESET_ID="<from above>"
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_CG/rulesets/$RULESET_ID" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  | jq '.result.rules[] | {id, description, expression}'

# Update a rule (PATCH)
RULE_ID="<from above>"
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_CG/rulesets/$RULESET_ID/rules/$RULE_ID" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "description": "Block DDoS patterns",
    "action": "block",
    "expression": "<updated expression here>",
    "enabled": true
  }'
```

### 3. Emergency: enable Under Attack Mode

Note: UAM adds a JS challenge that blocks API clients. Do NOT enable on unorouter.ai.

```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_CG/settings/security_level" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  -H "Content-Type: application/json" \
  --data '{"value":"under_attack"}'
```

Disable when attack stops:

```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_CG/settings/security_level" \
  -H "X-Auth-Email: $EMAIL" -H "X-Auth-Key: $KEY" \
  -H "Content-Type: application/json" \
  --data '{"value":"high"}'
```

## Attack Analysis: March 4, 2026

### Overall Scale

| Metric | Value |
|---|---|
| Total requests (at origin) | 2,076,384 |
| Total unique IPs | 479,514 |
| Duration | 14:15 to 19:02 UTC (~5 hours) |
| Peak req/sec | 18,952 |
| Target | coding-global-web (96.7% of all traffic) |
| Botnet signature | 100% empty UA, 100% HTTP/2.0, IPv4+IPv6 |
| Avg requests per IP | ~4.3 (highly distributed, per-IP limits useless) |

### Response Code Breakdown

| Code | Count | % | Meaning |
|---|---|---|---|
| 499 | 1,327,555 | 63.9% | Client disconnected (botnet fire and forget) |
| 429 | 286,889 | 13.8% | Rate limited by inflight-limit |
| 307 | 282,156 | 13.6% | Redirect (reached backend) |
| 403 | 80,006 | 3.9% | Blocked by CrowdSec |
| 404 | 58,762 | 2.8% | Not found (reached backend) |
| 504 | 16,587 | 0.8% | Gateway timeout (responseHeaderTimeout=15s) |
| 200 | 12,067 | 0.6% | Success (reached backend) |
| 502 | 6,183 | 0.3% | Bad gateway |
| 503 | 3,162 | 0.2% | Circuit breaker tripped |
| 500 | 2,412 | 0.1% | Internal server error |

### Wave 1: Empty UA Volumetric Flood (14:15 to 17:59 UTC)

- **~1.2M requests** targeting `/` and `/en`
- **479K+ unique IPs**, avg ~4.3 requests per IP
- Peak burst: **192K req/min** at 16:26, **18,952 req/sec**
- 79% resulted in 499 (fire and forget, client disconnects immediately)
- No query params, pure volumetric flood
- Timeline: initial burst at 14:23, lull until 16:20, massive spike 16:20 to 16:28
- Blocked by: `http.user_agent eq ""` at Cloudflare edge (millions blocked, thousands overflow)

### Wave 2: /fuckyyou Cache Buster (18:00 to 19:00 UTC)

- **868K requests** targeting `/fuckyyou` and `/en/fuckyyou`
- **237K unique IPs**
- Mixed methods: GET 67.6%, POST 22.7%, HEAD 9.7%
- Cache-busting query params: `ts`, `param1`, `v`, `rnd`, `cachebypass`, `_`
- Average response time: **6.6 seconds** (many tied up backend for 5 to 60s)
- 33% caught by rate limiting (429), 7.5% by CrowdSec (403)
- Blocked by: empty UA rule at edge + inflight-limit + circuit-breaker at origin

### Earlier Waves (from previous sessions)

**Wave 2 (earlier):** Cache buster with spoofed IPs. Params: `?_=<timestamp>&r=<random>&_cb=<random>`. Double IPs in XFF. Blocked by: `contains "_cb="` and `x_forwarded_for contains ","`

**Wave 3 (earlier):** Adapted cache buster. Changed `_cb=` to `cb=`, added random garbage paths. Blocked by: `contains "cb="`

## Lessons Learned

### Conntrack exhaustion is the silent killer
The conntrack table at default 262K filled up during the attack (247K/262K). When full, the kernel drops ALL new TCP connections for every service on the server. This was the root cause of unorouter.ai going down during a coding.global attack. Fixed by increasing to 2M via the `sysctl-init` container.

### Per-IP rate limiting is useless against distributed botnets
With 479K unique IPs each sending ~4 requests, per-IP limits never trigger. The inflight-limit must be **global** (total concurrent requests to a service) to have any effect.

### The 499 fire-and-forget signature
63.9% of attack requests resulted in 499 (client disconnected). The botnet sends the request and immediately drops the connection. The backend still begins processing, consuming resources. The `responseHeaderTimeout=15s` prevents these from hanging for 125+ seconds.

### CrowdSec false-positive cascade
During the attack, mass 403 responses triggered CrowdSec's `http-probing` scenario, which started banning legitimate residential IPs caught in the crossfire. Watch for this and clear false bans after attacks.

### Backend response time cascade
Once one backend starts responding slowly (125s during attack), Traefik holds all connections open, consuming goroutines and starving other services. The circuit-breaker (trips at >5s median latency) and `responseHeaderTimeout=15s` prevent this cascade.

## Kernel Conntrack Tuning

During large DDoS attacks (18K+ req/sec), the Linux conntrack table fills up and the kernel drops ALL new connections, taking down every site on the server. The `sysctl-init` container in docker-compose sets these params at startup:

| Parameter | Value | Why |
|-----------|-------|-----|
| `nf_conntrack_max` | 2,097,152 | Default 262K fills up during attacks |
| `nf_conntrack_buckets` | 262,144 | Hash table size for lookup performance |
| `nf_conntrack_tcp_timeout_time_wait` | 30s | Default 120s, faster cleanup of closed connections |
| `nf_conntrack_tcp_timeout_established` | 300s | Default 432000s (5 days), frees stale entries |
| `nf_conntrack_tcp_timeout_close_wait` | 10s | Default 60s, faster cleanup |

### Check conntrack during an attack

```bash
# Current usage vs max
ssh don@$SERVER_IP "cat /proc/sys/net/netfilter/nf_conntrack_count && echo '/' && cat /proc/sys/net/netfilter/nf_conntrack_max"

# Emergency increase (if sysctl-init hasn't run)
sudo sysctl -w net.netfilter.nf_conntrack_max=2097152
```

## Traefik Protection Middlewares

Defined in `dynamic/protection.yml`, applied per-service via Docker labels.

### inflight-limit
- Caps total concurrent requests to a service at 100 (global, not per-IP)
- Returns 429 immediately when exceeded
- Prevents one overwhelmed backend from consuming all Traefik connections

### circuit-breaker
- Trips when median response time > 5s, or 30% 5xx, or 10% network errors
- Returns 503 immediately for 15s, then probes for recovery
- Prevents request queuing from cascading to other services

### Backend timeouts (static config)
- `responseHeaderTimeout=15s`: Kills backend connections with no response after 15s
- `readTimeout=30s` / `writeTimeout=30s`: Entrypoint level connection timeouts
- `dialTimeout=5s`: Backend must accept connection within 5s

### Apply to a service

Add to the service's Docker labels:
```yaml
- "traefik.http.routers.<name>.middlewares=inflight-limit@file,circuit-breaker@file"
```

## Container Resource Limits

| Container | Memory | CPU |
|-----------|--------|-----|
| traefik | 16GB | 6 cores |
| crowdsec | no limit | 0.5 cores |
| coding-global-web | 2GB | no limit |

## CrowdSec Server IP Whitelist

The server IP and admin VPN IP are whitelisted in CrowdSec to prevent banning from internal traffic and false positives. Whitelist config: `crowdsec/server-whitelist.yaml`, mounted to both parser (`s02-enrich`) and postoverflow (`s01-whitelist`) stages.

## Useful Cloudflare Expression Fields (Free Plan)

| Field | Description |
|-------|-------------|
| `http.request.uri.path` | Request path |
| `http.request.uri.query` | Query string |
| `http.user_agent` | User agent header |
| `http.x_forwarded_for` | X-Forwarded-For header |
| `cf.client.bot` | Known good bot (Google, Bing etc.) |
| `cf.threat_score` | Cloudflare threat score (0-100, higher = worse) |
| `ip.src` | Source IP |
| `ip.src.asnum` | Source ASN number |
| `ip.src.country` | Source country code |
| `http.request.method` | HTTP method (GET, POST etc.) |
| `http.request.version` | HTTP version |
| `ssl` | Whether request used HTTPS |
| `http.host` | Hostname (useful for multi-domain skip rules) |

**Not available on free plan:** `matches` (regex), `cf.bot_management.score`, WAF attack score.

## Datacenter ASN List

Hosting/VPN providers commonly used by botnets:

| ASN | Provider |
|-----|----------|
| 13238 | Yandex |
| 14061 | DigitalOcean |
| 15169 | Google Cloud |
| 16276 | OVH |
| 24940 | Hetzner |
| 32934 | Facebook/Meta |
| 36352 | ColoCrossing |
| 45102 | Alibaba |
| 55286 | Tatar Telecom |
| 63949 | Linode/Akamai |
| 197540 | netcup |
| 39351 | 31173 Services |

## Free Plan Limits

- 5 custom WAF rules (active + inactive count)
- 1 rate limiting rule
- No regex (`matches` operator)
- No `managed_challenge` in rate limiting (block only)
- Rate limit mitigation timeout fixed at 10 seconds
- No AI bot protection
- No WAF attack score
- DDoS L7 managed rules: full access, customizable
