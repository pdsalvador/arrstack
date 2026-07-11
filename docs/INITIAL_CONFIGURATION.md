# Initial Configuration

This guide walks through the initial configuration of the Arr Stack
after the containers have been deployed.

The goal is to configure the applications in the correct order so they
can communicate with one another and take advantage of hardlinks and
atomic moves.

> **Important**
>
> Throughout this guide, all filesystem paths refer to paths **inside
> the Docker containers**.
>
> The host path configured in `docker-compose.yaml` (for example
> `/mnt/data1`, `/srv/media`, or `/data`) is bind-mounted to:
>
> ``` text
> /data
> ```
>
> Inside the containers, always use `/data`.

------------------------------------------------------------------------

# Configuration Order

Configure the applications in the following order:

1.  qBittorrent
2.  Prowlarr
3.  Radarr
4.  Sonarr
5.  Lidarr
6.  Bazarr
7.  Jellyfin

Each application builds upon the previous one, so following this order
reduces repeated configuration.

------------------------------------------------------------------------

# Understanding the Filesystem

Every media application shares the same container path:

``` text
/data
├── media
└── torrents
```

Downloads are written to `/data/torrents` by qBittorrent.

The Arr applications then import completed downloads into `/data/media`
using hardlinks whenever possible.

Because both directories exist on the same filesystem, imports are
nearly instantaneous and no duplicate disk space is consumed.

------------------------------------------------------------------------

# Configure qBittorrent

qBittorrent acts as the download client for the stack and should be
configured first.

After the first startup, retrieve the temporary administrator password:

``` bash
docker logs qbittorrent
```

Once logged in:

-   Change the administrator password.
-   Set the **Default Save Path** to `/data/torrents`.
-   Enable Automatic Torrent Management.
-   Configure category-based downloads.
-   Enable subcategories and category paths.

Create the following categories:

``` text
Category    Save Path
--------    ---------
movies      movies
tv          tv
music       music
anime       anime
books       books
```

------------------------------------------------------------------------

# Configure Prowlarr

Prowlarr manages indexers centrally and distributes them to the Arr
applications.

Configure:

-   Authentication
-   qBittorrent as the download client
-   Your preferred indexers

The remaining Arr applications will later be connected back to Prowlarr
using their API keys.

------------------------------------------------------------------------

# Configure Radarr

Configure:

-   Root Folder: `/data/media/movies`
-   Enable **Use Hardlinks instead of Copy**
-   Add qBittorrent as the download client
-   Category: `movies`

------------------------------------------------------------------------

# Configure Sonarr

Configure:

-   Root Folder: `/data/media/tv`
-   Enable **Use Hardlinks instead of Copy**
-   Add qBittorrent as the download client
-   Category: `tv`

------------------------------------------------------------------------

# Configure Lidarr

Configure:

-   Root Folder: `/data/media/music`
-   Add qBittorrent as the download client
-   Category: `music`

------------------------------------------------------------------------

# Configure Bazarr

Connect Bazarr to:

-   Radarr
-   Sonarr

Then configure:

-   Language Profiles
-   Subtitle Providers

------------------------------------------------------------------------

# Configure Jellyfin

Create your administrator account and add media libraries.

Recommended library paths:

``` text
Movies     /data/media/movies
TV Shows   /data/media/tv
Music      /data/media/music
```

------------------------------------------------------------------------

# Verification

Once complete, verify:

-   qBittorrent downloads into `/data/torrents`
-   Prowlarr can reach your download client
-   Radarr, Sonarr, and Lidarr successfully import media
-   Hardlinks are being created instead of copied
-   Jellyfin detects your libraries correctly

If all of the above succeeds, the stack is fully configured and ready
for use.
