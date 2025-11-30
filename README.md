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
- **HTTP**: `http://192.168.219.100/` → React Frontend
- **HTTPS**: `https://192.168.219.100/` → React Frontend (self-signed cert)
- **API**: `http://192.168.219.100/api/*` → Spring Backend

### External Access (Port Forwarding Configured)
- **HTTP**: `http://182.210.216.60/` → React Frontend
- **HTTPS**: `https://182.210.216.60/` → React Frontend (self-signed cert)
- **API**: `http://182.210.216.60/api/*` → Spring Backend

### Direct Container Access (Testing Only)
- **Spring**: `http://192.168.219.100:8000` → Spring Backend (container port 8080)
- **React**: `http://192.168.219.100:3000` → React Frontend (container port 3000)

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
- **Local IP**: 192.168.219.100 (current Mac IP)
- **External IP**: 182.210.216.60 (LG U+ / Xpeed)
- **Gateway**: 192.168.219.1
- **ISP**: LG U+ (Xpeed)
- **Proxy Network**: proxy_network (bridge)
- **Volumes**: /data, /config for persistence

**Note**: External IP may change if your ISP uses dynamic IP. Check current IP with:
```bash
curl ifconfig.me
```

### Port Forwarding (Configured ✅)

Port forwarding has been configured on the router:

**Current Configuration:**
- **HTTP**: External Port 80 → Internal IP 192.168.219.100:80
- **HTTPS**: External Port 443 → Internal IP 192.168.219.100:443

**Router**: 192.168.219.1
**External Access**: http://182.210.216.60/ (working ✅)

If external IP changes or access stops working:
1. Check current external IP: `curl ifconfig.me`
2. Verify local IP hasn't changed: `ifconfig en0 | grep "inet "`
3. Update port forwarding if local IP changed

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
