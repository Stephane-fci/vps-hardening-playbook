# Step 8: Kernel Hardening

---

## What

Apply sysctl parameters that harden the kernel against network attacks, memory exploits, and information leaks. Hide /proc between users. Disable IPv6 if not needed.

## Why

The kernel is the foundation everything runs on. These params disable features that are useful for routers but dangerous for servers (IP forwarding, ICMP redirects), protect against common attacks (SYN floods, IP spoofing), and limit information leaks (kernel addresses, dmesg). None of these break normal server operation.

## How

### 8.1 — Create hardening config

```bash
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# VPS Hardening Playbook — Kernel hardening

# --- Network ---
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# --- Kernel ---
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 1
kernel.yama.ptrace_scope = 2

# --- Memory ---
vm.swappiness = 10
vm.overcommit_memory = 0

# --- IPv6 (disable if not using) ---
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

### 8.2 — Apply

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
sysctl --system
```

### 8.3 — Create boot persistence service

**🪤 TRAP: Some sysctl params get silently overridden after boot** by network drivers or other services. A config file isn't enough — you need a service that re-applies after network is up.

```bash
cat > /etc/systemd/system/sysctl-hardening.service << 'EOF'
[Unit]
Description=Re-apply sysctl hardening after boot
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/sysctl --system
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable sysctl-hardening.service
```

### 8.4 — Hide /proc between users

Prevents one user from seeing another's processes, command lines, and environment variables:

```bash
mount -o remount,hidepid=2 /proc
echo "proc /proc proc defaults,hidepid=2 0 0" >> /etc/fstab
```

### 8.5 — Disable IPv6 in UFW

```bash
sed -i 's/^IPV6=yes/IPV6=no/' /etc/default/ufw
ufw reload
```

## Verify

**Check RUNTIME values, not just config files:**

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
sysctl net.ipv4.ip_forward                    # Should be 0
sysctl net.ipv4.conf.all.accept_redirects     # Should be 0
sysctl net.ipv4.tcp_syncookies                # Should be 1
sysctl net.ipv4.conf.all.log_martians         # Should be 1
sysctl kernel.randomize_va_space              # Should be 2
sysctl kernel.dmesg_restrict                  # Should be 1
sysctl vm.swappiness                          # Should be 10
sysctl net.ipv6.conf.all.disable_ipv6         # Should be 1
mount | grep "hidepid"                        # Should show hidepid=2 or hidepid=invisible
grep "^IPV6" /etc/default/ufw                 # Should show IPV6=no
```

**⚠️ If any runtime value doesn't match the config, the boot service will fix it on next reboot. But apply manually now:** `sysctl -w [param]=[value]`

---

## Checklist

- [ ] `/etc/sysctl.d/99-hardening.conf` created
- [ ] `sysctl --system` applied
- [ ] Boot persistence service enabled
- [ ] /proc hidden (hidepid=2) + in fstab
- [ ] IPv6 disabled in sysctl and UFW
- [ ] **All runtime values verified** (not just config)
