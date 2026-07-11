# Arr Stack

> An opinionated Docker Compose stack for the Servarr ecosystem with a
> focus on simplicity, consistency, and long-term maintainability.

A clean, production-ready Docker Compose configuration for deploying the
Servarr ecosystem. While the stack has been designed and tested
primarily on Docker running inside an **unprivileged Proxmox VE LXC**,
it runs anywhere Docker Compose is supported, including standard Debian
and Ubuntu hosts, virtual machines, and other Linux distributions.

------------------------------------------------------------------------

## Why this project?

There are countless Arr Stack examples available online. Most work well,
but many have gradually accumulated inconsistent bind mounts, duplicated
configuration, and deployment-specific tweaks that make them harder to
understand and maintain.

This project takes a different approach.

Instead of adding every possible service or feature, it focuses on
providing a clean foundation built around modern Docker practices, a
predictable storage layout, and sensible defaults. The goal is to make
the stack easy to deploy, easy to understand, and easy to maintain over
time.

------------------------------------------------------------------------

## Design Principles

-   Keep the compose file easy to read.
-   Use one consistent `/data` mount across media applications.
-   Optimise for hardlinks and atomic moves.
-   Minimise duplicated configuration.
-   Stay close to upstream Servarr recommendations.
-   Remain suitable for both generic Docker hosts and Proxmox VE
    deployments.

------------------------------------------------------------------------

## Included Applications

-   Radarr
-   Sonarr
-   Lidarr
-   Bazarr
-   Prowlarr
-   qBittorrent
-   Jellyfin
-   FlareSolverr (optional)

------------------------------------------------------------------------

## Architecture

``` text
                  Docker Host
      (Debian, Ubuntu, VM or Proxmox)

                  Host Storage
                       │
                       ▼
                  /mnt/data1
                       │
                       ▼
                Docker Bind Mount
                    /data
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

Every media application shares the same `/data` mount. Downloads and
libraries therefore exist on the same filesystem, allowing hardlinks and
atomic moves instead of slow file copies.

------------------------------------------------------------------------

## Recommended Directory Layout

``` text
/data
├── media
│   ├── movies
│   ├── tv
│   ├── music
│   ├── anime
│   ├── books
│   └── audiobooks
└── torrents
    ├── movies
    ├── tv
    ├── music
    ├── anime
    ├── books
    ├── audiobooks
    └── incomplete
```

Example Proxmox deployment:

``` text
Host
└── /mnt/pve/media2/arrstack
        │
        ▼
LXC
└── /mnt/data1
        │
        ▼
Docker
└── /data
```

On a standard Docker host, simply bind your preferred storage location
directly to `/data`.

------------------------------------------------------------------------

## Quick Start

Clone the repository:

``` bash
git clone https://github.com/<your-username>/<repository>.git
cd <repository>
```

Review the volume paths in `docker-compose.yaml`, then start the stack:

``` bash
docker compose up -d
```

Verify the deployment:

``` bash
docker compose ps
```

------------------------------------------------------------------------

## Initial Configuration

Once the containers are running, a small amount of configuration is required before the applications can work together correctly.

The most important concept is that **every application must reference the same `/data` mount**. This ensures downloads and media libraries remain on the same filesystem, allowing Servarr applications to use hardlinks and atomic moves instead of copying files.

### Configure qBittorrent

Log in to the qBittorrent Web UI.

If this is your first time starting the container, retrieve the temporary administrator password from the container logs:

```bash
docker logs qbittorrent
```

Once logged in:

1. Navigate to **Tools → Options → Downloads**.
2. Set the **Default Save Path** to:

```text
/data/torrents
```

3. Under **Torrent Management**, configure the following options:

- Default Torrent Management Mode: **Automatic**
- When Torrent Category Changed: **Relocate Torrent**
- When Default Save Path Changed: **Switch affected torrents to Manual Mode**
- When Category Save Path Changed: **Switch affected torrents to Manual Mode**

4. Enable:

- Use Subcategories
- Use Category Paths in Manual Mode

Save the configuration before continuing.

### Create Categories

Next, create the download categories that will be used by the Arr applications.

Navigate to **Categories**, then create the following:

| Category | Save Path |
| -------- | --------- |
| movies | movies |
| tv | tv |
| music | music |
| anime | anime |
| books | books |

Because the default save path is already `/data/torrents`, each category will automatically resolve to:

```text
/data/torrents/movies
/data/torrents/tv
/data/torrents/music
/data/torrents/anime
/data/torrents/books
```

### Configure Radarr

Open **Settings → Media Management**.

Set the root folder to:

```text
/data/media/movies
```

Enable **Use Hardlinks instead of Copy**.

Next, navigate to **Settings → Download Clients** and add qBittorrent using:

- Host: `qbittorrent`
- Port: `8080`
- Category: `movies`

The category **must** match the category created earlier in qBittorrent.

### Configure Sonarr

Open **Settings → Media Management**.

Set the root folder to:

```text
/data/media/tv
```

Enable **Use Hardlinks instead of Copy**.

Add qBittorrent as the download client using:

- Host: `qbittorrent`
- Port: `8080`
- Category: `tv`

### Configure Lidarr

Open **Settings → Media Management**.

Set the root folder to:

```text
/data/media/music
```

Add qBittorrent as the download client using:

- Host: `qbittorrent`
- Port: `8080`
- Category: `music`

### Configure Prowlarr

Prowlarr acts as the central indexer manager for the stack.

First, add qBittorrent under **Settings → Download Clients**.

Then connect Radarr, Sonarr, and Lidarr under **Settings → Apps** using each application's API key.

Use the following internal container addresses:

```text
Prowlarr:   http://prowlarr:9696

Radarr:     http://radarr:7878
Sonarr:     http://sonarr:8989
Lidarr:     http://lidarr:8686
```

Because every service shares the same Docker network, the container name is used instead of an IP address.

### Configure Jellyfin

During the initial setup wizard, create your administrator account.

When adding media libraries, use the following paths:

Movies

```text
/data/media/movies
```

TV Shows

```text
/data/media/tv
```

Music

```text
/data/media/music
```

Additional libraries can be added following the same directory structure.

### Why this layout?

The stack intentionally uses a single shared `/data` mount.

```text
/data
├── media
└── torrents
```

Downloads are initially saved into `/data/torrents`.

Once imported, the Arr applications create hardlinks into `/data/media` instead of copying the files.

This results in:

- Instant imports (atomic moves)
- No duplicate disk usage
- Faster library updates
- A consistent directory structure across every application
------------------------------------------------------------------------

## Screenshots

*Coming soon.*

-   Docker Compose
-   qBittorrent
-   Radarr
-   Sonarr
-   Jellyfin

------------------------------------------------------------------------

## Credits

This project is based on the excellent **arr-new** repository by
Automation Avenue.

Rather than replacing the original project, this repository builds upon
it with a simplified architecture, cleaner Docker Compose configuration,
and improvements aimed at long-term homelab deployments, especially
those running on Proxmox VE.

Original project:

https://github.com/automation-avenue/arr-new
