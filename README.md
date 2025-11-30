# Proxy Server

Caddy-based reverse proxy for Spring and React services.

## Architecture

```
User → Caddy (80/443)
         │
         ├─ /api/*    → Spring (spring:8080)
         └─ /*        → React (mohe-react-dev:3000)
```

## Access URLs

### Local Network Access
- **HTTP**: `http://192.168.219.134/` → React Frontend
- **HTTPS**: `https://192.168.219.134/` → React Frontend (self-signed cert)
- **API**: `http://192.168.219.134/api/*` → Spring Backend

### External Access (Requires Port Forwarding)
- **HTTP**: `http://211.241.95.238/` → React Frontend
- **HTTPS**: `https://211.241.95.238/` → React Frontend (self-signed cert)
- **API**: `http://211.241.95.238/api/*` → Spring Backend

### Direct Container Access (Testing Only)
- **Spring**: `http://192.168.219.134:8000` → Spring Backend (container port 8080)
- **React**: `http://192.168.219.134:3000` → React Frontend (container port 3000)

## Routing Rules

### Path-based Routing
- **API Backend**: `/api/*` → Spring Backend (spring:8080)
- **Web App**: `/*` → React Frontend (mohe-react-dev:3000)
- **Ports**: 80 (HTTP), 443 (HTTPS)

## Configuration

### SSL/HTTPS
- **Self-Signed Certificate**: Enabled by default with `local_certs`
- **Certificate Location**: `/data/caddy/pki/authorities/local/`
- **Browser Warning**: You'll see a security warning due to self-signed cert
  - Click "Advanced" → "Proceed to site" to access

### Network Setup
- **Local Network**: 192.168.219.0/24
- **Local IP**: 192.168.219.134
- **External IP**: 211.241.95.238
- **Proxy Network**: proxy_network (bridge)
- **Volumes**: /data, /config for persistence

### Port Forwarding (For External Access)

To access from outside your local network (211.241.95.238), configure port forwarding on your router:

1. **Router Settings** → Port Forwarding
2. Add the following rules:
   - **HTTP**: External Port 80 → Internal IP 192.168.219.134:80
   - **HTTPS**: External Port 443 → Internal IP 192.168.219.134:443
3. Save and restart router

Without port forwarding, only local network access (192.168.219.x) will work.

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
