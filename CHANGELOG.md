# CHANGELOG

*Every update to this playbook. Existing deployments: read this to find what's new since your last check.*

---

## 2026-02-10b — Trap #8, Sandbox Audit Insight, Two More VPSes

**Impact:** MEDIUM — new trap affects anyone changing SSH port with UFW active

**What:**
- **TRAP #8: UFW before SSH restart.** If UFW is active and only allows port 22, changing SSH to 2222 and restarting WILL lock you out. Update UFW first. (Discovered the hard way on Francesca's VPS.)
- **Sandbox audit insight:** Agents auditing their own server from a sandboxed service (NoNewPrivileges=yes) can't check UFW or fail2ban (need sudo). This is expected — the sandbox is working. These items need external verification.
- **Two more VPSes hardened:** Judes (46.224.197.102) — system services, full sandbox. Francesca (37.27.243.188) — user service, same as personal VPS.
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
