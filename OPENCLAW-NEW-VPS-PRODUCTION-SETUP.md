# OpenClaw New VPS Production Setup Roadmap

This is the short execution roadmap for taking an empty Ubuntu VPS to a secure, production-ready OpenClaw agent.

It is not a full agent deployment playbook. It covers the security-critical path:

```text
empty VPS → hardened Linux baseline → OpenClaw service running → first secure channel test → weekly self-audit configured
```

## Operating rules

1. **Do not lock yourself out.** Keep the current SSH session open until a second fresh SSH login works.
2. **Do not expose OpenClaw by accident.** Gateway/dashboard stay localhost-only unless explicitly designed otherwise.
3. **Do not connect channels before the access model is decided.** Discord/Slack access is part of the security model.
4. **Do not auto-fix access policy.** Slack/Discord/OpenClaw access warnings are documented unless the human owner explicitly approves the exact change.
5. **Each agent stays in its own boundary.** No cross-agent security reports inside a client/team server.
6. **Evidence matters.** Save command outputs so the final state is auditable.
7. **Unknown audit warnings page.** The recurring self-audit may only report `✅ Clean` when OpenClaw WARN labels are absent or explicitly documented in `acceptedWarnKeys`.

## Variables to define first

```bash
AGENT_NAME="example-agent"
HOSTNAME="example-agent-vps"
OPERATOR_USER="stephane"
AGENT_USER="example-agent"
SSH_PORT="2222"
TIMEZONE="Europe/Paris"
WORKSPACE="/home/${AGENT_USER}/.openclaw/workspace"
STATE_DIR="/home/${AGENT_USER}/.openclaw"
SERVICE_NAME="openclaw"
EVIDENCE_DIR="/home/${AGENT_USER}/security-setup-evidence"
```

For a shared VPS, define one `AGENT_USER`, `WORKSPACE`, `STATE_DIR`, and `SERVICE_NAME` per agent. Do not reuse credentials or state directories unless explicitly documented.

## Production-ready definition

A new VPS is production-ready when:

- SSH is key-only and root/password login is disabled.
- Firewall and fail2ban are active.
- Only intentional ports are reachable.
- OpenClaw runs as a service under the right user.
- OpenClaw gateway/control surfaces are not publicly exposed by accident.
- Credentials live outside the workspace and have tight permissions.
- The agent has a clear role, communication surface, and tool boundary.
- The agent has its own server/channel setup where applicable.
- `openclaw doctor` and `openclaw security audit --deep` have been run.
- Weekly/local self-audit is configured for this agent only.
- Weekly/local self-audit is smoke-tested and unknown OpenClaw WARN labels produce `⚠️ Review needed`.
- Rollback/rescue access is documented.

---

# Phase 0 — Agent security profile

Answer these before or during VPS setup.

1. **What is this agent's role?** Personal agent, client operator, support agent, content agent, fleet/controller agent, experimental agent?
2. **Who is allowed to control it?** One owner, one trusted person, client team, private family server, public/community surface?
3. **Where will people talk to it?** Discord, Slack, Telegram, email, webhooks, other?
4. **Should it listen ambiently or only when mentioned?** Private one-agent channel can be normal conversation. Group/team channels usually need mention or explicit allowlist.
5. **What data will it see?** Personal, client, ecommerce, support tickets, financial, analytics, credentials, private family context?
6. **What tools will it have?** Read-only tools, browser, filesystem, exec, external writes, ecommerce tools, email, social posting, dashboards?
7. **What can cause real damage if misused?** Sending messages, changing orders, issuing refunds, writing ads, deleting files, restarting services, changing DNS/config, exposing private context?
8. **Does it need public web surfaces?** Dashboard, FileBrowser, preview site, webhook, API, client portal? If no, keep all web surfaces private/localhost.
9. **Is this VPS single-agent or shared?** Shared VPS requires separate Linux users, services, workspaces, configs, credentials, logs, and channels.
10. **What accepted risks are intentional?** Example: a fleet controller may have broad shell/filesystem access; client agents should usually have narrower boundaries.

