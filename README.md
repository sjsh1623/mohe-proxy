# Proxy Server

Caddy-based reverse proxy for Spring and React services.

## Architecture

```
User → Caddy (80/443/8080/3000)
         │
         ├─ /api/*    → Spring (spring:8000)
         ├─ /*        → React (react:3000)
         ├─ :8080     → Spring (direct)
         └─ :3000     → React (direct)
```

## Routing Rules

### Path-based Routing (Port 80)
- **API Backend**: `http://your-ip/api/*` → Spring Backend
- **Web App**: `http://your-ip/*` → React Frontend

### Direct Port Access (Development)
- **Spring**: `http://your-ip:8080` → Spring Backend
- **React**: `http://your-ip:3000` → React Frontend

## Configuration

- **SSL**: Let's Encrypt auto-renewal (disabled by default, set `auto_https on`)
- **Volumes**: /data, /config for persistence
- **Network**: proxy_network (bridge)

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
