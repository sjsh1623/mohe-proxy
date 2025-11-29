# Proxy Server

Caddy-based reverse proxy for Spring and React services.

## Architecture

```
User → Caddy (80/443/8080/3000) → Spring (spring:8000)
                                 → React (react:3000)
```

## Configuration

- **Spring**: Port 8080 → spring:8000
- **React**: Port 3000 → react:3000
- **SSL**: Let's Encrypt auto-renewal (when enabled)
- **Volumes**: /data, /config for persistence

## Usage

### Start Proxy Server
```bash
docker compose up -d
```

### Stop Proxy Server
```bash
docker compose down
```

### Restart Proxy Server
```bash
docker compose restart
```

### View Logs
```bash
docker compose logs -f caddy
```

### Rebuild
```bash
docker compose down
docker compose up -d --build
```

## Network Setup

Backend services (Spring, React) must be connected to `proxy_network`:

```yaml
# In your backend docker-compose.yml
networks:
  proxy_network:
    external: true
    name: proxy_network
```

## Files

- `docker-compose.yml` - Caddy container configuration
- `Caddyfile` - Reverse proxy rules
- `README.md` - Documentation