Save the output:

```text
Agent name:
Role:
Owner/controller:
Communication surfaces:
Allowed humans:
Main data/tools:
Dangerous actions:
Public surfaces needed:
Single or shared VPS:
Accepted risks:
Security posture: personal / client-team / high-risk-write / experimental
```

## Quick decision table

| If the answer is... | Default posture |
|---|---|
| Owner-only personal/operator agent | Powerful tools allowed, narrow human control surface |
| Client/team agent | Explicit allowlists, mention in broad channels, write actions gated |
| Support/ecommerce agent | Separate read/write capabilities, no sends/refunds without workflow rules |
| Family/private agent | Simple trusted surface, no unrelated/client context copied in |
| Public/community surface | Mention required, no dangerous tools without explicit approval |
| Shared VPS | Separate Linux user/service/workspace/config/credentials per agent |
| Needs dashboard/FileBrowser | Localhost first; expose only behind Cloudflare Access or equivalent |

---

# Phase 1 — VPS provision and break-glass access

- [ ] VPS provider and plan chosen.
- [ ] Hostname chosen.
- [ ] Public IP documented.
- [ ] Provider rescue console / rebuild path documented.
- [ ] Approved SSH public key added at provision time.
- [ ] Initial root login works only long enough to create the real operator user.
- [ ] Timezone set.
- [ ] System packages updated.
- [ ] Reboot after first update if required.

Core commands:

```bash
timedatectl set-timezone "$TIMEZONE"
hostnamectl set-hostname "$HOSTNAME"
apt-get update
apt-get -y upgrade
apt-get -y install sudo curl git jq ufw fail2ban unattended-upgrades ca-certificates gnupg lsb-release
mkdir -p "$EVIDENCE_DIR"
```

Create the operator user:

```bash
adduser "$OPERATOR_USER"
usermod -aG sudo "$OPERATOR_USER"
install -d -m 700 -o "$OPERATOR_USER" -g "$OPERATOR_USER" "/home/${OPERATOR_USER}/.ssh"
nano "/home/${OPERATOR_USER}/.ssh/authorized_keys"
chmod 600 "/home/${OPERATOR_USER}/.ssh/authorized_keys"
chown -R "$OPERATOR_USER:$OPERATOR_USER" "/home/${OPERATOR_USER}/.ssh"
```

Test a second login as the operator user before touching firewall/SSH restrictions.

---

# Phase 2 — SSH hardening

Create `/etc/ssh/sshd_config.d/99-openclaw-hardening.conf`:

```text
Port 2222
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
AllowUsers stephane
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding local
```

Adjust `Port` and `AllowUsers`. Use `AllowTcpForwarding no` if local SSH tunnels are not needed.

Validate before restart:

```bash
sshd -t
systemctl restart ssh || systemctl restart sshd
ss -tulpn | grep ssh
```

Keep the original session open. Open a second fresh SSH session on the final port before continuing.

---

# Phase 3 — UFW and fail2ban

UFW baseline:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow "${SSH_PORT}/tcp"
# Add 80/443 only if public web is intentional.
# ufw allow 80/tcp
# ufw allow 443/tcp
ufw enable
ufw status verbose
```

fail2ban baseline:

```bash
systemctl enable --now fail2ban
fail2ban-client status
fail2ban-client status sshd
```

If fail2ban does not see SSH logs on Ubuntu 24.04, use the systemd backend for the sshd jail.

---

# Phase 4 — Kernel and package hygiene

Apply only compatible hardening. Verify live runtime values.

```bash
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.all.secure_redirects
sysctl net.ipv4.conf.all.send_redirects
sysctl kernel.randomize_va_space
sysctl kernel.dmesg_restrict
sysctl kernel.kptr_restrict
sysctl kernel.yama.ptrace_scope
```

Target values are listed in `OPENCLAW-SECURITY-CHECKLIST.md`.

Also check:

```bash
systemctl is-active unattended-upgrades
ls /var/run/reboot-required 2>/dev/null || true
ss -tulpn
```

Remove useless exposed packages/services rather than only stopping them.

---

# Phase 5 — OpenClaw install and service

Create a dedicated runtime user for the agent:

```bash
useradd --system --create-home --shell /usr/sbin/nologin "$AGENT_USER"
install -d -m 700 -o "$AGENT_USER" -g "$AGENT_USER" "$STATE_DIR"
install -d -m 700 -o "$AGENT_USER" -g "$AGENT_USER" "$WORKSPACE"
```

Install OpenClaw according to the current official docs for your environment.

Service baseline:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=example-agent
Group=example-agent
WorkingDirectory=/home/example-agent/.openclaw/workspace
Environment=HOME=/home/example-agent
Environment=OPENCLAW_STATE_DIR=/home/example-agent/.openclaw
Environment=NODE_OPTIONS=--max-old-space-size=4096
ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway
Restart=always
RestartSec=5
TimeoutStartSec=120
MemoryHigh=3G
MemoryMax=4G
UMask=0077

[Install]
WantedBy=multi-user.target
```

