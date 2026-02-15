# Step 3: SSH Hardening

⚠️ **This step changes how you access the server. One mistake can lock you out.**

**Safety rule:** Keep your current SSH session open. Test the new config from a NEW session. Only close the old session after confirming the new one works.

---

## What

Disable root login, disable password authentication, change the default port, restrict access to your admin user only.

## Why

SSH is the #1 target on any server. Automated bots scan port 22 and try common passwords within seconds of a VPS going live. Disabling root + passwords + using a custom port eliminates the vast majority of attacks. It won't stop a determined, targeted attacker, but it stops 99.9% of the noise.

## How

### 3.1 — Backup the current config

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

**This backup is your lifeline.** If you're locked out, restoring this file (via Hetzner Console or backup operator) brings back the original SSH config.

### 3.2 — Check for SSH socket activation

```bash
systemctl is-active ssh.socket
```

**🪤 TRAP: Ubuntu 24.04 SSH Socket Activation.**
If this returns "active", the `Port` directive in sshd_config is **IGNORED**. Ubuntu 24.04 uses `ssh.socket` to manage the listening port, and any port change must be done via a systemd socket override instead of sshd_config.

If socket activation is active, create the override:
```bash
mkdir -p /etc/systemd/system/ssh.socket.d
cat > /etc/systemd/system/ssh.socket.d/port.conf << 'EOF'
[Socket]
ListenStream=
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
EOF
```

The empty `ListenStream=` clears the inherited default. Without it, the old port stays active.

If socket activation is NOT active (older Ubuntu, other distros), the `Port` directive in sshd_config works normally.

### 3.3 — Apply hardened SSH config

Append to `/etc/ssh/sshd_config`:

```bash
cat >> /etc/ssh/sshd_config << 'EOF'

# ============================================
# HARDENED SSH CONFIG
# ============================================
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers admin
X11Forwarding no
PermitEmptyPasswords no
EOF
```

Replace `admin` with your actual admin username. Replace `2222` with your chosen port.

Comment out any conflicting earlier values in the file:
```bash
sed -i 's/^PermitRootLogin yes/# PermitRootLogin yes  # DISABLED/' /etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication yes/# PasswordAuthentication yes  # DISABLED/' /etc/ssh/sshd_config
```

### 3.4 — Test config BEFORE restarting

```bash
sshd -t
```

If this shows ANY errors, **DO NOT restart SSH.** Fix the errors first. If you restart with a broken config, SSH may not come back.

### 3.5 — UPDATE FIREWALL FIRST (before restarting SSH!)

**🪤 TRAP: If UFW is active and only allows port 22, changing SSH to port 2222 and restarting will lock you out.** Update the firewall BEFORE restarting SSH:

```bash
ufw allow 2222/tcp comment "SSH new port"
# Don't delete the old rule yet — wait until new port is confirmed working
```

### 3.6 — Apply the changes

**🪤 TRAP: Ubuntu 24.04+ uses `ssh.service`, NOT `sshd.service`.** If you're used to `systemctl restart sshd` from CentOS/RHEL/older Ubuntu, it won't work (or will silently fail). Always use `ssh` on modern Ubuntu/Debian.

```bash
# If using socket activation (Ubuntu 24.04):
systemctl daemon-reload
systemctl restart ssh.socket
systemctl restart ssh

# If NOT using socket activation:
systemctl restart ssh
```

**Note:** On CentOS/RHEL/Fedora, the service IS `sshd.service`. This trap is Ubuntu/Debian-specific.

### 3.6 — Verify from a NEW session

**Keep your current session open!**

From another terminal (or ask your human to test):
```bash
ssh admin@[SERVER_IP] -p 2222
```

Only close the old session after confirming the new one works.

## Verify

```bash
# Port
ss -tlnp | grep sshd
# Should show port 2222 (or your chosen port)

# Config
grep -E "^(PermitRoot|PasswordAuth|AllowUsers|Port)" /etc/ssh/sshd_config
# Should show: PermitRootLogin no, PasswordAuthentication no, AllowUsers admin, Port 2222

# Root rejected
ssh root@[SERVER_IP] -p 2222
# Should fail with "Permission denied"
```

### 3.7 — Tune MaxStartups for agent workloads (optional)

If your agent does bulk file transfers over SSH (e.g., deploying workspace files to another VPS), the default `MaxStartups 10:30:60` may throttle connections and cause "connection refused" errors during bursts.

For agent workloads:
```bash
# In /etc/ssh/sshd_config, add or update:
MaxStartups 30:50:100
MaxSessions 10
```

Then restart SSH. This allows more concurrent connection attempts before rate-limiting kicks in.

**Default (10:30:60):** Start rejecting 30% of connections after 10 unauthenticated, hard limit at 60.
**Agent-tuned (30:50:100):** Start rejecting 50% after 30 unauthenticated, hard limit at 100.

Only needed if you're doing multi-file transfers or running multiple parallel SSH commands. For single-agent personal use, the defaults are fine.

## Rollback

If locked out:
1. Access via provider console (Hetzner Console, etc.)
2. Login as root (console bypasses SSH)
3. Restore: `cp /etc/ssh/sshd_config.backup /etc/ssh/sshd_config`
4. If socket override exists: `rm -rf /etc/systemd/system/ssh.socket.d`
5. `systemctl daemon-reload && systemctl restart ssh.socket ssh`

**⚠️ Hetzner Console limitation:** If you've disabled password authentication (which you should), you can't log in via the web console either — it uses a terminal emulator that requires password auth. Recovery options:
- **Hetzner Rescue Mode:** Boot into rescue, mount disk, edit `sshd_config` to re-enable PasswordAuthentication temporarily.
- **Backup operator user:** See `profiles/single-agent.md` for creating a local recovery account before lockdown.
- **Snapshot:** If you took a snapshot before hardening, restore it.

---

## Checklist

- [ ] Backup created (`sshd_config.backup`)
- [ ] Socket activation checked (and override created if needed)
- [ ] Hardened config appended
- [ ] Old conflicting values commented out
- [ ] `sshd -t` passed with no errors
- [ ] SSH restarted
- [ ] New session works on new port with admin user
- [ ] Root login rejected
- [ ] Password login rejected
- [ ] MaxStartups tuned (if agent workloads need it)
