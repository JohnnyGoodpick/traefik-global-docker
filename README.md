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

The dashboard is bound to **localhost only** (`127.0.0.1:9090`) with basic auth. It is not exposed to the public internet.

### Remote access via Tailscale (recommended)

[Tailscale](https://tailscale.com) creates a private mesh network (tailnet) between your devices. Install it on your servers and personal devices to securely access admin interfaces without public exposure.

```
┌─────────────────────────────────────────────────────────┐
│                     Your Tailnet                        │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│  │  Laptop  │   │  Phone   │   │  Tablet  │  ← Clients │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘            │
│       │              │              │                   │
│       └──────────────┼──────────────┘                   │
│                      │                                  │
│         ┌────────────┴────────────┐                     │
│         │                         │                     │
│         ▼                         ▼                     │
│   ┌───────────┐            ┌───────────┐               │
│   │  Preprod  │            │   Prod    │   ← Servers   │
│   │   :9090   │            │   :9090   │               │
│   └───────────┘            └───────────┘               │
│    preprod.tailnet          prod.tailnet               │
└─────────────────────────────────────────────────────────┘
```

#### Setup

1. **Install Tailscale on each VPS**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```

2. **Install Tailscale on your devices**
   - macOS/Windows/Linux: [tailscale.com/download](https://tailscale.com/download)
   - iOS/Android: App Store / Play Store

3. **Get your server's Tailscale hostname**
   ```bash
   tailscale status
   # Example output:
   # 100.x.y.z   preprod   linux   -
   ```

4. **Access the dashboard** from any device on your tailnet
   ```
   http://preprod.your-tailnet.ts.net:9090
   http://prod.your-tailnet.ts.net:9090
   ```

The dashboard requires basic auth even over Tailscale (defense in depth).

#### Why Tailscale over public exposure?

| Public (with auth) | Tailscale |
|--------------------|-----------|
| Visible to port scanners | Invisible to internet |
| Vulnerable to brute force | No public endpoint |
| Zero-day risk | Identity-based access |
| Just a password | Device + user authentication |

#### ACLs: Team access control (optional)

By default, all devices on your tailnet can access all other devices. For solo use, this is fine.

For teams, Tailscale ACLs let you restrict who can access what. ACLs are configured in the Tailscale admin console: **Access Controls** → `https://login.tailscale.com/admin/acls`

```
┌─────────────────────────────────────────┐
│              Your Tailnet               │
│                                         │
│  You (admin)─────────┬─► Prod :9090 ✓   │
│                      └─► Preprod :9090 ✓│
│                                         │
│  Contractor ─────────┬─► Prod :9090 ✗   │  ← ACL blocks
│                      └─► Preprod :9090 ✓│  ← ACL allows
└─────────────────────────────────────────┘
```

**Example ACL configuration:**

```jsonc
{
  // Define user groups
  "groups": {
    "group:admin": ["you@email.com"],           // Full access
    "group:contractors": ["contractor@email.com"] // Limited access
  },

  // Define who can manage server tags
  "tagOwners": {
    "tag:prod": ["group:admin"],
    "tag:preprod": ["group:admin"]
  },

  // Access rules (evaluated top to bottom)
  "acls": [
    {
      // Admins can access everything on all servers
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    {
      // Contractors can only access preprod dashboard
      "action": "accept",
      "src": ["group:contractors"],
      "dst": ["tag:preprod:9090"]
    }
    // Implicit deny: contractors cannot reach prod
  ]
}
```

**Apply tags to your servers:**

```bash
# On preprod server
sudo tailscale up --advertise-tags=tag:preprod

# On prod server
sudo tailscale up --advertise-tags=tag:prod
```

| Scenario | ACLs needed? |
|----------|--------------|
| Solo developer | No |
| Small trusted team | No |
| Team with contractors | Yes |
| Compliance requirements | Yes |

### Local access (alternative)

```bash
# On the server directly
curl -u admin:yourpassword http://localhost:9090/api/overview

# Via SSH tunnel (if Tailscale not available)
ssh -L 9090:localhost:9090 user@your-vps
# Then visit http://localhost:9090 in your browser
```

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
