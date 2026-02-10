# Update Protocol

*How existing deployments check for updates and apply them safely.*

---

## When to Check

- When your human asks you to
- Periodically (e.g., monthly via cron job)
- After any security incident

## How to Check

1. Pull or fetch the latest from this repo
2. Read `CHANGELOG.md` — find entries newer than your last check date
3. If no new entries → "Checked the hardening playbook, nothing new since [date]." Stay silent if this was an automated check.
4. If new entries exist → proceed to step 4

## How to Propose Changes

4. For each new changelog entry, explain to your human:
   - **What changed** and why
   - **Impact level** (breaking / enhancement / minor)
   - **What it means for YOUR specific server** — not generic, specific
   - **What needs to happen** to apply it

Example:
> "The hardening playbook has 1 update since Feb 10:
>
> **New trap: systemd timer override** (enhancement)
> A new gotcha was discovered where systemd timers can reset sysctl params. For our server, this means our kernel hardening might be getting silently undone on reboot.
>
> To fix it: add a oneshot service that re-applies sysctl after boot. Takes 2 minutes.
>
> Want me to apply this?"

5. **Wait for approval.** Never apply without explicit confirmation.

## How to Apply

6. Create a mini roadmap for the approved changes
7. Execute step by step
8. Verify each change with the runtime verification command (not just config)
9. Record today's date as your last check date
10. If the change is security-critical, recommend a quick audit

## Config File Safety

Any change that touches these files requires **extra caution and explicit approval:**
- `/etc/ssh/sshd_config` (or ssh.socket override)
- `/etc/ufw/` or any firewall rule
- `/etc/fail2ban/jail.local`
- `/etc/sysctl.d/*.conf`
- systemd service files (especially OpenClaw's)
- `/etc/fstab`

For these files:
- Explain what EXACTLY will change
- Explain what could go wrong (worst case)
- Explain how to roll back
- Wait for explicit "go ahead"
- Keep a backup of the original

---

*The golden rule: conversation first, changes second. Your human should never be surprised by what you did.*
