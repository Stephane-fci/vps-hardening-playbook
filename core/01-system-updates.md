# Step 1: System Updates

---

## What

Update all packages, install security tools, configure automatic security updates.

## Why

A VPS is under attack the moment it gets an IP address. SSH brute-force bots start hitting port 22 within seconds. Unpatched vulnerabilities are the easiest entry point. Automatic security updates mean you're patched even when no one is watching.

## How

### 1.1 — Update everything

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
apt update && DEBIAN_FRONTEND=noninteractive apt upgrade -y
```

⚠️ This may take a few minutes. If a kernel update is included, a reboot will be needed eventually (we handle this with auto-reboot config below).

### 1.2 — Install security tools

```bash
apt install -y ufw fail2ban ca-certificates gnupg curl wget
```

### 1.3 — Configure automatic security updates

Check if unattended-upgrades is installed:
```bash
dpkg -l | grep unattended-upgrades
```

If not installed: `apt install -y unattended-upgrades`

Configure auto-reboot for kernel updates (use a drop-in override, not the main file):
```bash
cat > /etc/apt/apt.conf.d/51unattended-upgrades-custom << 'EOF'
// Auto-reboot when kernel updates require it
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
EOF
```

**Why a drop-in file?** The main config (`50unattended-upgrades`) gets overwritten by package updates. A separate file (`51-*`) survives. Higher number = applied later = overrides.

**Why 4AM UTC?** Low-traffic window. Adjust to your timezone if needed.

## Verify

```bash
# Updates applied
apt list --upgradable 2>/dev/null | wc -l
# Should show 1 (just the header line) or very few

# Security tools installed
dpkg -l ufw fail2ban | grep "^ii"
# Should show both

# Auto-reboot configured
cat /etc/apt/apt.conf.d/51unattended-upgrades-custom
# Should show Automatic-Reboot "true"
```

## Traps

**🪤 TRAP: fail2ban may need a kick.** On some Ubuntu installs, fail2ban is installed but not enabled or active. Always verify both:
```bash
systemctl is-active fail2ban    # Should say "active"
systemctl is-enabled fail2ban   # Should say "enabled"
```
If either says no: `systemctl enable --now fail2ban`

---

## Checklist

- [ ] apt update + upgrade complete
- [ ] ufw and fail2ban installed
- [ ] unattended-upgrades configured
- [ ] Auto-reboot configured (drop-in override)
- [ ] fail2ban is active AND enabled
