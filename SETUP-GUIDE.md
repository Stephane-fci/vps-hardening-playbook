# Setup Guide — Fresh VPS Hardening

*Follow this from top to bottom for a new VPS. Each step references a detailed file in `core/`.*

---

## Before You Begin

**Read `core/00-preparation.md` first.** It covers:
- Questions to ask your human (SSH access, backups, what stays public, audit preference)
- Taking a snapshot/backup (mandatory before any changes)
- Verifying access (SSH, Tailscale if applicable)
- Setting up a backup operator (Claude Code or similar)

**Do NOT proceed until preparation is complete.** If you skip the snapshot and something goes wrong, there's no rollback.

---

## Execution Flow

### Phase 1: Foundation (Low Risk)

These steps are safe and non-disruptive. They prepare the server without changing access.

| Step | File | What | Risk |
|------|------|------|------|
| 1 | `core/01-system-updates.md` | apt update/upgrade, install security tools, configure auto-updates | Low |
| 2 | `core/02-admin-user.md` | Create non-root admin user with SSH key and sudo | Low |

**After Phase 1:** Verify you can SSH as the new admin user. Don't proceed until this works.

### Phase 2: Access Hardening (Medium Risk)

These steps change how SSH works. **One mistake can lock you out.**

| Step | File | What | Risk |
|------|------|------|------|
| 3 | `core/03-ssh.md` | Harden SSH config, change port, disable root/password | ⚠️ Medium |
| 4 | `core/04-firewall.md` | UFW: default deny, allow only what's needed | ⚠️ Medium |
| 5 | `core/05-fail2ban.md` | Brute-force protection with SSH jail | Low |

**After Phase 2:** Verify SSH still works on the new port. Verify firewall allows what it should and blocks what it shouldn't. This is a good checkpoint for the auditor.

⚠️ **If you're an agent hardening your own server:** Phase 2 changes might restart SSH. You won't lose your connection (existing sessions survive), but verify the new config before closing your session.

### Phase 3: Network Lockdown (High Risk)

This step removes public SSH access entirely. After this, SSH only works through Tailscale VPN.

| Step | File | What | Risk |
|------|------|------|------|
| 6 | `core/06-tailscale.md` | Install/verify Tailscale, lock SSH to Tailscale only | 🔴 High |

**After Phase 3:** Verify SSH works via Tailscale IP. Verify public SSH is refused. **Your backup operator must also verify they can still reach the server.**

⚠️ **This is the most dangerous step.** If Tailscale breaks and public SSH is already gone, you're locked out. Recovery requires Hetzner Console (web VNC) or provider equivalent. Make sure your backup operator is ready.

**Skip this phase if:** You don't use Tailscale, or you need public SSH access. Adjust firewall rules in Step 4 to allow SSH from specific IPs instead.

### Phase 4: System Hardening (Low-Medium Risk)

These steps harden the OS and services. Low risk individually but applied together they make a real difference.

| Step | File | What | Risk |
|------|------|------|------|
| 7 | `core/07-services.md` | Fix exposed ports, remove unnecessary services | Low |
| 8 | `core/08-kernel.md` | sysctl hardening, /proc hidden, IPv6 disabled | Low |
| 9 | `core/09-memory.md` | Swap, swappiness, memory management | Low |

### Phase 5: Application Hardening (Medium Risk)

This step hardens the OpenClaw (or other agent) service. **Restart required.**

| Step | File | What | Risk |
|------|------|------|------|
| 10 | `core/10-systemd.md` | MemoryMax, NoNewPrivileges, service hardening | ⚠️ Medium |

⚠️ **If you're an agent hardening your own server:** This step restarts you. Your backup operator must be ready to fix things if you don't come back. Tell your human: "I'm about to restart myself. If I'm not back in 60 seconds, use Claude Code to check on me."

### Phase 6: Monitoring & Verification (Low Risk)

| Step | File | What | Risk |
|------|------|------|------|
| 11 | `core/11-monitoring.md` | Memory alerts, disk protection, logging | Low |
| 12 | `core/12-verification.md` | Post-flight checks — verify EVERYTHING at runtime | Low |

---

## After Hardening: The Audit

Read `audit/AUDIT-PROTOCOL.md`.

Tell your human:

> "Hardening is complete. All steps passed verification. I recommend an independent audit — a separate agent (Claude Code or Cowork) SSHs in and verifies everything from scratch. This catches things I might have missed and also confirms that a backup operator can access the server if I ever crash during future updates.
>
> Want me to generate the audit prompt? I'll include the SSH details and what to check."

If they say yes → generate the prompt from `audit/AUDIT-PROMPT-TEMPLATE.md`.

If they say no → that's fine. The hardening is self-verified. The audit is recommended, not required.

---

## Profiles

After completing the core hardening, check if a profile applies:

- **Single OpenClaw agent?** → Read `profiles/single-agent.md`
- **Multiple isolated agents?** → Read `profiles/multi-agent.md`

Profiles add scenario-specific steps ON TOP of the core hardening.

---

## Time Estimate

| Phase | Steps | Time |
|-------|-------|------|
| Preparation | Snapshot + access check | 5-10 min |
| Phase 1: Foundation | Updates + admin user | 5-10 min |
| Phase 2: Access | SSH + firewall + fail2ban | 10 min |
| Phase 3: Network | Tailscale lockdown | 5 min |
| Phase 4: System | Services + kernel + memory | 5 min |
| Phase 5: Application | systemd hardening + restart | 5 min |
| Phase 6: Verification | Post-flight checks | 5 min |
| Audit (optional) | Independent verification | 5-10 min |
| **Total** | | **~30-50 min** |
