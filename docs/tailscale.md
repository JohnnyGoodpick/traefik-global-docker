# Tailscale Setup for Dashboard Access

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

## Setup

1. **Install Tailscale on each VPS**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```

2. **Bind dashboard to Tailscale IP** (in `.env`)
   ```bash
   # Get your Tailscale IP
   tailscale ip -4
   # Example output: 100.x.y.z

   # Add to .env
   DASHBOARD_BIND=100.x.y.z

   # Restart Traefik
   docker compose up -d --force-recreate traefik
   ```

3. **Install Tailscale on your devices**
   - macOS/Windows/Linux: [tailscale.com/download](https://tailscale.com/download)
   - iOS/Android: App Store / Play Store

4. **Get your server's Tailscale hostname**
   ```bash
   tailscale status
   # Example output:
   # 100.x.y.z   preprod   linux   -
   ```

5. **Access the dashboard** from any device on your tailnet
   ```
   http://preprod.your-tailnet.ts.net:9090
   http://prod.your-tailnet.ts.net:9090
   ```

The dashboard requires basic auth even over Tailscale (defense in depth).

## Why Tailscale over public exposure?

| Public (with auth) | Tailscale |
|--------------------|-----------|
| Visible to port scanners | Invisible to internet |
| Vulnerable to brute force | No public endpoint |
| Zero-day risk | Identity-based access |
| Just a password | Device + user authentication |

## ACLs: Team access control (optional)

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

### How ACLs work

- **Joining the tailnet** = getting through the gate (controlled via Tailscale admin console invites)
- **ACLs** = which rooms you can enter once inside

Without ACLs, everyone inside can go everywhere. With ACLs, you control movement within.

### Example ACL configuration

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

### Apply tags to your servers

```bash
# On preprod server
sudo tailscale up --advertise-tags=tag:preprod

# On prod server
sudo tailscale up --advertise-tags=tag:prod
```

### When do you need ACLs?

| Scenario | ACLs needed? |
|----------|--------------|
| Solo developer | No |
| Small trusted team | No |
| Team with contractors | Yes |
| Compliance requirements | Yes |
