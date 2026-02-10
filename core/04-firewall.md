# Step 4: Firewall (UFW)

---

## What

Configure UFW (Uncomplicated Firewall) to block all incoming traffic by default, then explicitly allow only what's needed.

## Why

The "VPS bro" analogy: it's an elimination diet. Block everything, then slowly reintroduce what you actually need. Without a firewall, every port is exposed to the internet — and you might not even know what's listening.

## How

### 4.1 — Check current state

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
ufw status verbose
```

**🪤 TRAP: UFW might already be active** with rules you didn't set. Check what's there before changing anything. If rules exist, understand them first.

### 4.2 — Set defaults

```bash
ufw default deny incoming
ufw default allow outgoing
```

### 4.3 — Allow SSH

```bash
# Allow SSH on your custom port (from Step 3)
ufw allow 2222/tcp comment "SSH"
```

If you plan to do Tailscale lockdown (Step 6), this is temporary — you'll restrict it further later.

### 4.4 — Allow Tailscale (if using)

```bash
ufw allow in on tailscale0 comment "Tailscale VPN"
```

### 4.5 — Allow public services

Only if needed (ask your human which services stay public):

```bash
# Web server (if running nginx/apache)
ufw allow 80/tcp comment "HTTP"
ufw allow 443/tcp comment "HTTPS"

# Custom ports (only if specifically needed)
# ufw allow [PORT]/tcp comment "[DESCRIPTION]"
```

### 4.6 — Enable UFW

```bash
ufw enable
```

### 4.7 — Clean up old rules

Remove any rules that shouldn't be there (e.g., old port 22 if you changed SSH):
```bash
ufw delete allow 22/tcp
```

## Verify

```bash
ufw status verbose
```

Check:
- Default: deny (incoming), allow (outgoing)
- Only your explicitly allowed ports are listed
- No unexpected rules from before

**Runtime check — are only expected ports actually reachable?**
```bash
ss -tlnp | grep -E "0.0.0.0|:::"
```

Any service binding to `0.0.0.0` or `:::` is listening on all interfaces. If UFW is active, it will block connections to ports not in the allow list. But services should ideally bind to `127.0.0.1` if they don't need public access (see Step 7).

## Traps

**🪤 TRAP: UFW might already be active.** Don't assume a fresh state. Always check `ufw status` first.

**🪤 TRAP: Enabling UFW before adding SSH rule = locked out.** Always `ufw allow [SSH_PORT]` BEFORE `ufw enable`.

---

## Checklist

- [ ] Checked existing UFW state
- [ ] Default deny incoming, allow outgoing
- [ ] SSH port allowed
- [ ] Tailscale interface allowed (if applicable)
- [ ] Public services allowed (80/443 if web server)
- [ ] UFW enabled
- [ ] Old unnecessary rules removed
- [ ] `ufw status verbose` shows only expected rules
