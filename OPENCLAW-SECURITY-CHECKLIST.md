# OpenClaw VPS Security Checklist

Use this checklist for existing OpenClaw agents and new VPS deployments.

## Security target

Maximum agent freedom, protected by the strongest VPS and OpenClaw security baseline that does not break the operating model.

There is no zero-risk setup. The goal is:

- no unknown exposure;
- no avoidable stupid risk;
- every accepted risk documented;
- drift visible before it becomes an incident.

## Scoring labels

- **PASS** — correct.
- **WARN** — not ideal, but not urgent or needs judgment.
- **FAIL** — should be fixed.
- **N/A** — not relevant for this VPS/agent.
- **ACCEPTED** — known risk deliberately kept.
- **INVESTIGATE** — do not fix blindly; may break something.

---

# Part 1 — Universal VPS baseline

## 1. Public exposure

Purpose: know exactly what the internet can touch.

- [ ] Public IP is documented.
- [ ] Provider firewall is enabled where available.
- [ ] UFW is enabled.
- [ ] UFW default incoming policy is deny.
- [ ] Only required public ports are open.
- [ ] Public web ports are intentional.
- [ ] No database ports are public.
- [ ] No browser debugging / CDP ports are public.
- [ ] No forgotten dev server is public.
- [ ] Docker-published ports are reviewed separately.
- [ ] `ss -tulpn` output is reviewed and saved.

## 2. SSH hardening

Default for new VPSes is hardened public SSH, unless a private network/VPN-only posture is explicitly chosen.

- [ ] SSH runs on the intended port.
- [ ] `PermitRootLogin no`.
- [ ] `PasswordAuthentication no`.
- [ ] `KbdInteractiveAuthentication no`.
- [ ] `PubkeyAuthentication yes`.
- [ ] `PermitEmptyPasswords no`.
- [ ] `AllowUsers` or `AllowGroups` restricted to expected operators.
- [ ] `MaxAuthTries 3`.
- [ ] `LoginGraceTime 20-30s`.
- [ ] `ClientAliveInterval 300`.
- [ ] `ClientAliveCountMax 2`.
- [ ] `X11Forwarding no`.
- [ ] `AllowTcpForwarding no`, unless there is a documented need.
- [ ] `AllowAgentForwarding no`, unless there is a documented need.
- [ ] Agent Linux users do not have SSH access unless documented.
- [ ] Root SSH is never used.
- [ ] Before changing SSH/firewall rules, keep one working session open and test a second login.
- [ ] Provider rescue console / break-glass path is documented.

## 3. Firewall and intrusion protection

- [ ] UFW rules match the intended exposure model.
- [ ] SSH is protected by key-only auth and fail2ban.
- [ ] SSH is rate-limited or source-restricted if compatible with operational needs.
- [ ] Cloudflare-only web exposure is enforced where intended.
- [ ] fail2ban is installed and active.
- [ ] fail2ban has an `sshd` jail.
- [ ] fail2ban has a `recidive` jail where useful.
- [ ] fail2ban uses a working backend on Ubuntu 24.04, usually systemd/journald.
- [ ] fail2ban `ignoreip` contains only loopback and trusted operator/static infrastructure IPs.
- [ ] `fail2ban-client status` is reviewed and saved.
- [ ] `ufw status verbose` is reviewed and saved.

## 4. Updates and package hygiene

- [ ] `unattended-upgrades` is installed.
- [ ] Security updates are enabled.
- [ ] Reboot policy is explicit.
- [ ] Recent unattended-upgrades logs are reviewed.
- [ ] Pending reboot marker is checked.
- [ ] Current high-profile Linux CVEs are checked against package/kernel state when relevant.
- [ ] Copy Fail / CVE-2026-31431 posture is checked on Ubuntu 24.04-class hosts: patched `kmod` package or explicit `algif_aead` module block exists, `algif_aead` is not loaded, and mitigation evidence is saved.
- [ ] If Copy Fail is unmitigated, apply the mitigation below before marking the host clean.
- [ ] Useless server packages are removed, not only stopped.
- [ ] CUPS is not installed/running unless explicitly needed.
- [ ] VNC / x11vnc is not running unless explicitly needed and protected.
- [ ] websockify is not running unless explicitly needed and protected.
- [ ] Samba/NFS/rpcbind/FTP/telnet are absent unless explicitly needed.
- [ ] Old dashboards, dev servers, stale cron jobs, and old systemd units are reviewed.


### Copy Fail / CVE-2026-31431 mitigation checklist

