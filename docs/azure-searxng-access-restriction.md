# SearXNG Access Restriction on Azure

## Overview

SearXNG is deployed as an Azure Container App and is restricted to only accept requests from the NextChat Web App.

## Architecture

```
User Browser → NextChat Web App → SearXNG Container App
                (proxy)           (restricted access)
```

- **NextChat**: Azure Web App (`Next-Chat` in `DefaultResourceGroup-EA`)
- **SearXNG**: Azure Container App (`searxng` in `searxng-rg`)
- **SearXNG URL**: `https://searxng.yellowmushroom-1df91506.eastasia.azurecontainerapps.io`

## Access Restriction Method

Using Azure Container App's IP-based access restrictions to only allow NextChat's outbound IPs.

### Why IP Restriction?

- Simple to configure
- No code changes required
- Built-in Azure feature

## Configuration Commands

### View NextChat's Outbound IPs

```bash
# Current outbound IPs
az webapp show -n Next-Chat -g DefaultResourceGroup-EA --query "outboundIpAddresses" -o tsv

# All possible outbound IPs (important for stability)
az webapp show -n Next-Chat -g DefaultResourceGroup-EA --query "possibleOutboundIpAddresses" -o tsv
```

### View Current Access Restrictions

```bash
az containerapp ingress access-restriction list -n searxng -g searxng-rg -o table
```

### Add an IP to Allow List

```bash
az containerapp ingress access-restriction set \
  -n searxng \
  -g searxng-rg \
  --rule-name "allow-nextchat-1" \
  --ip-address "x.x.x.x/32" \
  --action Allow \
  --description "NextChat IP"
```

### Remove an Access Restriction

```bash
az containerapp ingress access-restriction remove \
  -n searxng \
  -g searxng-rg \
  --rule-name "allow-nextchat-1"
```

## Current Configuration (as of 2026-01-21)

25 IP addresses are allowed (7 current + 18 possible):

| Type | Count | Description |
|------|-------|-------------|
| `outboundIpAddresses` | 7 | Currently used by NextChat |
| `possibleOutboundIpAddresses` | 18 | May be used after scaling/restart |

### IP List

```
# outboundIpAddresses (current)
20.24.236.196
20.24.237.10
20.24.237.207
20.24.238.146
20.24.238.160
20.24.238.252
20.205.69.85

# Additional possibleOutboundIpAddresses
20.24.239.204
20.24.239.40
20.24.239.86
20.255.192.101
20.255.192.197
20.255.192.225
20.255.192.88
20.255.193.115
20.255.193.149
20.255.193.180
20.255.193.203
20.255.193.215
20.255.193.239
20.255.193.245
20.255.193.63
20.255.194.113
20.255.194.166
20.255.194.47
```

## Testing

### Test Direct Access (Should Fail)

```bash
curl -s -w "%{http_code}" "https://searxng.yellowmushroom-1df91506.eastasia.azurecontainerapps.io/search?q=test&format=json"
# Expected: 403 (RBAC: access denied)
```

### Test via NextChat Proxy (Should Work)

```bash
curl -s -w "%{http_code}" "https://nextchat.siyuan0.tech/api/proxy/search?q=test&format=json" \
  -H "X-Base-URL: https://searxng.yellowmushroom-1df91506.eastasia.azurecontainerapps.io"
# Expected: 200
```

## Maintenance

### When to Update IP Restrictions

- **App Service Plan change** (e.g., Basic → Premium): `possibleOutboundIpAddresses` will change completely
- **Region change**: New set of IPs

### Script to Refresh All IPs

```bash
#!/bin/bash
# Get all possible IPs
ips=$(az webapp show -n Next-Chat -g DefaultResourceGroup-EA --query "possibleOutboundIpAddresses" -o tsv | tr ',' ' ')

# Add each IP
idx=1
for ip in $ips; do
  az containerapp ingress access-restriction set \
    -n searxng \
    -g searxng-rg \
    --rule-name "allow-nextchat-$idx" \
    --ip-address "$ip/32" \
    --action Allow \
    --description "NextChat IP" \
    --only-show-errors
  idx=$((idx+1))
done
```

## Alternative Approaches (Not Implemented)

1. **Azure Private Endpoint / VNet Integration**
   - More secure (private network)
   - More complex setup
   - Additional cost

2. **API Key / Header Authentication**
   - Add authentication layer to SearXNG
   - Requires code/config changes

3. **Azure Front Door with WAF**
   - Enterprise-grade solution
   - Higher cost

## References

- [Azure Container Apps IP Restrictions](https://learn.microsoft.com/en-us/azure/container-apps/ip-restrictions)
- [Azure Web App Outbound IPs](https://learn.microsoft.com/en-us/azure/app-service/overview-inbound-outbound-ips)
