# Step 10: systemd Service Hardening

⚠️ **This step requires restarting the agent service.** If you're an agent hardening your own server, this restarts YOU.

**Tell your human before proceeding:**
> "I need to restart myself to apply memory limits. If I'm not back in 60 seconds, use Claude Code or Cowork to check on me. The commands they need: `systemctl --user status openclaw-gateway` and `systemctl --user restart openclaw-gateway` (or the equivalent system service)."

---

## What

Add memory limits and basic security directives to the OpenClaw (or other agent) systemd service file.

## Why

Without memory limits, a runaway agent (too many sub-agents, memory leak, heavy sessions) can consume all RAM and trigger the OOM killer, which kills the process without saving state. MemoryMax tells systemd to constrain the process. MemoryHigh applies back-pressure before the hard limit — giving the agent a chance to compact or clean up.

## How

### 10.1 — Find the service file

For OpenClaw user service:
```bash
cat ~/.config/systemd/user/openclaw-gateway.service
```

For a system service:
```bash
systemctl cat openclaw-gateway
```

### 10.2 — Add hardening directives

Add these to the `[Service]` section:

```ini
# Memory: hard cap at ~80% of total RAM
MemoryMax=6G
# Memory: soft limit (applies pressure before hard cap)
MemoryHigh=5G
# Prevent privilege escalation
NoNewPrivileges=yes
```

Adjust MemoryMax based on your server's RAM:
- 4GB RAM → MemoryMax=3G, MemoryHigh=2.5G
- 8GB RAM → MemoryMax=6G, MemoryHigh=5G
- 16GB RAM → MemoryMax=12G, MemoryHigh=10G

### 10.3 — What about ProtectSystem, PrivateTmp?

**🪤 TRAP: systemd user services have limited sandboxing.**

| Directive | User service | System service |
|-----------|-------------|---------------|
| MemoryMax / MemoryHigh | ✅ Works | ✅ Works |
| NoNewPrivileges | ✅ Works | ✅ Works |
| ProtectSystem=strict | ❌ Fails silently | ✅ Works |
| PrivateTmp=yes | ❌ Fails silently | ✅ Works |
| ReadWritePaths | ❌ No effect | ✅ Works |

If OpenClaw runs as a user service (which is the default), only MemoryMax, MemoryHigh, and NoNewPrivileges will work. For full sandboxing, you'd need to migrate to a system service — that's a larger project.

### 10.4 — Apply and restart

```bash
# For user service:
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway

# For system service:
systemctl daemon-reload
systemctl restart openclaw-gateway
```

**Wait 30-60 seconds, then verify.**

## Verify

```bash
# Service running
systemctl --user is-active openclaw-gateway  # or systemctl for system service

# Memory limits
systemctl --user show openclaw-gateway | grep -E "MemoryMax=|MemoryHigh="
# MemoryMax should show bytes (e.g., 6442450944 = 6GB)

# NoNewPrivileges
systemctl --user show openclaw-gateway | grep NoNewPriv
# Should show yes

# Current memory usage
ps -p $(pgrep -f openclaw-gatewa) -o pid,rss --no-headers
# RSS in KB — should be well below MemoryMax
```

---

## Checklist

- [ ] Human warned about restart
- [ ] Backup operator ready (if applicable)
- [ ] MemoryMax and MemoryHigh added
- [ ] NoNewPrivileges=yes added
- [ ] daemon-reload done
- [ ] Service restarted
- [ ] Service came back healthy
- [ ] Memory limits confirmed at runtime
