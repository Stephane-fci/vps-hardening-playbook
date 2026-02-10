# Audit Prompt Template

*The executing agent fills in the blanks below and gives this to the auditor (Claude Code, Cowork, etc.). Copy everything below the line.*

---

# VPS Security Audit Request

## Context

I just completed hardening a VPS using the VPS Hardening Playbook (https://github.com/Stephane-fci/vps-hardening-playbook). I need you to independently verify that everything was done correctly.

## Server Access

- **IP:** [FILL: Tailscale IP or public IP]
- **SSH User:** [FILL: admin username]
- **SSH Port:** [FILL: custom port]
- **SSH Key:** [FILL: key location or how to access]

Connect with:
```bash
ssh [USER]@[IP] -p [PORT]
```

## What Was Done

The following hardening steps were executed:

1. System updates (apt upgrade, security tools, auto-updates with reboot at 4AM)
2. Admin user created: [FILL: username]
3. SSH hardened: port [FILL], root disabled, password disabled, key-only
4. UFW firewall: default deny, allowed ports: [FILL: list of allowed ports and why]
5. fail2ban: SSH jail + recidive jail
6. Tailscale lockdown: [FILL: yes/no — if yes, SSH only via Tailscale]
7. Service cleanup: [FILL: what was fixed/removed]
8. Kernel hardening: 15+ sysctl params, /proc hidden, IPv6 disabled
9. Swap: [FILL: size] with swappiness=10
10. systemd: MemoryMax=[FILL], MemoryHigh=[FILL], NoNewPrivileges=yes
11. Monitoring: memory check cron every 15 min, journald limited to 500MB
12. Post-flight verification: all checks passed

## What To Check

Read the audit protocol at: https://github.com/Stephane-fci/vps-hardening-playbook/blob/main/audit/AUDIT-PROTOCOL.md

Key areas to verify:
- SSH config (runtime, not just config file)
- Firewall rules (only expected ports open)
- fail2ban (running, correct port)
- Services (nothing unexpected on public interfaces)
- **Kernel sysctl values at RUNTIME** (config file can lie — check with `sysctl` command)
- OpenClaw/agent service (MemoryMax enforced, NoNewPrivileges=yes)
- Swap and monitoring

## Known Limitations

[FILL: anything you know isn't perfect — e.g., "ProtectSystem not set because user service"]

## Report Format

Please report as:
- **Confirmed working:** list of items verified OK
- **Issues found:** with severity (critical/important/minor), what you found vs expected, and suggested fix

---

*Replace all [FILL: ...] sections before giving this to the auditor.*
