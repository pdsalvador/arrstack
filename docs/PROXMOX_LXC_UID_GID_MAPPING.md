# Proxmox LXC UID/GID Mapping

This guide explains how UID/GID mapping works in **unprivileged Proxmox VE LXC containers**, why permission issues occur when mounting host directories, and how to correctly expose host-owned files to Docker containers running inside an LXC.

This guide assumes you're already familiar with the overall architecture of this project. If you're new to the stack, it's recommended to read the Architecture guide first.

---

# Why does this matter?

One of the most common issues when running Docker inside an **unprivileged** LXC container is seeing files owned by:

```text
nobody:nogroup
```

or receiving errors such as:

```text
Permission denied
```

This typically happens when media libraries or bind mounts are owned by a normal user on the Proxmox host, but the LXC container cannot map those ownership IDs correctly.

Understanding how UID/GID mapping works makes these issues much easier to diagnose and resolve.

---

# Understanding Unprivileged Containers

An unprivileged container does **not** use the host's user IDs directly.

Instead, Proxmox translates every user and group ID inside the container into a different range on the host.

For example:

```text
Container UID      Host UID

0 (root)    ───►   100000
1000         ───►   101000
1001         ───►   101001
```

This translation is called **UID/GID mapping**.

The primary purpose is security.

If an attacker gains root access inside the container, they are **not** root on the Proxmox host. They are simply another unprivileged user with a high-numbered UID.

---

# The Three Layers

The easiest way to understand the mapping is to think of three separate layers.

```text
Host
│
├── UID 1000 (sysad)
│
▼

Unprivileged LXC
│
├── UID 1000
│
▼

Docker Container
│
├── PUID=1000
└── PGID=1000
```

Although each layer appears to use **UID 1000**, they are not automatically the same user.

Without an explicit mapping, the LXC translates container UID 1000 into host UID 101000.

If the mounted files are actually owned by host UID 1000, the container cannot match the ownership and instead displays:

```text
nobody:nogroup
```

---

# Default UID Mapping

By default, an unprivileged container maps every UID into a shifted range.

```text
Container UID      Host UID

0-65535    ───►   100000-165535
```

For example:

```text
Container

root (0)

↓

Host

100000
```

and

```text
Container

1000

↓

Host

101000
```

This mapping works well for container isolation, but it prevents direct access to files owned by normal users on the Proxmox host.

---

# The Problem

Imagine your media library is owned by:

```text
Host

UID:1000
```

Your Docker containers are configured with:

```yaml
PUID: 1000
PGID: 1000
```

Inside the LXC, Docker correctly starts the containers as UID 1000.

However, Proxmox silently translates that UID to:

```text
Host UID 101000
```

The host filesystem still belongs to UID 1000.

Because those IDs do not match, Linux refuses write access.

The result is:

- Files appear as `nobody:nogroup`
- Applications cannot rename media
- Imports fail
- Hardlinks cannot be created
- Manual imports produce permission errors

---

# How ID Mapping Solves It

Rather than translating every UID, Proxmox allows individual IDs to be passed directly to the host.

Instead of:

```text
Container 1000

↓

Host 101000
```

we explicitly tell Proxmox:

```text
Container 1000

↓

Host 1000
```

Only that single UID is passed through.

Every other user continues using the normal translated range.

This preserves the security benefits of an unprivileged container while allowing one trusted user to access the mounted filesystem correctly.

---

# Configuring an ID Mapping

## Step 1

Allow the Proxmox host to map UID/GID 1000.

```bash
echo "root:1000:1" >> /etc/subuid
echo "root:1000:1" >> /etc/subgid
```

---

## Step 2

Edit the LXC configuration.

```bash
nano /etc/pve/lxc/<CTID>.conf
```

Add:

```text
lxc.idmap: u 0 100000 1000
lxc.idmap: g 0 100000 1000

lxc.idmap: u 1000 1000 1
lxc.idmap: g 1000 1000 1

lxc.idmap: u 1001 101001 64535
lxc.idmap: g 1001 101001 64535
```

This creates three mapping ranges:

1. UIDs 0-999 use the normal translated range.
2. UID 1000 maps directly to host UID 1000.
3. Remaining UIDs continue using translated IDs.

---

## Step 3

Restart the container.

```bash
pct stop <CTID>
pct start <CTID>
```

---

# Existing Files

Changing the mapping does **not** update ownership of existing files.

Any files previously created with incorrect ownership should be corrected from the **Proxmox host**.

For media libraries:

```bash
chown -R 1000:1000 /path/to/media
chmod -R u=rwX,g=rwX,o=rX /path/to/media
```

For Docker application data:

```bash
pct stop <CTID>
pct mount <CTID>

chown -R 1000:1000 /var/lib/lxc/<CTID>/rootfs/opt/arrstack

pct unmount <CTID>
pct start <CTID>
```

---

# Verifying the Mapping

On the Proxmox host:

```bash
ls -ln /path/to/media
```

Expected:

```text
1000 1000
```

Inside the LXC:

```bash
ls -ln /mnt/data1
```

Expected:

```text
1000 1000
```

Inside a Docker container:

```bash
docker exec radarr ls -ln /data
```

Expected:

```text
1000 1000
```

All three layers should now report the same ownership.

---

# Common Symptoms

Typical signs of an incorrect mapping include:

- Files appear as `nobody:nogroup`
- Manual imports fail
- "Permission denied" errors
- Failed renames
- Hardlinks are not created
- Containers cannot write to mounted directories

---

# Related Documentation

- [Architecture](ARCHITECTURE.md)
- [Initial Configuration](INITIAL_CONFIGURATION.md)
