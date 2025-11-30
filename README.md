# Proxy Server

Caddy-based reverse proxy for Spring and React services.

## Architecture

```
User → Caddy (80/443)
         │
         ├─ /api/*    → Spring (spring:8080)
         └─ /*        → React (mohe-react-dev:3000)
```

## Routing Rules

### Path-based Routing (Port 80)
- **API Backend**: `http://your-ip/api/*` → Spring Backend (spring:8080)
- **Web App**: `http://your-ip/*` → React Frontend (mohe-react-dev:3000)

### Direct Port Access (For Testing)
- **Spring**: `http://your-ip:8000` → Spring Backend (container port 8080)
- **React**: `http://your-ip:3000` → React Frontend (container port 3000)

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
