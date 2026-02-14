# Backup Operator Setup Prompt

*Give this to Claude Code (or any backup agent) BEFORE hardening begins. Fill in the blanks, then paste the section below the line.*

---

# VPS Backup Operator — Setup & Standing Orders

## Your Role

You are the backup operator for a VPS that is about to be hardened. Your job:

1. **Verify you can SSH in** (right now, before anything changes)
2. **Stay available** during hardening in case the primary agent locks itself out
3. **Run an independent audit** after hardening is complete
4. **Be the emergency recovery path** if the primary agent crashes or gets locked out

## Server Access

- **IP:** [FILL: server IP]
- **SSH User:** [FILL: admin username — will be created during hardening if it doesn't exist yet]
- **SSH Port:** [FILL: current port, likely 22 — will change to 2222 during hardening]
- **SSH Key:** [FILL: path to key or "already in authorized_keys"]

**Connect now to verify:**
```bash
ssh [USER]@[IP] -p [PORT]
```

**After hardening, the connection will change to:**
```bash
ssh [USER]@[IP] -p 2222
```

If Tailscale is being set up, you may also need the Tailscale IP:
```bash
ssh [USER]@[TAILSCALE_IP] -p 2222
```

## What Will Change During Hardening

The primary agent will:
1. Create a non-root admin user (if not exists)
2. Change SSH to port 2222
3. Disable root login and password auth
4. Enable UFW firewall (only port 2222 + any web ports)
5. Install fail2ban
6. Add swap, kernel hardening, memory limits
7. Possibly restart the OpenClaw service (which restarts the primary agent)

**High-risk moments** (when you might be needed):
- After SSH port change → primary agent might lose connection
- After UFW enable → wrong rules could lock everyone out
- After OpenClaw service restart → primary agent goes offline temporarily
- After Tailscale lockdown → SSH only via VPN mesh

## Emergency Recovery Commands

If the primary agent goes offline and the human asks for help:

**Check if the server is reachable:**
```bash
nc -z -w5 [IP] 2222 && echo "Port 2222 open" || echo "Port 2222 blocked"
nc -z -w5 [IP] 22 && echo "Port 22 still open" || echo "Port 22 closed"
```

**If SSH works but the agent is down:**
```bash
# Check service status (try both — depends on setup)
systemctl --user status openclaw-gateway
systemctl status openclaw-gateway

# Restart the agent
systemctl --user restart openclaw-gateway
systemctl restart openclaw-gateway

# Check logs
journalctl --user -u openclaw-gateway --since "5 min ago" --no-pager
journalctl -u openclaw-gateway --since "5 min ago" --no-pager
```

**If locked out (can't SSH):**
- Ask the human to use Hetzner Console (web VNC) or provider's rescue mode
- From console: `ufw allow 2222/tcp` or `ufw disable` to restore access

**If UFW is blocking everything:**
```bash
# From Hetzner Console (not SSH):
ufw status
ufw allow 2222/tcp
ufw allow 22/tcp   # temporary fallback
```

## After Hardening — Audit

Once the primary agent reports hardening is complete, run the full audit.

**Audit protocol:** https://github.com/Stephane-fci/vps-hardening-playbook/blob/main/audit/AUDIT-PROTOCOL.md

Quick version — check these runtime values:
```bash
# SSH
ss -tlnp | grep sshd
grep "^PermitRootLogin" /etc/ssh/sshd_config
grep "^PasswordAuthentication" /etc/ssh/sshd_config

# Firewall
sudo ufw status verbose

# fail2ban
sudo fail2ban-client status

# Swap
swapon --show
sysctl vm.swappiness

# Kernel (RUNTIME, not config)
sysctl net.ipv4.conf.all.log_martians
sysctl net.ipv6.conf.all.disable_ipv6
sysctl kernel.randomize_va_space

# Services
sudo ss -tlnp | grep LISTEN

# OpenClaw hardening
systemctl show openclaw-gateway 2>/dev/null | grep -E "MemoryMax=|NoNewPriv" || \
systemctl --user show openclaw-gateway 2>/dev/null | grep -E "MemoryMax=|NoNewPriv"
```

**Report format:**
- **Confirmed working:** [list]
- **Issues found:** [severity + details + fix]
- **Can't check:** [anything blocked by permissions]

---

*Replace all [FILL: ...] sections before giving this to the backup operator.*
