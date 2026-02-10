# Step 6: Tailscale Lockdown

🔴 **HIGH RISK.** After this step, public SSH is gone. Only Tailscale SSH works.

**Prerequisites:**
- Tailscale installed and connected on the server
- Tailscale installed on any machine that needs SSH access
- Backup operator can reach the server via Tailscale
- You know the Tailscale IP (`tailscale ip -4`)

**If you don't use Tailscale:** Skip this step. Your SSH is already protected by custom port + key-only + fail2ban. That's good enough for most deployments.

---

## What

Remove public SSH access from the firewall. Allow SSH only through the Tailscale VPN interface. After this, the server is only reachable via your private VPN mesh.

## Why

Even with key-only auth and a custom port, SSH is technically reachable from any IP on the internet. Tailscale lockdown means the server is invisible to the public internet — SSH requests from outside the VPN mesh are dropped entirely. Zero attack surface.

## How

### 6.1 — Verify Tailscale is working

```bash
tailscale status
tailscale ip -4
```

You should see your server and any connected peers. Note the Tailscale IP.

### 6.2 — Verify SSH works via Tailscale

```bash
# From another machine on the Tailscale network:
ssh admin@[TAILSCALE_IP] -p 2222
```

**Do NOT proceed until this works.** If Tailscale SSH doesn't work, fixing it after you've killed public SSH requires Hetzner Console.

### 6.3 — Remove public SSH, keep Tailscale SSH

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"

# Remove the public SSH rule
ufw delete allow 2222/tcp

# Allow SSH only on Tailscale interface
ufw allow in on tailscale0 to any port 2222 proto tcp comment "SSH via Tailscale only"
```

### 6.4 — Verify the lockdown

```bash
ufw status verbose
```

SSH should only appear with `on tailscale0`, not as a general allow.

```bash
# Test: SSH via Tailscale should work
ssh admin@[TAILSCALE_IP] -p 2222

# Test: SSH via public IP should be refused
ssh admin@[PUBLIC_IP] -p 2222
# Should time out or refuse connection
```

**Have your backup operator verify they can still connect via Tailscale.**

## Rollback

If Tailscale breaks and you're locked out:

1. Access via provider console (Hetzner Console / web VNC)
2. Re-add public SSH: `ufw allow 2222/tcp`
3. Fix Tailscale, then re-lock

---

## Checklist

- [ ] Tailscale running and connected
- [ ] SSH works via Tailscale IP
- [ ] Backup operator confirmed they can SSH via Tailscale
- [ ] Public SSH rule removed from UFW
- [ ] SSH only allowed on tailscale0 interface
- [ ] Public SSH confirmed refused
- [ ] Tailscale SSH confirmed working
