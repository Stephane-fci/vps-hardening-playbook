# Step 7: Service Cleanup

---

## What

Find services that are unnecessarily exposed or running, fix their binding, and remove what doesn't belong.

## Why

Every listening port is attack surface. Services that bind to `0.0.0.0` (all interfaces) are reachable from the public internet even if your firewall blocks the port — because firewall rules can be misconfigured or temporarily disabled. Defense in depth means the service itself should only bind to localhost if it doesn't need public access.

## How

### 7.1 — Find all listening services

```bash
ss -tlnp | grep LISTEN
```

Look for anything on `0.0.0.0` or `:::` that shouldn't be public. Common offenders:

| Port | Service | Should it be public? |
|------|---------|---------------------|
| 22/2222 | SSH | Via Tailscale only (see Step 6) |
| 80/443 | nginx/apache | Yes, if serving a website |
| 631 | CUPS (printing) | NO — remove it |
| 3000-9999 | App backends | Usually NO — bind to 127.0.0.1 |

### 7.2 — Fix services that bind to all interfaces

If a service is behind a reverse proxy (nginx), it should bind to `127.0.0.1` (localhost only). The reverse proxy handles public access.

**Example: Node.js app on port 8080**

Find the server code and change:
```javascript
// BAD: publicly accessible
server.listen(8080)

// GOOD: only accessible via localhost (nginx proxies to it)
server.listen(8080, '127.0.0.1')
```

Restart the service, then verify:
```bash
ss -tlnp | grep 8080
# Should show 127.0.0.1:8080, NOT 0.0.0.0:8080
```

### 7.3 — Remove unnecessary services

**🪤 TRAP: Ubuntu ships snap packages with services you didn't ask for.**

Check for snaps:
```bash
snap list
```

Common unnecessary snaps on a VPS: `cups` (printing), `lxd` (containers if not using them).

Remove:
```bash
snap remove cups
snap remove lxd  # Only if not using LXD containers
```

**Important:** `systemctl disable cups` won't work for snap services. The unit names are different (`snap.cups.cupsd.service`). Use `snap remove` instead.

Also check for other unnecessary services:
```bash
systemctl list-units --type=service --state=running
```

Disable anything that has no business on a server (print services, Bluetooth, GUI-related services).

## Verify

```bash
# No unexpected public listeners
ss -tlnp | grep -E "0.0.0.0|:::" | grep -v -E "(nginx|sshd|tailscale)"
# Should show nothing (or only services you explicitly want public)

# No CUPS
ss -tlnp | grep 631
# Should show nothing

# Backend services on localhost only
ss -tlnp | grep -E "8080|3000|3456"
# Should all show 127.0.0.1
```

---

## Checklist

- [ ] All listening ports audited
- [ ] Backend services bound to 127.0.0.1 (not 0.0.0.0)
- [ ] Unnecessary snaps removed (cups, lxd, etc.)
- [ ] Unnecessary services disabled
- [ ] No unexpected public listeners remain
