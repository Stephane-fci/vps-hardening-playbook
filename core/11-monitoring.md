# Step 11: Monitoring & Logging

---

## What

Set up memory monitoring, configure log size limits, and prepare for ongoing health checks.

## Why

Hardening is not a one-time event. The server needs ongoing monitoring to catch problems before they become crises. The OOM kill on Feb 9 happened because nobody was watching memory usage. A 15-minute cron check would have flagged it hours before.

## How

### 11.1 — Memory monitoring script

```bash
cat > /usr/local/bin/check-memory.sh << 'SCRIPT'
#!/bin/bash
# Memory monitoring — alerts when thresholds exceeded
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"

RAM_TOTAL=$(free -m | awk '/^Mem:/{print $2}')
RAM_USED=$(free -m | awk '/^Mem:/{print $3}')
RAM_PCT=$((RAM_USED * 100 / RAM_TOTAL))

SWAP_TOTAL=$(free -m | awk '/^Swap:/{print $2}')
SWAP_USED=$(free -m | awk '/^Swap:/{print $3}')
if [ "$SWAP_TOTAL" -gt 0 ]; then
    SWAP_PCT=$((SWAP_USED * 100 / SWAP_TOTAL))
else
    SWAP_PCT=0
fi

ALERT=""
if [ "$RAM_PCT" -gt 80 ]; then
    ALERT="RAM at ${RAM_PCT}% (${RAM_USED}/${RAM_TOTAL}MB)"
fi
if [ "$SWAP_PCT" -gt 50 ]; then
    ALERT="${ALERT} SWAP at ${SWAP_PCT}% (${SWAP_USED}/${SWAP_TOTAL}MB)"
fi

if [ -n "$ALERT" ]; then
    echo "ALERT: $ALERT"
    exit 1
else
    echo "OK: RAM ${RAM_PCT}% SWAP ${SWAP_PCT}%"
    exit 0
fi
SCRIPT
chmod +x /usr/local/bin/check-memory.sh
```

### 11.2 — Cron job

```bash
cat > /etc/cron.d/memory-monitor << 'EOF'
# Memory monitoring — check every 15 minutes
*/15 * * * * root /usr/local/bin/check-memory.sh > /dev/null 2>&1 || logger -t memory-alert "$(/usr/local/bin/check-memory.sh 2>&1)"
EOF
```

This logs alerts to syslog (`/var/log/syslog`). If the agent has Discord or other messaging capability, it can be wired to send alerts there too.

### 11.3 — Journald size limit

```bash
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/size-limit.conf << 'EOF'
[Journal]
SystemMaxUse=500M
RuntimeMaxUse=100M
EOF

systemctl restart systemd-journald
```

**Why a drop-in file?** Same reason as auto-updates: the main config gets overwritten by package updates. Drop-in overrides survive.

### 11.4 — Check disk usage

```bash
df -h /
```

If above 80%, investigate what's using space: `du -sh /* | sort -rh | head -10`

## Verify

```bash
# Monitoring script works
/usr/local/bin/check-memory.sh
# Should output OK or ALERT

# Cron job exists
cat /etc/cron.d/memory-monitor

# Journald limited
journalctl --disk-usage
# Should be under 500MB

# Disk healthy
df -h / | awk 'NR==2{print $5}'
# Should be under 80%
```

---

## Checklist

- [ ] Memory monitoring script created and executable
- [ ] Cron job running every 15 minutes
- [ ] Journald size limited (drop-in override)
- [ ] Disk usage checked
