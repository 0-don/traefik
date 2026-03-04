# DDoS Protection Playbook

## Architecture

```
Client -> Cloudflare Edge (WAF rules, DDoS L7) -> Traefik (CrowdSec bouncer) -> Backend
```

Cloudflare blocks at the edge. CrowdSec catches what slips through. Backend containers have memory limits so one site going down does not take the server with it.

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

# Check for cache buster params
ssh don@$SERVER_IP "docker exec traefik tail -100 /var/log/traefik/access.log | grep 'cb='"
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

# Remove a ban (e.g. self-ban)
ssh don@$SERVER_IP "docker exec crowdsec cscli decisions delete --ip <IP>"
```

## Current Cloudflare Rules (Free Plan: 5 custom, 1 rate limit)

### Custom Rules (4/5 used)

| # | Name | Action | Expression |
|---|------|--------|------------|
| 1 | Allow IndexNow | skip | `http.request.uri.path eq "/fa52fd419e4c203cf499dabb0beaa1fe.txt"` |
| 2 | Block DDoS patterns | block | `(http.request.uri.query contains "cb=") or (http.x_forwarded_for contains ",") or (http.user_agent eq "") or (http.request.method eq "HEAD" and not cf.client.bot)` |
| 3 | Challenge high threat score | managed_challenge | `(not cf.client.bot and cf.threat_score gt 10)` |
| 4 | Challenge datacenter traffic | managed_challenge | `(not cf.client.bot and ip.src.asnum in {13238 14061 15169 16276 24940 32934 36352 45102 55286 63949 197540 39351})` |

### Rate Limiting (1/1 used)

100 requests per 10 seconds per IP per colo, action: block.

### Other Settings

- Security level: high
- Bot Fight Mode: enabled
- Crawler protection: enabled
- DDoS L7 override: block at default sensitivity

## How to Update Rules During an Attack

### 1. Identify the attack pattern

Look at the latest logs for what is different:

```bash
# What do the attack requests look like?
ssh don@$SERVER_IP "docker exec traefik tail -200 /var/log/traefik/access.log | grep ' 499 ' | tail -10"
```

Common attack fingerprints to look for:
- Query params (`?cb=`, `?_cb=`, random strings)
- Empty user agent (`"-"`)
- Double IPs in the source (`1.2.3.4,5.6.7.8`)
- Random garbage paths (`/HZvoJf`, `/z4uYwR`)
- Specific HTTP method or version
- All hitting one backend service

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

## Known Attack Patterns (March 2026)

### Wave 1: Empty user agent botnet
- 199K+ unique IPs, GET `/` and `/en`
- Empty user agent, HTTP/2.0, 499 responses
- Peak: 179K req/min, multiple waves over hours
- Blocked by: `http.user_agent eq ""`

### Wave 2: Cache buster with spoofed IPs
- Query params: `?_=<timestamp>&r=<random>&_cb=<random>`
- Double IPs in X-Forwarded-For: `9.218.248.68,189.50.33.120`
- Blocked by: `contains "_cb="` and `x_forwarded_for contains ","`

### Wave 3: Adapted cache buster
- Changed `_cb=` to `cb=` (dropped underscore)
- Added random garbage paths: `/HZvoJf?cb=XYZ`
- Blocked by: `contains "cb="` (catches both variants)

### Wave 4: Garbage path flood
- 105K+ unique IPs, `/fuckyyou` with legit-looking params (`?utm_medium=`, `?rnd=`, `?rev=`)
- Mixed GET/POST/HEAD, empty user agent
- Peak: 205K req/min, ran simultaneously with Wave 1
- Blocked by: empty UA rule + HEAD block

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

**Not available on free plan:** `matches` (regex), `cf.bot_management.score`, WAF attack score.

## Datacenter ASN List

These are hosting/VPN providers commonly used by botnets:

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

## Container Resource Limits

Traefik and backend containers have memory limits to prevent one service from crashing the entire server:

| Container | Memory | CPU |
|-----------|--------|-----|
| traefik | 16GB | 4 cores |
| crowdsec | no limit | 0.5 cores |
| coding-global-web | 2GB | no limit |

## CrowdSec Server IP Whitelist

The server IP is whitelisted in CrowdSec to prevent self-banning from internal API traffic. Whitelist config: `/etc/crowdsec/parsers/s02-enrich/server-whitelist.yaml`
