# ODVD Platform — Architecture

Short technical architecture for the platform half of the ODVD project.
The format is defined in the `odvd-spec` repo; this document describes how
the platform uses it. No application code exists yet — this is the design
the code will follow.

## Services

Shipped as one compose file. Multiple services are allowed — do not force
everything into one container if separate services make the build cleaner.
One compose file, one command to start, is the deploy contract.

| Service | Responsibilities |
| --- | --- |
| `web` | Web app + library UI + API. Serves the shelf, box sets, inserts/detail pages, compile queue UI, settings, and first-run wizard. Serves `.odvd` manifests and segments (HTTP range requests) to the player. Owns SQLite. |
| `worker` | Compiler. Watches the input volume and the compile queue, parses discs, transcodes cells, hashes segments into the CAS, writes manifests. Runs one job at a time by default; queue depth is visible in the UI. |
| `player` | Client-side, not a server service: the browser app (segment player + DVD VM + overlay renderer). It talks to `web` over HTTP only. No server-side per-playback transcoding in the normal path. |

There is no separate database service; SQLite lives in the `config`
volume. There is no separate transcoding service; `worker` owns ffmpeg.

### Why the worker is separate

Compiling is slow (AV1 especially) and CPU-heavy. If it ran inside the
web process, a compile would starve playback and the UI. A separate worker
means the library stays responsive while discs compile in the background,
and the queue can be paused, retried, or run overnight.

## Data flow

```
ISO / VIDEO_TS folder (input volume)
  -> worker: libdvdread parse (IFO -> titles, PGCs, menus, buttons,
     hidden titles, VM pre/post commands)
  -> worker: per-cell transcode (ffmpeg)
        video: AV1 default / H.264 / H.265, visually lossless
        audio: keep AC-3 master + Opus proxy for browser playback
               (Opus opt-in: drop AC-3 master, keep only Opus)
        subtitles: bitmap subpictures preserved as timed overlays
  -> worker: hash each transcoded segment (SHA-256)
        hash already in CAS?  -> reference it, store nothing
        hash new?             -> store segment in CAS, refcount = 1
  -> worker: write .odvd manifest (JSON) into library volume
        manifest references segments by hash, never by path
  -> library: disc appears on shelf (box set = one box, discs inside)
  -> player: fetch manifest -> fetch segments by hash over HTTP range
        -> VM executes navigation -> overlay renderer draws menus,
           subtitles, and button highlights
```

Key properties of this flow:

- **Compile once, serve forever.** Playback is static file serving.
  There is no live transcoding per play.
- **Dedup happens at compile time, not at play time.** The CAS check is
  part of the worker; the player never knows or cares that a segment is
  shared.
- **The manifest is the only thing the player trusts for navigation.**
  Segment files carry no navigation information.
- **Failed compiles never reach the shelf.** A manifest is written only
  when the whole disc compiles. Partial output goes to the Needs Attention
  folder with the error and a retry action.

## Storage layout

Library and assets live together physically (same volume / same server)
but are logically separate. Never mix them in one directory.

```
/data
  /input            # ISOs or VIDEO_TS folders dropped in or uploaded
    /movie.iso
    /tv-show-disc1/VIDEO_TS/
  /library          # manifests + grouping + metadata (small files)
    /<disc-id>/
      manifest.json          # the .odvd bundle manifest
      cover.jpg              # user upload or online pull (optional)
      notes.md               # user designations (optional)
    /<box-set-id>/
      set.json               # box-set grouping: discs in order
      manifest.json          # per-disc manifests live under the set
      /disc1/manifest.json
      /disc2/manifest.json
  /assets           # content-addressed segment store (large files)
    /<sha256[0:2]>/<sha256>.mp4
    /<sha256[0:2]>/<sha256>.opus   # Opus audio proxies, where applicable
    cas-index.db    # hash -> refcount, size, codec, created-by
  /config
    library.db      # SQLite: discs, sets, users, resume, jobs
    settings.json
    logs/
```

Notes:

- A `.odvd` bundle is a manifest plus references to the CAS, not a
  self-contained file with video embedded. Embedding the video would
  defeat dedup.
- `cas-index.db` is derived state: it can be rebuilt by scanning
  `/assets` and re-reading manifests. The manifests are the source of
  truth for what is referenced.
- Deleting a disc decrements refcounts; it never deletes a segment still
  referenced by another manifest. A periodic garbage-collection pass
  removes zero-refcount segments.
- Resume state lives in SQLite, per user per disc (see multi-user), not
  in the manifest — the manifest is the disc, resume is the viewer.

## API sketch

All JSON over HTTP, served by `web`. Auth: v1 single admin session;
v2 adds per-user sessions. This is a sketch, not a contract — the
implementation may rename endpoints, but the resource split should stay.

### Library

