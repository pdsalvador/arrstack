# Media Workflow

This document explains what happens behind the scenes when requesting media using the Arr Stack.

Rather than focusing on configuration, this guide follows the complete lifecycle of a media request, from the moment a movie or episode is added until it becomes available in Jellyfin.

Understanding this workflow makes it much easier to troubleshoot issues, optimise your setup, and understand the role of each application.

---

# High-Level Workflow

```text
User
 │
 ▼
Radarr / Sonarr / Lidarr
 │
 ▼
Prowlarr
 │
 ▼
Indexer
 │
 ▼
qBittorrent
 │
 ▼
Downloads
 │
 ▼
Arr Import
 │
 ▼
Media Library
 │
 ▼
Jellyfin
```

Each application performs one specific task before handing responsibility to the next.

---

# Step 1 — Requesting Media

The workflow begins when a user requests new media.

For example:

- Add a movie in Radarr
- Add a TV series in Sonarr
- Add an artist in Lidarr

At this stage nothing has been downloaded.

The application simply records what media you want and begins searching for available releases.

```text
User
 │
 ▼
Radarr
```

---

# Step 2 — Searching Indexers

The Arr application sends a search request to Prowlarr.

Prowlarr then queries every enabled indexer.

```text
Radarr
    │
    ▼
Prowlarr
    │
    ├── Indexer A
    ├── Indexer B
    └── Indexer C
```

Using Prowlarr provides several advantages:

- Centralised indexer management
- Shared indexers across all Arr applications
- One place to update credentials
- Less duplicated configuration

---

# Step 3 — Selecting a Release

Prowlarr returns all matching releases.

The Arr application evaluates each release using:

- Quality Profiles
- Custom Formats
- Release Restrictions
- Preferred Words
- Availability

Once the best release has been identified, it is sent to qBittorrent.

```text
Prowlarr

↓

Radarr

↓

Best Release Selected
```

---

# Step 4 — Downloading

Radarr instructs qBittorrent to begin downloading.

The download is placed inside:

```text
/data/torrents
```

For example:

```text
/data/torrents/movies
```

The Arr applications do not download files themselves.

Their responsibility ends once the download request has been submitted.

```text
Radarr

↓

qBittorrent

↓

Download Starts
```

---

# Step 5 — Download Completion

When qBittorrent finishes downloading, it notifies Radarr using its API.

Radarr now knows the download has completed and begins the import process.

```text
qBittorrent

↓

Download Complete

↓

Radarr
```

---

# Step 6 — Importing Media

This is where the filesystem layout becomes important.

Because both locations exist under the same `/data` mount:

```text
/data
├── torrents
└── media
```

Radarr can create a hardlink instead of copying the file.

Instead of this:

```text
Torrent File

↓

Copy

↓

Movie Library
```

it performs:

```text
Torrent File
      │
      ├──────────────┐
      ▼              ▼

Torrent Folder   Movie Library
```

Both directory entries reference the same data on disk.

Benefits include:

- Instant imports
- No duplicate storage
- Continued torrent seeding
- Reduced disk activity

---

# Step 7 — Organising the Library

Once imported, the Arr application organises the media library.

Examples include:

- Renaming files
- Creating folders
- Moving subtitles
- Importing metadata
- Cleaning up empty directories

The download remains in the torrent folder while the organised library appears under:

```text
/data/media
```

---

# Step 8 — Jellyfin Library Scan

Jellyfin continuously monitors the media library.

When new media appears, it scans the directory and adds the content to its database.

Jellyfin is completely independent of the download process.

It simply serves whatever exists inside the media library.

```text
Media Library

↓

Jellyfin Scan

↓

Available to Watch
```

---

# Complete Workflow

Putting everything together:

```text
User

↓

Radarr

↓

Prowlarr

↓

Indexers

↓

Best Release Selected

↓

qBittorrent

↓

Download Complete

↓

Hardlink Import

↓

Media Library

↓

Jellyfin

↓

Watch Movie
```

---

# Common Misconceptions

## Jellyfin downloads media

It does not.

Jellyfin only serves media that already exists in the library.

---

## qBittorrent organises media

It does not.

Its only responsibility is downloading files.

The Arr applications organise the media library.

---

## Prowlarr downloads torrents

It does not.

Prowlarr only searches indexers.

---

## Radarr copies files

Not when configured correctly.

With a shared `/data` mount, Radarr creates hardlinks instead of copying files.

---

# Related Documentation

- [Architecture](ARCHITECTURE.md)
- [Initial Configuration](INITIAL_CONFIGURATION.md)
- [Proxmox LXC UID/GID Mapping](PROXMOX_LXC_UID_GID_MAPPING.md)