Copy Fail is a Linux local privilege escalation issue. It is not a remote exploit by itself, but it matters on agent hosts because any local code execution path can become much worse if local privilege escalation is available.

For Ubuntu 24.04-class hosts, verify and mitigate like this:

- [ ] Check OS/kernel/kmod state and save evidence:

```bash
lsb_release -a 2>/dev/null || cat /etc/os-release
uname -a
dpkg-query -W kmod libkmod2 2>/dev/null || true
lsmod | grep -E '^algif_aead\b' || true
modprobe -n -v algif_aead 2>&1 || true
```

- [ ] Preferred fix: install the Ubuntu-patched `kmod`/`libkmod2` packages when available.

```bash
apt-get update
apt-get install --only-upgrade kmod libkmod2
```

- [ ] If apt/dpkg is locked, first determine whether it is a live update or a stale lock. Do **not** kill active package work blindly. If it is stale, clear it deliberately, then finish dpkg before retrying package updates.

```bash
ps -fp <APT_OR_DPKG_PID>
dpkg --configure -a
apt-get -f install
```

- [ ] Add or keep an explicit module block for defense in depth:

```bash
printf 'install algif_aead /bin/false\nblacklist algif_aead\n' > /etc/modprobe.d/disable-algif_aead.conf
```

- [ ] Verify the module is not loaded and cannot be loaded:

```bash
lsmod | grep -E '^algif_aead\b' && echo 'FAIL: algif_aead loaded' || echo 'PASS: algif_aead not loaded'
modprobe algif_aead && echo 'FAIL: algif_aead load succeeded' || echo 'PASS: algif_aead blocked'
```

- [ ] If a newer kernel was installed, schedule a controlled reboot and verify the running kernel afterward. A package/module mitigation can make the host safe enough short-term, but a pending kernel reboot should remain visible.

```bash
uname -r
ls /boot/vmlinuz-* 2>/dev/null | sort -V | tail
```

Classification:

- **PASS** — patched `kmod`/`libkmod2` installed or explicit `algif_aead` block exists, `algif_aead` is not loaded, and loading it fails.
- **WARN** — module blocked but newer kernel is installed and not yet running; schedule controlled reboot.
- **FAIL** — no patched package, no explicit module block, or `algif_aead` can still load.
- **INVESTIGATE** — package manager lock, unknown distro status, or custom kernel/module behavior.

## 5. Kernel and runtime hardening

Check live runtime values, not only config files.

- [ ] `net.ipv4.tcp_syncookies=1`.
- [ ] `accept_source_route=0`.
- [ ] `accept_redirects=0`.
- [ ] `secure_redirects=0`.
- [ ] `send_redirects=0`.
- [ ] `log_martians=1`.
- [ ] `rp_filter` hardened.
- [ ] `randomize_va_space=2`.
- [ ] `dmesg_restrict=1`.
- [ ] `kptr_restrict=1` or stricter.
- [ ] `ptrace_scope=1` or stricter, preferably `2` if compatible.
- [ ] `kernel.unprivileged_bpf_disabled=1` where compatible.
- [ ] `net.core.bpf_jit_harden=2` where available.
- [ ] `ip_forward=0`, unless Docker/routing/VPN needs it. If `1`, mark INVESTIGATE or ACCEPTED with reason.
- [ ] IPv6 firewalling/hardening is correct if IPv6 is enabled.
- [ ] IPv6 disabled if intentionally unused.
- [ ] `/proc` hidepid protection is enabled where compatible.

## 6. Users, sudo, and file permissions

- [ ] Runtime user is known.
- [ ] Service ownership is documented.
- [ ] Sudo rights are minimal.
- [ ] No `NOPASSWD:ALL` unless explicitly accepted.
- [ ] Agent users are separated where useful.
- [ ] `.ssh` directories are `700`.
- [ ] SSH private keys are `600`.
- [ ] `authorized_keys` files are `600`.
- [ ] OpenClaw state directory is `700` class permissions.
- [ ] `openclaw.json` is `600` class permissions.
- [ ] auth profiles are `600` class permissions.
- [ ] credentials directories are `700` class permissions.
- [ ] session stores/logs are not world-readable.
- [ ] config backups have the same permission discipline as live config.
- [ ] no credentials are stored in git-tracked workspace files.
- [ ] stale tokens/auth profiles are removed when safe.

## 7. Services and reliability

