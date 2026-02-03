# Traefik Global Proxy

A hardened Traefik v3.6.7 reverse proxy for Docker container auto-discovery on a VPS.

```
                        ┌──────────────────────────────────────────────────────────────────┐
                        │                             VPS                                  │
                        │                                                                  │
┌──────────┐            │   ┌──────────────────────────────────────────────────────────┐   │
│          │ :80 / :443 │   │                    app-proxy network                     │   │
│ Internet │◄───────────┼──►│                                                          │   │
│          │            │   │   ┌───────────┐                                          │   │
└──────────┘            │   │   │  Traefik  │─── access.log                            │   │
 example.com            │   │   │  :80/:443 │─── acme.json (SSL certs)                 │   │
 mysite.io              │   │   └─────┬─────┘                                          │   │
 shop.org               │   │         │ routes by hostname                             │   │
 api.example.com        │   │         │                                                │   │
 admin.mysite.io        │   │   ┌─────┴──────────┬──────────────────┬───────────────┐  │   │
 ...                    │   │   │                │                  │               │  │   │
                        │   │   ▼                ▼                  ▼               ▼  │   │
                        │   │ ┌──────┐        ┌──────┐          ┌──────┐       ┌──────┐│   │
                        │   │ │App 1 │        │App 2 │          │App 3 │  ...  │App N ││   │
                        │   │ │Laravel│        │Laravel│          │Laravel│       │      ││   │
                        │   │ └──────┘        └──────┘          └──────┘       └──────┘│   │
                        │   │  example.com     mysite.io         shop.org              │   │
                        │   │  api.example.com admin.mysite.io   cdn.shop.org          │   │
                        │   │  blog.example.com                                        │   │
                        │   └──────────────────────────────────────────────────────────┘   │
                        │                             ▲                                    │
                        │   ┌─────────────────────────┴───────┐                            │
                        │   │         socket-proxy            │                            │
                        │   │     (filtered Docker API)       │                            │
                        │   └─────────────────────────────────┘                            │
                        │                             ▲                                    │
                        │                             │                                    │
                        │                  /var/run/docker.sock                            │
                        └──────────────────────────────────────────────────────────────────┘
```

## Quick Start

```bash
# 1. Create required files/directories
sudo touch acme.json && sudo chown root:root acme.json && sudo chmod 600 acme.json
mkdir -p logs

# 2. Generate dashboard password hash
# Run interactively - it will prompt for password twice
docker run --rm -it httpd:2-alpine htpasswd -nB admin
# Output: admin:$2y$05$xyz...
#
# Or non-interactive:
# docker run --rm httpd:2-alpine htpasswd -nbB admin 'yourpassword'

# 3. Create .env file
cp .env.example .env
# Set DASHBOARD_PASSWORD_HASH to the hash from step 2
# IMPORTANT: Double all $ signs ($ -> $$) for Docker Compose
# Example: $2y$05$xyz... becomes $$2y$$05$$xyz...

# 4. Open firewall for HTTP/3 (QUIC uses UDP)
sudo ufw allow 443/udp  # Ubuntu/Debian
# or: sudo firewall-cmd --add-port=443/udp --permanent && sudo firewall-cmd --reload  # RHEL/CentOS

# 5. Start Traefik (network auto-created)
docker compose up -d
```

## Production Deployment

### Recommended locations

| Location | Use case |
|----------|----------|
| `/opt/traefik/` | System-wide service (recommended) |
| `/srv/traefik/` | Service data focused |
| `/opt/docker/traefik/` | Multiple Docker projects |

### Setup

```bash
# Copy to production location
sudo mkdir -p /opt/traefik
sudo cp -r ./* /opt/traefik/
cd /opt/traefik

# Create ACME storage file (required before first run)
# Must be owned by root - Traefik runs as root inside the container
sudo touch acme.json && sudo chown root:root acme.json && sudo chmod 600 acme.json

# Configure environment
cp .env.example .env
# Edit .env with your values

# Start
docker compose up -d
```

## Features

- **HTTP/3 (QUIC)** - Faster connections, auto-negotiated with browsers
- **TLS 1.2+** - Modern encryption only
- **Gzip compression** - Smaller responses (excludes images/video)
- **Rate limiting** - 100 req/s average, 50 burst per IP
- **Security headers** - HSTS, X-Frame-Options, etc.
- **Auto SSL** - Let's Encrypt certificates
- **Access logging** - All requests in JSON format
- **Docker socket proxy** - Isolated Docker API access

## Verify HTTP/3

```bash
# Using curl (requires curl 7.88+ with HTTP/3 support)
curl -I --http3 https://yourdomain.com

# Or use online tool
# https://http3check.net
```

## Dashboard

Access at: `https://gateway.yourdomain.com`

## Adding Apps

Add these labels to any container:

```yaml
services:
  myapp:
    networks:
      - app-proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.example.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=myresolver"

networks:
  app-proxy:
    external: true
```

## Access Logs

All HTTP requests are logged to `./logs/access.log` in JSON format.

```bash
# View live access logs
tail -f logs/access.log | jq

# Filter by app/service
tail -f logs/access.log | jq 'select(.ServiceName == "myapp@docker")'

# Filter by status code (errors only)
tail -f logs/access.log | jq 'select(.DownstreamStatus >= 400)'
```

### Log Rotation

Install logrotate config to prevent disk filling:

```bash
# Edit logrotate.conf - update the path to match your install location
sudo cp logrotate.conf /etc/logrotate.d/traefik

# Test rotation (dry run)
sudo logrotate -d /etc/logrotate.d/traefik
```

Rotation: daily, keeps 14 days, compresses old logs.

## Troubleshooting

### `acme.json: permission denied`

The `acme.json` file must be owned by `root:root` with `600` permissions. Traefik runs as root inside the container, so a file owned by your host user will be inaccessible.

```bash
sudo chown root:root acme.json && sudo chmod 600 acme.json
docker compose up -d --force-recreate traefik
```

### Middleware "does not exist" errors

If you see errors like `middleware "rate-limit@docker" does not exist` or `middleware "security-headers@docker" does not exist`, the global middlewares defined as Traefik container labels were not loaded properly. A simple `restart` is not sufficient — you need a full recreate:

```bash
docker compose up -d --force-recreate traefik
```

### Certificate resolver errors

`Router uses a nonexistent certificate resolver` is typically a cascading effect of the `acme.json` permission issue above. Fix the file permissions first, then recreate the container.

## Files

```
traefik-global-docker/
├── compose.yaml     # Main Traefik configuration
├── .env             # Credentials (create from .env.example)
├── .env.example     # Template
├── acme.json        # SSL certificates (auto-managed)
├── logrotate.conf   # Log rotation config (install to /etc/logrotate.d/)
└── logs/
    └── access.log   # All HTTP requests (JSON)
```
