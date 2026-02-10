# Profile: Single Agent

*One OpenClaw instance running on the VPS. Personal deployment or single-purpose server.*

---

## When This Applies

- One OpenClaw agent on the server
- The agent's human is the only user
- No multi-tenancy concerns

## Additional Considerations

### Memory Sizing

For a single agent on 8GB RAM:
- **MemoryMax=6G** — leaves ~2GB for OS, nginx, Tailscale, and other services
- Sub-agents spawn temporarily and add ~200-400MB RSS each
- With 4+ concurrent sub-agents, you can hit 4-5GB easily
- 4GB swap gives you a buffer before OOM

### Workspace

The agent's workspace (e.g., `/root/clawd`) contains:
- Config files (`~/.openclaw/openclaw.json`)
- Memory files (daily notes, long-term memory)
- Project files, skills, tools

Ensure the workspace is included in any backup strategy.

### Backup Operator Pattern

The standard pattern for a single agent:
- **Primary:** The OpenClaw agent runs on the VPS
- **Backup:** Claude Code on the human's local machine, with SSH access via Tailscale
- If the agent crashes (OOM, bad config, systemd issue), Claude Code SSHs in and fixes it

Set this up during preparation (Step 0) and verify during the audit.

### Auto-Recovery

The systemd service has `Restart=always` and `RestartSec=5`. If the agent crashes, it restarts automatically within seconds. The human may not even notice a crash unless they check logs.

The risk is a crash LOOP (bad config → start → crash → restart → crash). MemoryMax prevents one common cause (OOM). For config-related crashes, the backup operator is the fix.
