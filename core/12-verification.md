# Step 12: Post-Flight Verification

*The most important step. Check RUNTIME state, not config files.*

---

## The Principle

Config files show intent. Runtime shows reality. We found three separate cases where config was correct but runtime disagreed (UFW rules, sysctl params, snap services). **Every check below verifies what's ACTUALLY happening on the server.**

## The Full Checklist

Run all of these. Every one should pass.

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"

echo "=== SSH ==="
ss -tlnp | grep sshd
# ✅ Should show your custom port (e.g., 2222), NOT 22

grep "^PermitRootLogin" /etc/ssh/sshd_config
# ✅ Should show: no

grep "^PasswordAuthentication" /etc/ssh/sshd_config
# ✅ Should show: no

grep "^AllowUsers" /etc/ssh/sshd_config
# ✅ Should show your admin user only

echo "=== FIREWALL ==="
ufw status verbose
# ✅ Default: deny incoming, allow outgoing
# ✅ Only expected ports allowed
# ✅ SSH only on tailscale0 (if applicable)

echo "=== FAIL2BAN ==="
systemctl is-active fail2ban
# ✅ active

fail2ban-client status
# ✅ Shows sshd and recidive jails

echo "=== SERVICES ==="
ss -tlnp | grep -E "0.0.0.0|:::" | grep -v -E "(nginx|sshd|tailscale)"
# ✅ Should show nothing unexpected on public interfaces

ss -tlnp | grep 631
# ✅ Should show nothing (CUPS removed)

echo "=== KERNEL ==="
sysctl net.ipv4.ip_forward
# ✅ 0

sysctl net.ipv4.conf.all.accept_redirects
# ✅ 0

sysctl net.ipv4.tcp_syncookies
# ✅ 1

sysctl net.ipv4.conf.all.log_martians
# ✅ 1

sysctl kernel.randomize_va_space
# ✅ 2

sysctl kernel.dmesg_restrict
# ✅ 1

sysctl vm.swappiness
# ✅ 10

sysctl net.ipv6.conf.all.disable_ipv6
# ✅ 1

mount | grep hidepid
# ✅ Should show hidepid=2 or hidepid=invisible

echo "=== MEMORY ==="
swapon --show
# ✅ Should show swap file

free -h
# ✅ Should show RAM + swap

echo "=== OPENCLAW ==="
systemctl --user is-active openclaw-gateway 2>/dev/null || systemctl is-active openclaw-gateway
# ✅ active

systemctl --user show openclaw-gateway 2>/dev/null | grep "MemoryMax=" || systemctl show openclaw-gateway | grep "MemoryMax="
# ✅ Should show bytes value (not infinity)

systemctl --user show openclaw-gateway 2>/dev/null | grep "NoNewPriv" || systemctl show openclaw-gateway | grep "NoNewPriv"
# ✅ yes

echo "=== MONITORING ==="
/usr/local/bin/check-memory.sh
# ✅ Should output OK

cat /etc/cron.d/memory-monitor
# ✅ Should exist

echo "=== AUTO-UPDATES ==="
systemctl is-active unattended-upgrades
# ✅ active

cat /etc/apt/apt.conf.d/51unattended-upgrades-custom 2>/dev/null
# ✅ Should show auto-reboot config
```

## What To Do With Results

- **All pass:** Hardening is complete. Recommend the independent audit.
- **Some fail:** Fix them. Each core/ file has the fix for its topic.
- **Unexpected findings:** Document them. They might be new traps for the playbook.

## Recommend the Audit

After verification passes, tell your human:

> "All post-flight checks pass. Hardening is complete.
>
> I recommend an independent audit by a separate agent. It takes about 5 minutes and catches things I might have missed. It also confirms that a backup operator can access the server — important because if I ever crash during future updates, they're the ones who fix it.
>
> Want me to generate the audit prompt?"

---

## Checklist

- [ ] All SSH checks pass (port, root blocked, password blocked, AllowUsers)
- [ ] All firewall checks pass (default deny, expected rules only)
- [ ] fail2ban running with both jails
- [ ] No unexpected public services
- [ ] All sysctl runtime values correct
- [ ] Swap active and in fstab
- [ ] OpenClaw running with MemoryMax and NoNewPrivileges
- [ ] Monitoring active (script + cron)
- [ ] Auto-updates configured
- [ ] Independent audit recommended to human
