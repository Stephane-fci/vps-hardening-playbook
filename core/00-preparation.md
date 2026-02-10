# Step 0: Preparation

*Do not touch anything on the server until this step is complete.*

---

## What

Gather information, take backups, verify access, and set up your safety net.

## Why

Hardening changes SSH config, firewall rules, and service files. One wrong move can lock you out. A snapshot lets you roll back. A backup operator can rescue you if you crash yourself.

## Questions to Ask Your Human

Before starting, get answers to:

1. **SSH access details:**
   - Server IP address
   - Current SSH user and port (likely `root@[IP]:22` for a fresh VPS)
   - SSH key (do you have one, or does the user need to provide it?)

2. **What services need to stay public?**
   - Web server (ports 80/443)?
   - Any custom APIs or webhooks?
   - These need firewall exceptions in Step 4.

3. **Do you want Tailscale VPN?**
   - If yes: SSH will be locked to Tailscale only (most secure)
   - If no: SSH stays on a custom port with firewall restrictions

4. **Do you want an independent audit?**
   - Recommended. Explain:
     > "After I complete the hardening, a separate agent (like Claude Code or Cowork) can SSH in and verify everything independently. This catches mistakes I might miss. It also confirms that a backup operator can reach the server — which matters because some hardening steps might restart me, and you'd need someone else to fix things if I don't come back."
   - If yes: you'll generate the audit prompt at the end
   - If no: proceed without (hardening is self-verified)

5. **Do you have a backup operator?**
   - Another agent or tool that can SSH into the server independently
   - Critical for Step 6 (Tailscale lockdown) and Step 10 (service restart)
   - If Claude Code: user needs to give it SSH key and connection details
   - If no backup operator: proceed with extra caution on high-risk steps

## Take a Snapshot (MANDATORY)

**Do this BEFORE any other changes.**

- **Hetzner:** Cloud Console → Server → Snapshots → Take Snapshot
- **DigitalOcean:** Droplet → Snapshots → Take Snapshot
- **AWS:** EC2 → Instance → Create AMI
- **Other:** Whatever your provider offers

Tell your human:
> "I need a point-in-time snapshot of the server before I start hardening. This is our rollback point if anything goes wrong. Can you take one now?"

If the provider requires the server to be off for a clean snapshot, decide with the human whether downtime is acceptable. Hot snapshots are usually fine.

**Do NOT proceed until the snapshot is confirmed.**

## Verify Access

SSH into the server and run:
```bash
echo "Connected as $(whoami) on $(hostname)"
uname -r
lsb_release -ds 2>/dev/null || cat /etc/os-release | grep PRETTY_NAME
```

Check what's currently running:
```bash
ss -tlnp | grep LISTEN
systemctl list-units --type=service --state=running | head -20
```

Record this output — it's your "before" state.

## Prepare the Backup Operator

If an independent audit or backup operator was agreed:

1. Share SSH access details with the backup operator (IP, user, port, key)
2. Have them verify they can connect BEFORE you start
3. Give them `audit/AUDIT-PROTOCOL.md` or the generated audit prompt

---

## Checklist

- [ ] SSH access details confirmed
- [ ] Public services identified (for firewall exceptions)
- [ ] Tailscale decision made (yes/no)
- [ ] Audit decision made (yes/no)
- [ ] Backup operator identified (or decided to proceed without)
- [ ] **Snapshot taken and confirmed**
- [ ] SSH access verified
- [ ] Current state recorded ("before" snapshot)
- [ ] Backup operator can connect (if applicable)
