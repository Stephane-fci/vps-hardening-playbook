# Independent Audit Protocol

*You're a separate agent (Claude Code, Cowork, or another tool) verifying someone else's hardening work.*

---

## Your Role

You are the independent verifier. You did NOT do the hardening. You SSH in fresh and check whether everything is actually secure. Your job is to find what the executor missed.

The executor may have also given you a filled-in audit prompt (from `AUDIT-PROMPT-TEMPLATE.md`) with specific details about what was changed. Read that first if available.

## Why This Matters

1. **Catch mistakes.** The executor is checking their own homework. You're the teacher.
2. **Verify runtime, not config.** The #1 lesson from this playbook: config files can lie. Check what's actually running.
3. **Confirm backup access.** Your ability to SSH in proves that a backup operator can reach the server. This matters because hardening steps (especially systemd restarts) can crash the primary agent, and someone needs to be able to fix it.

## What To Check

SSH into the server and run each check. Report anything that doesn't match expectations.

### SSH
```bash
ss -tlnp | grep sshd
# Expected: custom port (e.g., 2222), NOT port 22

grep "^PermitRootLogin" /etc/ssh/sshd_config
# Expected: no

grep "^PasswordAuthentication" /etc/ssh/sshd_config
# Expected: no

grep "^AllowUsers" /etc/ssh/sshd_config
# Expected: specific admin user, not empty/missing
```

Try logging in as root: `ssh root@[IP] -p [PORT]` — should be rejected.

### Firewall
```bash
sudo ufw status verbose
# Expected: active, default deny incoming, specific allow rules only
# Check: is SSH restricted to Tailscale? Are only needed ports open?
```

### fail2ban
```bash
sudo systemctl is-active fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
# Expected: active, sshd jail on correct port, recidive jail exists
```

### Services
```bash
sudo ss -tlnp | grep LISTEN
# Check: anything on 0.0.0.0 that shouldn't be public?
# Check: CUPS (port 631) should NOT be present
# Check: any unexpected services?
```

### Kernel (RUNTIME values)
```bash
sysctl net.ipv4.ip_forward                    # Expected: 0
sysctl net.ipv4.conf.all.accept_redirects     # Expected: 0
sysctl net.ipv4.tcp_syncookies                # Expected: 1
sysctl net.ipv4.conf.all.log_martians         # Expected: 1
sysctl kernel.randomize_va_space              # Expected: 2
sysctl kernel.dmesg_restrict                  # Expected: 1
sysctl vm.swappiness                          # Expected: 10
sysctl net.ipv6.conf.all.disable_ipv6         # Expected: 1
mount | grep hidepid                          # Expected: hidepid present
```

⚠️ **Check RUNTIME values, not config files.** This is where the executor most commonly has blind spots.

### OpenClaw / Agent Service
```bash
# Find the service
systemctl --user status openclaw-gateway 2>/dev/null || systemctl status openclaw-gateway

# Memory limits
systemctl --user show openclaw-gateway 2>/dev/null | grep -E "MemoryMax|MemoryHigh|NoNewPriv" || \
systemctl show openclaw-gateway | grep -E "MemoryMax|MemoryHigh|NoNewPriv"
# Expected: MemoryMax set (not infinity), NoNewPrivileges=yes
```

### Swap & Memory
```bash
swapon --show             # Expected: swap file present
free -h                   # Expected: reasonable usage
```

### Monitoring
```bash
/usr/local/bin/check-memory.sh    # Expected: exits 0 (OK)
cat /etc/cron.d/memory-monitor    # Expected: exists, runs every 15 min
```

### Auto-updates
```bash
systemctl is-active unattended-upgrades
cat /etc/apt/apt.conf.d/51unattended-upgrades-custom 2>/dev/null || \
grep "Automatic-Reboot" /etc/apt/apt.conf.d/50unattended-upgrades
```

### Bonus: check for overrides in unexpected places
```bash
# Drop-in directories (the executor might have used these correctly, but check)
ls /etc/systemd/system/ssh.socket.d/ 2>/dev/null
ls /etc/systemd/journald.conf.d/ 2>/dev/null
ls /etc/apt/apt.conf.d/51* 2>/dev/null
```

## How To Report

Structure your findings as:

```
AUDIT RESULTS — [date]

Confirmed working:
- [Item]: [details]

Issues found:
1. [Issue title] (severity: critical/important/minor)
   What: [what you found]
   Expected: [what it should be]
   Fix: [suggested fix]

2. ...
```

Send this back to the executor or the human. The executor should fix all critical/important issues and re-verify.

---

*Remember: your goal is not to find fault. It's to make the server actually secure. A clean audit is a great result.*