Use the Node entrypoint rather than a wrapper binary when `NODE_OPTIONS` must apply.

Then:

```bash
systemctl daemon-reload
systemctl enable --now "$SERVICE_NAME"
systemctl status "$SERVICE_NAME" --no-pager
```

---

# Phase 6 — OpenClaw security baseline

Check:

```bash
openclaw doctor
openclaw status --deep
openclaw security audit --deep
```

Required classifications:

- Critical/direct exposure: fix before production unless there is an explicit, documented exception.
- WARN labels: classify as fix, accepted, or follow-up.
- Unknown WARN labels in weekly audit: should produce `⚠️ Review needed`.
- Accepted WARN labels: only after human review, listed in `acceptedWarnKeys`, shown as notes, no page.

Gateway defaults:

- bind loopback;
- auth enabled;
- no public control UI unless explicitly designed;
- no `allowedOrigins=["*"]` outside temporary local testing;
- trusted proxies configured only when reverse proxy identity/local-client headers matter.

---

# Phase 7 — Channels and access model

Before enabling Discord/Slack/Telegram:

- [ ] agent owner documented;
- [ ] allowed humans documented by stable IDs;
- [ ] guild/workspace/channel IDs documented;
- [ ] broad rooms require mention unless ambient listening is intentional;
- [ ] bot-to-bot allowed only with loop guards;
- [ ] dangerous/external writes require explicit workflow rules.

Run at least one real channel test before production.

---

# Phase 8 — Weekly self-audit

Install a local recurring self-audit that checks only this agent and reports only to this agent's own server/channel.

Minimum checks:

- expected services active;
- disk usage;
- `ss -tulpn` listener inventory;
- OpenClaw health/status;
- `openclaw security audit --deep`;
- channel probe;
- accepted vs unknown OpenClaw WARN labels.

Rules:

- no shared client/team security report channel;
- no shared webhook across agents;
- unknown WARN labels page the owner;
- accepted labels do not page but appear as documented notes;
- evidence saved locally with private permissions.

---

# Phase 9 — Production gate

Do not mark production-ready until:

- [ ] root/password SSH disabled and fresh login verified;
- [ ] UFW/fail2ban active;
- [ ] sysctl hardening applied and runtime-checked;
- [ ] only intentional ports reachable;
- [ ] OpenClaw service active and restart-tested;
- [ ] gateway loopback-bound;
- [ ] credentials and OpenClaw state permissions tight;
- [ ] channel allowlists configured and tested;
- [ ] OpenClaw audit has no unclassified critical/direct exposure;
- [ ] WARN labels classified;
- [ ] self-audit installed and manually smoke-tested;
- [ ] unknown WARN behavior verified;
- [ ] evidence and rollback path saved.

## Stop conditions

Stop and ask before proceeding if:

- SSH/firewall changes might lock out all operators;
- OpenClaw control UI needs public exposure;
- a warning requires changing Slack/Discord/OpenClaw access policy;
- a tool restriction would reduce the agent's intended operating freedom;
- credentials or private context would need to be copied between agents;
- the system has an unclassified public listener.
