# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Caddy-based reverse proxy server that routes traffic to Spring Boot backend and React frontend services running in separate Docker containers. The proxy handles:
- Domain-based routing to mohe.today
- Path-based routing (/api/* → Spring, /image/* → Image processor, /* → React)
- Automatic HTTPS with Let's Encrypt
- CORS headers for image processing service
- Security headers (HSTS, X-Frame-Options, etc.)

## Architecture

```
Internet → Caddy (proxy-caddy container)
              │
              ├─ mohe.today/health   → Health check (returns "OK")
              ├─ mohe.today/api/*    → spring:8080 (external network)
              ├─ mohe.today/image/*  → moheimageprocessor-app-1:5200 (external network)
              └─ mohe.today/*        → mohe-react-dev:3000 (external network)
```

**Critical Architecture Notes:**
- Caddy runs in the `proxy_network` bridge network
- All backend services (Spring, React, Image processor) must also be connected to `proxy_network`
- Container names in Caddyfile must match actual running container names
- Internal container ports are used in reverse_proxy directives (not host ports)
- Direct IP access is blocked; only domain name access is allowed

## Common Commands

### Starting/Stopping the Proxy
```bash
# Start proxy server
docker compose up -d

# Stop proxy server
docker compose down

# Restart proxy server
docker compose restart

# View logs
docker compose logs -f caddy

# Rebuild after config changes
docker compose down
docker compose up -d --build
```

### Verifying Network Setup
```bash
# Check if proxy_network exists
docker network ls | grep proxy_network

# Inspect proxy network to see connected containers
docker network inspect proxy_network

# Verify backend containers are running and connected
docker ps | grep -E "spring|mohe-react-dev|moheimageprocessor"

# Test connectivity to backend services from Caddy
docker exec proxy-caddy wget -O- http://spring:8080/api/health
docker exec proxy-caddy wget -O- http://mohe-react-dev:3000
```

### Testing and Debugging
```bash
# Test external domain access
curl -I https://mohe.today

# Check Let's Encrypt certificate
docker exec proxy-caddy caddy list-certificates

# Validate Caddyfile syntax before applying
docker exec proxy-caddy caddy validate --config /etc/caddy/Caddyfile

# Reload Caddyfile without downtime
docker exec proxy-caddy caddy reload --config /etc/caddy/Caddyfile
```

## Configuration Files

### Caddyfile Structure
- **Global block**: Email for Let's Encrypt notifications
- **IP blocking block**: Blocks direct IP access (responds with 403)
- **Domain block (mohe.today)**: Main routing configuration
  - `handle /health`: Health check endpoint (returns "OK")
  - `handle /api/*`: Routes to Spring Boot backend
  - `handle /image/*`: Routes to image processor with CORS headers
    - Supports full filenames (URL-encoded): `/image/10000_파스토보이_월계점_1.jpg`
    - Supports ID-only lookup: `/image/10000` (finds matching file automatically)
    - Supports resizing: `/image/10000/400/300` (width x height)
  - `handle`: Default route to React frontend
  - Security headers and compression applied to all responses

### docker-compose.yml
- Single service: `caddy` (Caddy reverse proxy)
- Exposes ports 80 (HTTP) and 443 (HTTPS)
- Uses named volumes for persistence: `caddy_data`, `caddy_config`
- Connected to `proxy_network` bridge network

## Important Patterns

### Adding New Backend Services
When adding a new service to be proxied:

1. Ensure the service's docker-compose.yml connects to proxy_network:
```yaml
networks:
  proxy_network:
    external: true
    name: proxy_network
```

2. Add routing rule to Caddyfile using the container name and internal port:
```caddyfile
handle /newservice/* {
    reverse_proxy container-name:internal-port
}
```

3. Reload Caddy configuration:
```bash
docker exec proxy-caddy caddy reload --config /etc/caddy/Caddyfile
```

### Container Name Changes
If backend container names change, you must update the Caddyfile accordingly. Container names in `reverse_proxy` directives must match actual running containers, not image names or service names.

### CORS Configuration
The image processing service has CORS enabled. To add CORS to other routes, follow the pattern in lines 29-44 of Caddyfile:
- Handle OPTIONS preflight requests
- Add Access-Control-* headers to all responses

### SSL/HTTPS
- Let's Encrypt certificates are automatically obtained and renewed by Caddy
- Certificates are stored in the `caddy_data` volume
- HTTP automatically redirects to HTTPS
- Email in global block is used for Let's Encrypt notifications

## Troubleshooting

### Backend Services Not Reachable
1. Verify backend containers are running: `docker ps`
2. Check containers are on proxy_network: `docker network inspect proxy_network`
3. Verify container names match Caddyfile: compare `docker ps` output with reverse_proxy directives
4. Test connectivity from Caddy container: `docker exec proxy-caddy wget -O- http://container-name:port`

### SSL Certificate Issues
1. Check certificate status: `docker exec proxy-caddy caddy list-certificates`
2. Verify email is set in Caddyfile global block
3. Ensure ports 80 and 443 are accessible from internet (required for Let's Encrypt)
4. Check Caddy logs: `docker compose logs caddy`

### Configuration Changes Not Applied
After modifying Caddyfile, reload configuration:
```bash
docker exec proxy-caddy caddy reload --config /etc/caddy/Caddyfile
```

Or restart the container:
```bash
docker compose restart
```
