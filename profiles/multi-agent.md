# Profile: Multi-Agent Deployment

*Multiple isolated OpenClaw agents on the same VPS. Typically for client deployments where each team or function gets their own agent.*

---

## When This Applies

- Multiple agents on one server (e.g., one per team, one per client)
- Agents should NOT be able to access each other's data
- Different humans interact with different agents

## Additional Steps (On Top of Core Hardening)

### User Isolation

Create a separate Linux user for each agent:

```bash
useradd -m -s /bin/bash -d /home/agent-1 agent-1
useradd -m -s /bin/bash -d /home/agent-2 agent-2

# Lock down home directories
chmod 700 /home/agent-1 /home/agent-2

# Set strict umask
echo "umask 077" >> /home/agent-1/.bashrc
echo "umask 077" >> /home/agent-2/.bashrc
```

Each agent can only see its own files. /proc is already hidden between users (Step 8).

### Memory Limits Per Agent

Split MemoryMax across agents. For 8GB RAM with 2 agents:

```ini
# Agent 1 service
MemoryMax=3G
MemoryHigh=2.5G

# Agent 2 service  
MemoryMax=3G
MemoryHigh=2.5G
```

This leaves ~2GB for the OS. Adjust based on agent workload.

### systemd Services Per Agent

Each agent gets its own systemd service (system services, not user services, for proper isolation):

```bash
# /etc/systemd/system/openclaw-agent1.service
[Unit]
Description=OpenClaw Agent 1
After=network-online.target

[Service]
User=agent-1
WorkingDirectory=/home/agent-1
ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789
Restart=always
RestartSec=5
MemoryMax=3G
MemoryHigh=2.5G
NoNewPrivileges=yes
ProtectSystem=strict
PrivateTmp=yes
ReadWritePaths=/home/agent-1

[Install]
WantedBy=multi-user.target
```

**Note:** System services (not user services) support full sandboxing: ProtectSystem, PrivateTmp, ReadWritePaths all work here.

### Cross-Agent Management

If one agent should be able to manage another (e.g., a "boss" agent that can restart workers):

```bash
# /etc/sudoers.d/agent1-manage-agent2
agent-1 ALL=(root) NOPASSWD: /usr/bin/systemctl restart openclaw-agent2
agent-1 ALL=(root) NOPASSWD: /usr/bin/systemctl stop openclaw-agent2
agent-1 ALL=(root) NOPASSWD: /usr/bin/systemctl start openclaw-agent2
agent-1 ALL=(root) NOPASSWD: /usr/bin/systemctl status openclaw-agent2
agent-1 ALL=(root) NOPASSWD: /usr/bin/journalctl -u openclaw-agent2 *
```

This gives agent-1 power over agent-2's service without giving it root or access to agent-2's files.

### Network Isolation (Optional)

If agents should not be able to talk to each other's API ports:
```bash
# Use nftables to block cross-port access
# Agent 1 on port 18789, Agent 2 on port 18790
# Block agent-1 from reaching localhost:18790 and vice versa
```

This is optional — most deployments don't need it. Use when agents handle different clients' sensitive data.

### Backup Operator Pattern

For multi-agent deployments:
- **Primary admin:** An SSH user with sudo (the human or a management agent)
- **Per-agent recovery:** Each agent's human should have a backup path (Claude Code or Cowork)
- **Central monitoring:** One agent or a cron job that checks all agent services

---

## Checklist (Additional to Core)

- [ ] Separate Linux user per agent
- [ ] Home directories chmod 700
- [ ] Separate systemd services (system, not user)
- [ ] MemoryMax split across agents
- [ ] ProtectSystem=strict and PrivateTmp=yes (system services support this)
- [ ] ReadWritePaths limited to each agent's home
- [ ] Cross-management sudoers if needed
- [ ] Each agent's human has a backup operator path