- [ ] Agent runs under a named systemd service.
- [ ] Restart policy is explicit.
- [ ] Startup timeout is explicit.
- [ ] Service memory limits are set and documented.
- [ ] `NODE_OPTIONS` memory cap is set where needed.
- [ ] Service logs are bounded through journald config or equivalent.
- [ ] Service can restart cleanly.
- [ ] Service user/workspace ownership is correct.
- [ ] `UMask=0077` is preferred where compatible. If weaker, document why.
- [ ] `NoNewPrivileges=yes` is preferred where compatible. If `no`, document why.
- [ ] Advanced systemd hardening is evaluated carefully before enabling, not blindly applied.

---

# Part 2 — OpenClaw baseline

## 8. Gateway and dashboard security

- [ ] `openclaw status --deep` is healthy.
- [ ] `openclaw security audit` has no critical findings.
- [ ] `openclaw security audit --deep` has no unresolved direct exposure findings.
- [ ] OpenClaw WARN labels are classified: fixed, documented accepted risk, or added to follow-up.
- [ ] Known OpenClaw WARN labels are listed in `acceptedWarnKeys` only after review. Unknown WARN labels must page as `⚠️ Review needed`.
- [ ] Gateway bind is loopback unless explicitly reviewed.
- [ ] Gateway auth is enabled.
- [ ] Dashboard/control UI is not public unless explicitly reviewed.
- [ ] `gateway.controlUi.allowInsecureAuth=false`.
- [ ] `gateway.controlUi.dangerouslyDisableDeviceAuth=false`.
- [ ] `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=false`.
- [ ] `allowedOrigins` is explicit if dashboard is exposed through a browser origin.
- [ ] `allowedOrigins=["*"]` is not used outside temporary local testing.
- [ ] trusted proxy settings are configured if reverse proxy identity/local-client headers matter.
- [ ] gateway secrets are preferably environment variables, not plain config fields.
- [ ] sensitive logging redaction is enabled.
- [ ] mDNS/discovery is off unless needed. If on, document why.

## 9. Hooks, webhooks, plugins

- [ ] Internal hooks list is reviewed.
- [ ] Enabled hooks are still needed.
- [ ] Hook source is trusted.
- [ ] Hook events are specific, not overly broad.
- [ ] Webhooks are disabled unless explicitly needed.
- [ ] Webhook token is not reused as gateway token.
- [ ] Webhook path is not `/`.
- [ ] Webhook allowed agent IDs are restricted.
- [ ] Installed plugins are reviewed.
- [ ] Unused plugins are removed.
- [ ] Plugin hooks/tool hooks are understood before enabling.

## 10. Tool freedom and accepted risk

This section records deliberate tradeoffs. Do not blindly fix these.

- [ ] `exec` access is deliberate.
- [ ] `tools.exec.security=full`, if used, is marked ACCEPTED with reason.
- [ ] Elevated tools are enabled only where needed.
- [ ] Filesystem access is appropriate for the agent profile.
- [ ] Browser control is enabled only where useful.
- [ ] Browser profile is dedicated or risk is accepted.
- [ ] Remote nodes/devices are paired only where needed.
- [ ] Runtime tools are not exposed to untrusted rooms.
- [ ] Destructive/external/public actions require clear authorization.
- [ ] Agent-freedom exceptions are documented in the audit result.

---

# Part 3 — Agent access profiles

## 11. Universal channel/access checks

- [ ] Agent owner is documented.
- [ ] Expected Discord guilds are documented by ID.
- [ ] Expected Slack workspaces/channels are documented by ID.
- [ ] Expected Telegram chats/groups are documented by ID.
- [ ] Bot is not present in unknown guilds/workspaces/chats.
- [ ] Channel allowlists use IDs, not display names.
- [ ] Allowed human users are documented by ID where needed.
- [ ] Broad/shared rooms require mention unless ambient listening is intended.
- [ ] Dedicated private agent channels may allow normal messages if intended.
- [ ] Slash command exposure is reviewed.
- [ ] Bot messages are ignored unless bot-to-bot workflows are deliberate.
- [ ] If bot-to-bot is allowed, loop guards exist.

## 12. Common agent profile checks

### Super-agent / fleet controller

- [ ] Only the human owner has broad control authority.
- [ ] External/client guild access does not allow others to trigger the agent broadly.
- [ ] Dangerous actions are owner-authorized.
- [ ] Cross-client context remains protected.

### Client/team operator agent

- [ ] Permission database or explicit allowlist is the authority, not casual workspace membership.
- [ ] Read/write/draft/send powers are separated.
- [ ] Dangerous client-impacting actions require explicit capability or approval.
- [ ] Slack/Discord scopes and channel access are reviewed.
- [ ] Credential scope matches the agent's actual job.

### Multi-agent client boundary

