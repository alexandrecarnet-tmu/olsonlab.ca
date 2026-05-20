# Cloudflare IP Range Maintenance Guide

## Overview

The Caddy reverse proxy uses Cloudflare IP ranges to restrict access and ensure all traffic comes through Cloudflare's network. These IP ranges can change periodically, and it's important to keep them synchronized to avoid accidental lockouts.

## Update Process

### Manual Update

1. **Check current ranges** at https://www.cloudflare.com/en/ips/
2. **Update the file** at `caddy/snippets/CloudflareIPs` with the latest IPv4 and IPv6 ranges
3. **Reload Caddy** without downtime:
   ```bash
   docker compose exec caddy caddy reload
   ```

### Automated Update (Recommended)

Consider creating a scheduled job (cron) to:
1. Download latest ranges from Cloudflare's API endpoint
2. Compare against current configuration
3. Update `caddy/snippets/CloudflareIPs` if changes detected
4. Reload Caddy
5. Send a notification of any changes

Example script (to be created):
```bash
#!/bin/bash
# This could be run daily via cron
curl -s https://www.cloudflare.com/ips-v4 | while read ip; do echo "  $ip \"; done > /tmp/cf_ips_v4
curl -s https://www.cloudflare.com/ips-v6 | while read ip; do echo "  $ip \"; done > /tmp/cf_ips_v6
# Compare and update if different
# Reload Caddy
```

## Monitoring

### Signs of Outdated IP Ranges

- Users report connection errors that worked before
- Cloudflare traffic is blocked but shouldn't be
- Ghost admin becomes inaccessible
- Check Caddy logs: `docker compose logs caddy`

### Health Checks

After updating IP ranges:
```bash
# Test Cloudflare access works
curl -v https://your-domain.com

# Check that non-Cloudflare traffic is blocked (should get 403)
curl -v -H "CF-Connecting-IP: 1.1.1.1" https://your-domain.com

# View Caddy logs for any errors
docker compose logs -f caddy
```

## Response Behavior

**Before**: Non-Cloudflare traffic would receive a dropped connection (`abort`)  
**After**: Non-Cloudflare traffic receives an explicit `403 Forbidden` response

This change provides better debugging and clearer feedback to clients.

## File Locations

- **IP ranges**: `caddy/snippets/CloudflareIPs` - Contains documented IP list and update instructions
- **Guard logic**: `caddy/snippets/CloudflareGuard` - The reusable snippet that uses the IP ranges
- **Main config**: `caddy/Caddyfile` - References the guard snippet

## Rollback

If something goes wrong:
```bash
# Revert to previous working version (from git)
git checkout caddy/snippets/CloudflareIPs

# Reload Caddy
docker compose exec caddy caddy reload
```

## Testing New Ranges

Before deploying to production, test locally:
```bash
# Reload with dry-run
docker compose exec caddy caddy validate

# Gradual rollout: Update and monitor for errors
docker compose exec caddy caddy reload
docker compose logs -f caddy
```
