# CHANGELOG

## 2026-05-05 — OpenClaw security checklist refresh

- Added `OPENCLAW-SECURITY-CHECKLIST.md` as the current reusable VPS/OpenClaw audit checklist.
- Added `OPENCLAW-NEW-VPS-PRODUCTION-SETUP.md` for the empty-VPS-to-production OpenClaw roadmap.
- Added newer operating rules: classify OpenClaw WARN labels, document accepted risk, page on unknown WARNs, keep self-audits per-agent/local, and include Copy Fail / CVE-2026-31431 posture checks for Ubuntu 24.04-class hosts.
- Updated `README.md` to point fresh deployments and audits to the new OpenClaw-specific docs.


*Every update to this playbook. Existing deployments: read this to find what's new since your last check.*

---

## 2026-02-15 — SSH Service Name Trap, Agent Tuning, Console Recovery

**Impact:** LOW-MEDIUM — prevents confusion on Ubuntu 24.04+, helps agent deployments

**What:**
- **SSH service name clarification:** Ubuntu 24.04+ uses `ssh.service`, NOT `sshd.service`. Added explicit trap warning in Step 3.6. People coming from CentOS/RHEL will instinctively type `systemctl restart sshd` — on Ubuntu, that fails silently or doesn't exist.
- **MaxStartups tuning for agent workloads (Step 3.7):** When deploying workspace files to a VPS via bulk SSH transfers (common in multi-agent setups), the default `MaxStartups 10:30:60` throttles connections. New section with recommended values for agent workloads: `MaxStartups 30:50:100` + `MaxSessions 10`.
- **Hetzner Console recovery note:** Documented that Hetzner's web console doesn't work with key-only SSH (no password = can't login via console). Added alternative recovery paths: Rescue Mode, backup operator user, snapshots.

**Source:** Deployed fourth production VPS (multi-agent, Hetzner CPX32). Hit MaxStartups throttling during bulk file transfers. Confirmed Hetzner Console limitation firsthand during lockout recovery.

---

## 2026-02-10b — Trap #8, Sandbox Audit Insight, Two More VPSes

**Impact:** MEDIUM — new trap affects anyone changing SSH port with UFW active

**What:**
- **TRAP #8: UFW before SSH restart.** If UFW is active and only allows port 22, changing SSH to 2222 and restarting WILL lock you out. Update UFW first. (Discovered the hard way on Francesca's VPS.)
- **Sandbox audit insight:** Agents auditing their own server from a sandboxed service (NoNewPrivileges=yes) can't check UFW or fail2ban (need sudo). This is expected — the sandbox is working. These items need external verification.
- **Two more VPSes hardened:** one with system services (full sandbox), one with user service (same pattern as personal VPS).
- **Updated `core/03-ssh.md`:** Added firewall step before SSH restart.
- **Updated `audit/AUDIT-PROTOCOL.md`:** Added self-audit sandbox note.

**Source:** Hardened and audited three production VPSes. Independent audits by Jess (Judes agent) and Francesca (Sally's agent) confirmed all steps.

---

## 2026-02-10 — Initial Release

**Impact:** N/A (first version)

**What:** Complete VPS hardening playbook built from real production experience.

- 12-step hardening process (preparation through verification)
- 7 documented traps with detection and fixes
- 10 strategic lessons learned
- Independent audit protocol with prompt templates
- Profiles for single-agent and multi-agent deployments
- Full before/after comparison with runtime verification commands

**Source:** Hardened two production Hetzner VPS instances (personal + client). Independent audit by Claude Code confirmed all steps.