- [ ] Each agent has its own Linux runtime user, OpenClaw state dir, workspace, Discord app/bot, backup repo, snapshot worker, and security audit destination.
- [ ] Credentials are not copied between agents unless the role requires it and the transfer is documented.
- [ ] Weekly self-audits are per-agent and local. No agent posts a combined report about other agents inside a client/team server.
- [ ] Shared token-service consumers are documented as consumers only; refresh-token ownership remains centralized.

### Private/family/personal agent

- [ ] Agent is only in expected private server(s).
- [ ] Meaningful users are explicitly documented.
- [ ] No broad workspace/client exposure.
- [ ] No unexpected public ports or dashboards.
- [ ] No private context from other clients/agents leaks into this server.

### Shared VPS agents

- [ ] Separate Linux users.
- [ ] Separate workspaces.
- [ ] Separate configs.
- [ ] Separate credentials.
- [ ] Separate service names.
- [ ] Separate logs.
- [ ] Separate channel allowlists.
- [ ] No shared admin token unless explicitly documented.
- [ ] One agent does not inherit another agent's API access.
- [ ] Shared VPS disk/memory pressure is reviewed.

---

# Part 4 — Audit rhythm

## 13. Weekly light audit

Run weekly or biweekly.

- [ ] `openclaw status --deep`.
- [ ] `openclaw security audit --deep`.
- [ ] `ss -tulpn` port inventory.
- [ ] `ufw status verbose`.
- [ ] `fail2ban-client status` and key jails.
- [ ] disk usage.
- [ ] memory usage.
- [ ] service status.
- [ ] recent high-severity logs.
- [ ] unknown running services.
- [ ] Discord/Slack/Telegram access drift.
- [ ] credential directory permissions.
- [ ] pending reboot / unattended upgrades.

## 14. Monthly deep audit

Run monthly or after major changes.

- [ ] Full universal VPS baseline.
- [ ] Full OpenClaw baseline.
- [ ] Full agent profile review.
- [ ] config diff since last audit.
- [ ] credential/auth profile inventory.
- [ ] service unit review.
- [ ] reverse proxy review.
- [ ] plugin/hook review.
- [ ] channel/permission model review.
- [ ] accepted-risk list review.
- [ ] backup and restore path review.

## 15. Triggered audit

Run after any of these:

- [ ] New VPS created.
- [ ] OpenClaw upgraded.
- [ ] New channel connected.
- [ ] Discord/Slack permissions changed.
- [ ] New plugin/hook/webhook installed.
- [ ] New browser/node/device control enabled.
- [ ] Firewall/proxy/SSH changed.
- [ ] Credential or auth profile changed.
- [ ] Service unit changed.
- [ ] Agent gets a new external write tool.

---

# Part 5 — New VPS production gate

A new VPS is not production-ready until this section passes.

- [ ] Non-root admin user created.
- [ ] SSH key auth installed and tested.
- [ ] Root SSH disabled.
- [ ] Password SSH disabled.
- [ ] SSH hardening file validated with `sshd -t`.
- [ ] UFW enabled.
- [ ] fail2ban enabled.
- [ ] Provider firewall configured where available.
- [ ] sysctl hardening applied and live-checked.
- [ ] unattended upgrades enabled.
- [ ] only required packages installed.
- [ ] OpenClaw installed.
- [ ] OpenClaw gateway loopback-bound.
- [ ] OpenClaw service created and restart-tested.
- [ ] memory limits set.
- [ ] credentials outside workspace.
- [ ] `.openclaw` permissions set.
- [ ] channels configured with explicit allowlists.
- [ ] agent profile written.
- [ ] `openclaw doctor` passes or findings understood.
- [ ] `openclaw security audit --deep` passes or findings classified.
- [ ] `wf-security-audit` or equivalent local self-audit is installed, scheduled, manually smoke-tested, and reports only to this agent's own server/channel.
- [ ] self-audit config includes `acceptedCriticalKeys` and `acceptedWarnKeys` with only reviewed labels.
- [ ] Unknown OpenClaw WARN behavior is verified: unaccepted WARN labels produce `⚠️ Review needed`, accepted labels appear only as documented notes.
- [ ] open port inventory saved.
- [ ] real channel test completed.
- [ ] backup / rollback path documented.

---

# Audit output template

```text
VPS:
Agent(s):
Date:
Audit type: weekly / monthly / triggered / new VPS
Auditor:

Summary in plain English:

PASS:
WARN:
FAIL:
INVESTIGATE:
ACCEPTED:
N/A:

Fix now:
Investigate first:
Accepted risks:
Side notes:

Evidence saved:
Next audit date:
```
