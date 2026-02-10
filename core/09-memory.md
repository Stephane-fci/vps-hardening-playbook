# Step 9: Memory Management

---

## What

Configure swap, set swappiness for SSD, and ensure the system has a safety net before the OOM (Out Of Memory) killer gets involved.

## Why

On Feb 9, 2026, an OpenClaw instance was killed by the Linux OOM killer because the VPS had zero swap configured. 341 messages of work were lost instantly with no recovery. The OOM killer doesn't save state — it terminates the largest process immediately.

Swap gives the system breathing room. When RAM fills up, the kernel moves less-used pages to disk instead of killing processes. It's slower, but alive is better than dead.

## How

### 9.1 — Check current swap

```bash
swapon --show
free -h
```

If swap exists and is large enough (4GB+ for an 8GB VPS), skip to verification.

### 9.2 — Create swap file

```bash
# Create 4GB swap file
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make persistent
echo "/swapfile none swap sw 0 0" >> /etc/fstab
```

**Sizing rule of thumb:**
- 4-8GB RAM → 4GB swap
- 16GB+ RAM → equal to RAM or at least 4GB
- SSD vs HDD doesn't matter much at these sizes

### 9.3 — Set swappiness

Already done in Step 8 (sysctl config sets `vm.swappiness = 10`). Verify:
```bash
sysctl vm.swappiness
# Should show 10
```

Swappiness of 10 means: prefer RAM, only use swap when RAM is nearly full. Good for SSDs.

## Verify

```bash
swapon --show
# Should show your swap file with correct size

free -h
# Should show swap in the Swap line

sysctl vm.swappiness
# Should show 10

grep swap /etc/fstab
# Should show /swapfile entry
```

---

## Checklist

- [ ] Swap exists (4GB+ for 8GB RAM)
- [ ] Swap is in /etc/fstab (persists across reboots)
- [ ] Swappiness set to 10