- `GET /api/library` — shelf: boxes and standalone discs, with cover,
  title, disc count, compile status. Failed/partial excluded.
- `GET /api/library/:id` — inserts/detail page: box or disc, discs
  inside (for a set), metadata source for each field.
- `GET /api/library/needs-attention` — failed/partial compiles with
  error, stage failed at, retry action.
- `POST /api/library/:id/metadata` — user uploads/designations (cover,
  title override, notes).
- `DELETE /api/library/:id` — remove disc; decrement CAS refcounts.

### Compile queue

- `POST /api/compile` — enqueue an input (ISO path or VIDEO_TS folder).
  Body: input path, codec profile override (optional, library default
  otherwise), audio mode (default keep AC-3).
- `GET /api/compile` — queue: pending, running, done, failed, with
  progress per job (stage, current cell, percent).
- `POST /api/compile/:job/retry` — retry a failed job.
- `POST /api/compile/:job/cancel` — cancel a queued job.

### Playback

- `GET /api/play/:disc/manifest` — the disc manifest (VMG, titles,
  PGCs, buttons, segment hashes).
- `GET /api/assets/:hash` — segment bytes, HTTP range requests supported.
  The player fetches segments by hash, never by disc path.
- `GET /api/play/:disc/resume` — per-user resume position (VM state:
  title/PGC/cell/time). Absent when resume is off or never played.
- `POST /api/play/:disc/resume` — save position on pause/stop/exit.
  Resume is toggleable, on by default.
- `POST /api/play/:disc/reset-resume` — clear resume, start from First
  Play next time.

### Auth (v1 single admin; v2 multi-user)

- `POST /api/auth/login` — admin login, session cookie.
- `GET /api/auth/me` — current user, role.
- v2 (planned, schema designed now): `POST /api/users`, per-user
  library access, roles (household / guest). Resume and access are
  per-user from the start in the database schema, even though v1 exposes
  only one user.

## Multi-user design (v2, designed now)

- `users` table exists in v1 schema with a single admin row.
- Resume, recently played, and library access are keyed by
  `(user_id, disc_id)`, never by disc alone.
- Roles: household (full library) vs guest (subset, to be defined in v2).
- No v1 code path may assume a single global resume position.

## Decisions log

Locked vision decisions this architecture must not contradict. If a
decision here conflicts with `odvd-spec`, the spec wins on format
questions.

1. **DVD-only.** Blu-ray is out of scope.
2. **Audience:** homelab folks who already run Plex/home servers.
   Install target is Plex/Kodi level.
3. **1:1 bar:** if it works on a DVD player, it works here — including
   DVD games, multi-disc sets as multi-disc, PAL/NTSC preserved natively.
4. **Quality:** visually lossless. Not archival bit-perfect, not heavy
   compression.
5. **Virtual player:** segment player + DVD VM + overlay renderer. No
   physical-drive emulation, no MPEG-2 program-stream parsing, no sector
   seeks at play time.
6. **Video codec:** library-level choice, not per-disc (preserves dedup).
   AV1 default, H.264 fast/compatible, H.265 opt-in. AV1's higher encode
   cost is one-time, at compile time.
7. **Audio:** keep AC-3 by default, Opus as opt-in conversion. Browser
   playback uses a lightweight Opus proxy generated at compile time;
   Opus opt-in means dropping the AC-3 master entirely. Bitmap subtitles
   preserved as timed overlays.
8. **Library:** box set is one box with discs inside. Clicking a title
   opens the box to an inserts/detail page; Play plays the disc. TV DVD
   detail page lists its discs (e.g. 4). Resume toggleable, on by
   default. Failed/partial compiles hidden from the main library, in a
   separate folder.
9. **Metadata:** disc first, then user uploads/designations, then online
   databases (opt-in pull). Works fully offline.
10. **Library vs assets:** live together physically, logically separate.
    Library holds manifests; assets hold content-addressed segments.
    A duplicate asset in another library is not stored again — both
    manifests reference the same hash (reference-counted CAS).
11. **Creator vs player:** separate things, both part of the project.
    Standalone player is primary; a Jellyfin/Plex plugin, if built, is a
    thin launcher, not a player replacement.
12. **Multi-user:** essential, v2 is acceptable, but designed for from
    day one (per-user resume/access in the schema).
13. **Input:** disc insert vs ISO upload does not matter — both end as
    an ISO on the server through the same pipeline.
14. **Containers:** do not force one container. Multiple services via one
    compose file is fine.
15. **Repos:** two separate repos — `odvd-spec` (format) and
    `odvd-platform` (this repo). The spec is the contract between them.
16. **Deployment:** LAN + Tailscale recommended, no public exposure by
    default. Single compose file; volumes input/library/assets/config;
    Unraid first, then generic Docker/Synology.
