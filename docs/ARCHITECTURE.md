# Architecture

This document explains the architecture of the Arr Stack, the reasoning behind its design, and how the individual applications work together.

Unlike the deployment or configuration guides, this document focuses on **why** the stack is structured the way it is. Understanding these concepts will make it much easier to customise the stack, troubleshoot issues, and extend it with additional services.

---

# Overview

At a high level, the stack consists of four major components:

```text
                Internet
                    │
                    ▼
             Prowlarr (Indexers)
                    │
                    ▼
      Radarr • Sonarr • Lidarr
                    │
                    ▼
               qBittorrent
                    │
                    ▼
              Media Storage
                    │
                    ▼
                Jellyfin
```

Each application has a dedicated responsibility.

- **Prowlarr** manages indexers.
- **Radarr, Sonarr, and Lidarr** manage media requests and library imports.
- **qBittorrent** downloads content.
- **Jellyfin** serves the completed media library.

Keeping each application focused on a single responsibility makes the stack easier to maintain and replace over time.

---

# Application Responsibilities

## Prowlarr

Prowlarr acts as the central indexer manager.

Instead of configuring indexers individually in Radarr, Sonarr, and Lidarr, they are configured once in Prowlarr and synchronised automatically using each application's API.

Benefits include:

- Centralised indexer management
- One place to update credentials
- Consistent indexer configuration
- Reduced duplication

---

## Radarr

Radarr manages movie libraries.

Its responsibilities include:

- Monitoring wanted movies
- Searching indexers through Prowlarr
- Sending downloads to qBittorrent
- Importing completed downloads
- Renaming and organising media

---

## Sonarr

Sonarr performs the same role as Radarr but for television series.

---

## Lidarr

Lidarr manages music libraries using the same workflow.

---

## qBittorrent

qBittorrent is responsible only for downloading content.

It should not organise media libraries.

Instead, downloads remain in the torrent directory while the Arr applications create hardlinks into the media library.

This separation ensures torrents continue seeding while the media library remains organised.

---

## Jellyfin

Jellyfin is the media server.

Its responsibility begins only after media has been imported successfully.

For this reason, Jellyfin only requires read access to the media library.

The compose file mounts `/data` as read-only to reduce the risk of accidental modifications.

---

# Storage Architecture

One of the most important design decisions in this project is using a **single shared filesystem**.

Every media application receives exactly the same bind mount.

```text
Host Storage
      │
      ▼
 Docker Bind Mount
      │
      ▼
      /data
```

Every application sees:

```text
/data
├── media
└── torrents
```

Because every container shares the same filesystem, no path translation is required between applications.

---

# Why a Single /data Mount?

Many Docker examples mount folders individually.

For example:

```text
/movies
/tv
/downloads
```

While functional, this introduces unnecessary complexity.

Different containers often see different paths, making configuration harder and preventing hardlinks.

Instead, this project exposes one consistent mount:

```text
/data
```

Every application uses exactly the same paths.

Examples:

```text
/data/media/movies

/data/media/tv

/data/torrents/movies
```

This consistency simplifies configuration and follows Servarr and TRaSH Guide recommendations.

---

# Hardlinks and Atomic Moves

A common misconception is that media should be moved after downloading.

Instead, the Arr applications create **hardlinks**.

```text
Torrent File
      │
      ├──────────────┐
      ▼              ▼

Torrent Folder   Media Library
```

Both directory entries reference the same underlying data on disk.

Benefits include:

- No duplicate disk usage
- Instant imports
- Continued torrent seeding
- Reduced disk activity

Hardlinks only work when both directories exist on the same filesystem.

This is one of the primary reasons every container shares the same `/data` mount.

---

# Docker Networking

Every container is attached to a dedicated Docker network.

Instead of communicating using IP addresses, applications communicate using Docker's built-in DNS.

Examples:

```text
http://qbittorrent:8080

http://prowlarr:9696

http://radarr:7878
```

Using container names provides several advantages:

- No static IP addresses
- Easier migrations
- Simpler configuration
- Improved portability

---

# Filesystem Layers

One concept that often causes confusion is that different filesystem paths exist at different layers.

For example:

```text
Host
└── /mnt/pve/media2/arrstack

        │

Docker Host
└── /mnt/data1

        │

Container
└── /data
```

Although these paths are different, they all reference the same storage.

Applications should always use the container path:

```text
/data
```

Only the Docker bind mount should be modified to suit the host environment.

---

# Docker Inside an Unprivileged Proxmox LXC

Running Docker inside an unprivileged Proxmox VE LXC introduces an additional layer of UID/GID translation.

Without an ID mapping, files owned by normal users on the Proxmox host may appear inside the container as:

```text
nobody:nogroup
```

This prevents the Arr applications from importing or renaming media.

For a detailed explanation of UID/GID mapping and how to resolve these permission issues, see:

- [Proxmox LXC UID/GID Mapping](PROXMOX_LXC_UID_GID_MAPPING.md)

---

# Media Workflow

The complete workflow looks like this:

```text
Movie Request
      │
      ▼
Prowlarr searches indexers
      │
      ▼
Radarr selects release
      │
      ▼
qBittorrent downloads
      │
      ▼
Download completes
      │
      ▼
Radarr imports using hardlinks
      │
      ▼
Media appears in Jellyfin
```

Every application performs one specific task.

This separation of responsibilities makes the stack modular and easier to troubleshoot.

---

# Design Principles

Every design decision in this repository is based on a small number of guiding principles.

- Keep the compose file simple.
- Avoid duplicate configuration.
- Use one consistent filesystem layout.
- Follow Servarr best practices.
- Optimise for hardlinks.
- Prefer container names over IP addresses.
- Minimise maintenance effort.
- Make the stack easy to understand before making it easy to customise.

These principles guide both the Docker Compose configuration and the accompanying documentation.