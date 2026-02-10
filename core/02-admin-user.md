# Step 2: Create Admin User

---

## What

Create a non-root user with sudo privileges and SSH key access. This user replaces root for all administration.

## Why

Running as root means every command has unlimited power. If an agent runs a bad command, or if an attacker gains access, root = game over. A sudo user has the same capabilities when needed, but accidental damage is limited to user-level by default. It's the difference between walking around with a loaded gun and keeping it in a holster.

## How

### 2.1 — Create the user

```bash
export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
adduser --disabled-password --gecos "Admin User" admin
```

Replace `admin` with whatever username makes sense (e.g., `stephane`, `deployer`).

`--disabled-password` means no password login. SSH key only.

### 2.2 — Add to sudo group

```bash
usermod -aG sudo admin
```

### 2.3 — Enable passwordless sudo

Since the user has no password (key-only auth), they need passwordless sudo:

```bash
echo "admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin
chmod 440 /etc/sudoers.d/admin
visudo -cf /etc/sudoers.d/admin  # Validate syntax
```

### 2.4 — Copy SSH key

```bash
mkdir -p /home/admin/.ssh
cp /root/.ssh/authorized_keys /home/admin/.ssh/
chown -R admin:admin /home/admin/.ssh
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
```

### 2.5 — Test login

**Do NOT disable root yet.** First verify the new user works:

```bash
# Test sudo
su -s /bin/bash -c "sudo whoami" admin
# Should output: root
```

Tell your human to test SSH as the new user from their machine:
```bash
ssh admin@[SERVER_IP]
```

If they can't test right now, proceed but be extra careful in Step 3.

## Verify

```bash
id admin
# Should show: uid=..., groups=...sudo...

cat /etc/sudoers.d/admin
# Should show: admin ALL=(ALL) NOPASSWD:ALL

ls -la /home/admin/.ssh/authorized_keys
# Should exist, owned by admin, mode 600
```

## Traps

**🪤 TRAP: PATH issues.** On some Ubuntu versions, `adduser` and `usermod` are in `/usr/sbin/` which may not be in PATH for automated scripts. Always `export PATH="/usr/sbin:/usr/bin:/sbin:/bin:$PATH"` first.

**🪤 TRAP: Home directory ownership.** If the home directory already exists (from a previous attempt), `adduser` won't change ownership. Run `chown -R admin:admin /home/admin` manually.

---

## Checklist

- [ ] Admin user created
- [ ] Added to sudo group
- [ ] Passwordless sudo configured and validated
- [ ] SSH key copied with correct ownership and permissions
- [ ] Sudo test passed (`sudo whoami` → `root`)
- [ ] Root login still works (backup — don't disable yet)
