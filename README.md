# Arr Stack

> An opinionated Docker Compose stack for deploying the Servarr ecosystem with a focus on simplicity, consistency, and long-term maintainability.

This repository provides a clean, production-ready Docker Compose configuration for deploying the Servarr ecosystem.

Originally based on **arr-new** by Automation Avenue, this fork focuses on improving the overall architecture rather than expanding the number of services. The result is a compose file that is easier to understand, easier to maintain, and suitable for both new and experienced self-hosters.

Although the stack has been developed and tested primarily on Docker running inside an **unprivileged Proxmox VE LXC**, it is not tied to Proxmox. The same compose file can be deployed on any Linux host capable of running Docker Compose, including Debian, Ubuntu, virtual machines, and bare-metal Docker installations.

---

# Why this fork?

There are many Arr Stack repositories available today. Most work well, but over time many have accumulated inconsistent storage layouts, duplicated configuration, deployment-specific modifications, or unnecessary complexity.

This project takes a different approach.

Rather than adding every available application or feature, it focuses on providing a clean foundation built around modern Docker practices, predictable storage layouts, and sensible defaults. The objective is to make the stack easy to deploy, easy to understand, and easy to customise without sacrificing flexibility.

---

# Design Principles

This project follows a few simple principles.

- Keep the compose file easy to understand.
- Use a single, consistent `/data` mount across media applications.
- Optimise for hardlinks and atomic moves.
- Minimise duplicated configuration.
- Stay close to upstream Servarr recommendations.
- Support both generic Docker hosts and Proxmox VE deployments.

---

# Included Applications

The stack currently includes:

- Radarr
- Sonarr
- Lidarr
- Bazarr
- Prowlarr
- qBittorrent
- Jellyfin
- FlareSolverr *(optional)*

Each application is configured using the same design philosophy, resulting in a consistent deployment that's easy to extend and maintain.

---

# Architecture

The compose file intentionally presents the same filesystem layout to every media application.

```text
                    Docker Host

             (Any Linux distribution)

                       │
                       ▼

            Host Media Storage Location

         Examples:
         • /mnt/data1
         • /srv/media
         • /data

                       │
               Docker Bind Mount
                       │

              Host Path  ─────►  /data

                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Radarr         Sonarr         Lidarr
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                  qBittorrent
                       │
                       ▼
                    Jellyfin
```

Every media application shares the same `/data` mount.

Because downloads and media libraries reside on the same filesystem, Servarr applications can create hardlinks and perform atomic moves instead of copying files. This reduces disk usage, speeds up imports, and follows the storage recommendations from the Servarr and TRaSH Guides communities.

---

# Filesystem Layout

Inside the containers, every application expects the following directory structure.

```text
/data
├── media
│   ├── movies
│   ├── tv
│   ├── music
│   ├── anime
│   ├── books
│   └── audiobooks
│
└── torrents
    ├── movies
    ├── tv
    ├── music
    ├── anime
    ├── books
    ├── audiobooks
    └── incomplete
```

The compose file maps a directory from the Docker host into `/data` inside every container.

By default, the compose file uses:

```yaml
- /mnt/data1:/data
```

Update the **host path** to match your environment before starting the stack.

Examples:

```yaml
# Proxmox VE LXC
- /mnt/data1:/data

# Debian / Ubuntu
- /srv/media:/data

# Generic Linux
- /data:/data
```

Only the **host path** should be changed.

The **container path (`/data`) should remain unchanged**, as every application in the stack expects to use this location.

Example Proxmox VE deployment:

```text
Host
└── /mnt/pve/media2/arrstack
        │
        ▼
Docker Host
└── /mnt/data1
        │
        ▼
Containers
└── /data
```

---

# Quick Start

Clone the repository.

```bash
git clone https://github.com/<your-username>/<repository>.git
cd <repository>
```

Before starting the stack, review the bind mounts in `docker-compose.yaml` and update the **host paths** to match your storage layout.

Deploy the stack:

```bash
docker compose up -d
```

Verify the deployment:

```bash
docker compose ps
```

---

# Next Steps

Your Arr Stack is now deployed.

Continue with the guide below to complete the initial setup and connect all services together.

- 📖 **[Initial Configuration](docs/INITIAL_CONFIGURATION.md)**

Additional guides will be added as the project evolves.

---

# Credits

This project is based on the excellent **arr-new** repository by Automation Avenue.

Rather than replacing the original project, this repository builds upon it with a cleaner Docker Compose configuration, improved storage layout, and an architecture focused on long-term maintainability and Docker best practices.

Original project:

https://github.com/automation-avenue/arr-new