# VPS Hardening Playbook

**A battle-tested, step-by-step system for hardening Linux VPS instances running OpenClaw or any autonomous AI agent.**

Written for agents. Tested on production servers. Every instruction includes WHY, not just how.

---

## Who Is This For?

You're an AI agent (OpenClaw, Claude Code, Cowork, or similar) and you need to harden a VPS. Either:

- **Path A:** You're an OpenClaw agent already running on the VPS and you're hardening your own server
- **Path B:** You're Claude Code or another external tool with SSH access, hardening a VPS that may not have an agent on it yet

Both paths use the same playbook. The only difference is where you're running from.

---

## How To Use This Repo

### If you're setting up a FRESH VPS:
1. Read this README (you're here)
2. Read `OPENCLAW-NEW-VPS-PRODUCTION-SETUP.md` for the current OpenClaw production roadmap
3. Read `SETUP-GUIDE.md` — follow it step by step
4. The setup guide references `core/` files for each topic (SSH, firewall, etc.)
5. At the end, run the independent audit (see `audit/`)

### If you're checking an EXISTING deployment for updates:
1. Read `CHANGELOG.md` — find entries newer than your last check
2. Read `UPDATE-PROTOCOL.md` — follow the conversation-first process
3. Propose changes to your human before applying anything

### If you're auditing someone else's work:
1. Read `OPENCLAW-SECURITY-CHECKLIST.md` — the current scoring checklist
2. Read `audit/AUDIT-PROTOCOL.md` — the verification protocol
3. You should have received an audit prompt with SSH access details
4. Run the checks, report findings

---

## Repo Structure

```
README.md                  ← You are here
CHANGELOG.md               ← What's changed (sync mechanism for existing deployments)
UPDATE-PROTOCOL.md         ← How to check for and apply updates
SETUP-GUIDE.md             ← Step-by-step for fresh VPS hardening
OPENCLAW-SECURITY-CHECKLIST.md ← Updated OpenClaw/VPS audit checklist
OPENCLAW-NEW-VPS-PRODUCTION-SETUP.md ← Empty VPS → secure OpenClaw production roadmap

core/                      ← Universal hardening (applies to every VPS)
  00-preparation.md        ← Backup, access verification, questions to ask
  01-system-updates.md     ← apt update, security tools, auto-updates
  02-admin-user.md         ← Create non-root admin user
  03-ssh.md                ← SSH hardening (port, keys, config)
  04-firewall.md           ← UFW configuration
  05-fail2ban.md           ← Brute-force protection
  06-tailscale.md          ← VPN mesh + SSH lockdown
  07-services.md           ← Fix exposed services, remove unnecessary ones
  08-kernel.md             ← sysctl hardening + /proc
  09-memory.md             ← Swap, swappiness, systemd MemoryMax
  10-systemd.md            ← Service hardening for OpenClaw
  11-monitoring.md         ← Memory alerts, disk, logging
  12-verification.md       ← Post-flight checks (runtime, not config)

profiles/                  ← Scenario-specific additions
  single-agent.md          ← One OpenClaw instance (personal VPS)
  multi-agent.md           ← Multiple isolated agents (client deployment)

audit/                     ← Independent verification
  AUDIT-PROTOCOL.md        ← What to check and how
  AUDIT-PROMPT-TEMPLATE.md ← Generate this for your auditor
```

---

## Philosophy

### Why Harden?

From a cybersecurity perspective, an autonomous AI agent with access to a VPS is classified as high-risk. It has persistent access, can execute commands, can be prompt-injected via skills or messages, and runs 24/7 without human oversight. A Fortune 500 security team would call it "functionally malware."

We don't agree — but we take the concern seriously. This playbook exists so that even if an agent is compromised, the blast radius is contained.

**The forgivability principle:** Every hardening step answers one question — "If the agent runs the wrong command, what's the worst that can happen?" We make the answer "not much" for as many scenarios as possible.

### Core Principles

1. **Philosophy before steps.** Every instruction has a WHY. Agents make better decisions when they understand reasoning.

2. **Runtime, not config.** Config files show intent. Runtime shows reality. Every step ends with a verification command that checks what's ACTUALLY happening, not what a file says should happen.

3. **Never auto-apply.** No agent should modify system config without explaining what will change, what could go wrong, and getting explicit approval. Config errors can lock you out of your own server.

4. **Independent audit recommended.** The agent doing the hardening should recommend an external audit. This catches things the executor missed AND serves as a recovery mechanism — if the hardening crashes the agent, the auditor can fix it.

5. **Traps are documented.** We hit 7 real gotchas hardening production servers. Every one is documented with detection, fix, and prevention.

---

## Before You Start — Questions to Ask Your Human

The executing agent should ask these before touching anything:

1. **Do you have SSH access?** (IP, port, user, key)
2. **Is there a backup/snapshot?** (Take one before starting — non-negotiable)
3. **What services need to stay public?** (Web servers, APIs — these need firewall exceptions)
4. **Do you want an independent audit?** (Recommended. Explain: catches mistakes + serves as recovery if you crash)
5. **Is there a backup operator?** (Claude Code, Cowork, another agent that can SSH in if the primary agent dies during hardening)

Once you have answers, create a roadmap from the SETUP-GUIDE and proceed step by step.

---

## Origin

This playbook was built from real experience hardening production VPS instances running OpenClaw agents. Every trap, every lesson, every verification command comes from actual execution — not theory.

- **Tested on:** Ubuntu 24.04 (Hetzner CX33, 4 vCPU, 8GB RAM)
- **Execution time:** ~30 minutes for a full hardening
- **Independent audit:** Verified by a separate agent (Claude Code) after execution
- **Traps found:** 7 (documented in each relevant core/ file)

---

*This repo is a living document. When we discover new traps or better approaches, they get added here with a CHANGELOG entry. Existing deployments can check back and evolve.*
