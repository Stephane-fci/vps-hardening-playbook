# Step 5: fail2ban

---

## What

Configure fail2ban to automatically ban IPs that fail SSH authentication repeatedly.

## Why

Even with key-only auth, bots will keep trying. fail2ban watches auth logs and temporarily bans IPs after too many failures. The recidive jail escalates: repeat offenders get banned for a week.

## How

### 5.1 — Verify fail2ban is running

```bash
systemctl is-active fail2ban
systemctl is-enabled fail2ban
```

Both should say "active" / "enabled". If not: `systemctl enable --now fail2ban`

### 5.2 — Create jail config

```bash
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600

[recidive]
enabled = true
logpath = /var/log/fail2ban.log
banaction = %(banaction_allports)s
bantime = 604800
findtime = 86400
maxretry = 5
EOF
```

Update `port = 2222` to match your SSH port from Step 3.

### 5.3 — Restart and verify

```bash
systemctl restart fail2ban
sleep 2
fail2ban-client status
fail2ban-client status sshd
```

## Verify

```bash
# Both jails active
fail2ban-client status
# Should show: Jail list: recidive, sshd

# SSH jail details
fail2ban-client status sshd
# Should show port 2222 (or your port) and any banned IPs
```

Don't be surprised if IPs are already banned — bots are fast.

---

## Checklist

- [ ] fail2ban running and enabled
- [ ] `/etc/fail2ban/jail.local` created with correct SSH port
- [ ] SSH jail active (maxretry=3, bantime=1h)
- [ ] Recidive jail active (repeat offenders banned 1 week)
