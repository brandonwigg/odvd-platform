# ODVD Platform

A self-hostable platform that turns your own DVD collection into a faithful
1:1 digital recreation of the discs themselves — menus, extras, Easter eggs,
and navigation included — and lets you play them from anywhere on your home
network.

This is the platform half of the project. The format it reads and writes is
defined separately:

- **ODVD spec (format):** `odvd-spec` — the `.odvd` bundle format: manifest,
  per-cell segments, and the content-addressed asset store. The platform
  depends on that spec; it does not define the format.

DVD-only. Blu-ray is explicitly out of scope.

## What it is

- A server that compiles your DVDs into `.odvd` bundles and serves them.
- A library that looks like a shelf of boxes, not a streaming catalogue.
  A box set is one box, with its discs inside.
- A web player that recreates the DVD experience: First Play, warnings,
  trailers, menus, chapter selection, setup, extras, and DVD games, driven
  by a DVD virtual machine.
- A compile queue that does the slow work once, in the background, so
  playback is just serving static segments.

## What it is not

- Not Plex or Jellyfin. It does not turn a disc into a single movie file
  with a poster and a Play button. It does not strip menus.
- Not a streaming service. There is no catalogue to browse by genre, no
  recommendations, no watch-next row.
- Not a ripper for discs you do not own. This is for your own personal
  collection, for personal backup and playback.

## Vision

If it works on a DVD player, it works here. That is the bar for 1:1:

- First Play, FBI warnings, forced trailers, and studio logos play in order.
- Main menu, scene selection, setup, and extras work with remote-style
  navigation (arrow keys, Enter, Menu, Title), plus mouse/touch as a
  convenience.
- Every audio track, subtitle track, chapter, and angle is kept.
- Extras are reachable the way the disc intended, not dumped in a list.
- Hidden titles and Easter eggs are preserved, even when they are not
  listed anywhere.
- DVD games work (for example, the Harry Potter DVD game). If the disc
  uses VM state to track score and branching, the player must too.
- Multi-disc box sets are multi-disc: each disc is its own bundle, grouped
  under one box in the library.
- PAL and NTSC are preserved natively: 720x576 at 25 fps stays 720x576 at
  25 fps, 720x480 at 29.97 fps stays 720x480 at 29.97 fps. No frame-rate
  conversion, no audio pitch shift.

Quality target is **visually lossless**: nobody can tell the difference
visually, but the storage saving is obvious. Not archival bit-perfect, not
heavy compression.

## Architecture

The platform is a small set of services shipped as one compose file. It is
deliberately not forced into a single container: multiple services are fine
if they make the build cleaner, as long as deploy stays one command.

| Component | Job |
| --- | --- |
| Web app / library UI | Shelf, box sets, inserts/detail pages, compile queue, settings. |
| Player | Browser player: segment player + DVD VM + overlay renderer. Renders menus, subtitles, and button highlights. |
| Compiler worker | Takes an ISO or VIDEO_TS folder, parses it, transcodes cells, hashes segments into the CAS, and writes the `.odvd` manifest. Runs in the background. |
| Asset store (CAS) | Content-addressed segment store. Identical segments (same preview, logo, or warning on many discs) are stored once and referenced by hash, with reference counting. |
| SQLite | Library database: discs, box sets, manifests, users, per-user resume, compile jobs. |
| Auth | Single admin in v1. Multi-user logins in v2 (see below). |

See [ARCHITECTURE.md](ARCHITECTURE.md) for services, data flow, storage
layout, and the API sketch.

### Video and audio

These are library-level choices, not per-disc, so dedup keeps working
(identical segments hash identically only under identical settings).

- **Video:** AV1 (default), H.264 (fast / compatible), H.265 (opt-in).
  AV1 costs more compute at compile time, once only, and gives the best
  space saving at visually lossless quality. H.264 is the fastest compile
  and plays everywhere. H.265 is available for households that prefer it,
  but it is not the default because browser support is inconsistent.
- **Audio:** keep AC-3 by default, Opus as an opt-in conversion.
  Because browsers do not reliably play AC-3 (Safari can in MP4;
  Chrome/Firefox generally cannot), the compiler also generates a
  lightweight Opus proxy at compile time just for browser playback.
  The AC-3 master is still kept. Choosing the Opus opt-in means dropping
  the AC-3 master entirely and keeping only Opus, to save space.
- **Subtitles:** DVD bitmap subtitles are preserved as timed overlays,
  not OCR'd to text.

## Deployment model

Target audience: homelab folks who already run Plex or a home server.
Install difficulty target is Plex/Kodi level: a compose file and a
first-run wizard, no command-line work for normal use.

- **Primary:** Unraid, via an Unraid template.
- **Then:** generic Docker (same compose file) and Synology.
- **One compose file, multiple services inside it.** One command to start.
- **Volumes:**
  - `input` — ISOs or VIDEO_TS folders to compile (upload via web UI, or
    drop files in).
  - `library` — compiled `.odvd` manifests, box-set grouping, metadata.
  - `assets` — the content-addressed segment store.
  - `config` — settings, SQLite database, logs.
- **Network:** LAN-only by default, no public exposure. Remote access via
  Tailscale is the recommended path (direct traffic, no extra transcoding
  on the wire). A Cloudflare Tunnel hostname is possible but not the
  recommended default.
- **Releases:** versioned images on GHCR. Watchtower auto-pull is
  optional. The `.odvd` format is versioned separately, and old bundles
  keep playing after a platform update.
- **Hardware:** x86_64 primary, ARM64 best-effort. AV1 encoding is slow
  on low-end hardware — it queues overnight. Intel QuickSync helps for
  H.264/H.265.

Do not expose the platform directly to the internet. It is a home
service; keep it behind your LAN or your tailnet.

## First-run wizard

Under five minutes, sensible defaults already filled in:

1. Storage paths (input, library, assets, config).
2. Video codec: AV1 (default) / H.264 / H.265.
3. Audio: keep AC-3 (default) / convert to Opus.
4. Resume: on (default) / off.
5. Metadata: disc-first, user uploads, online databases opt-in (off by
   default; the collection works fully offline).
6. Admin login for v1.

## User journey

1. **Get a disc on the server.** Insert a disc into a USB drive passed
   through to the server, or upload an ISO via the web UI. It does not
   matter which — both end up as an ISO on the server and go through the
   same pipeline. (Rip a protected disc to an ISO first with a tool like
   MakeMKV Backup; the platform starts from the decrypted ISO.)
2. **Compile queue.** The disc goes into the background queue. The
   compiler parses the IFOs, transcodes each cell once, hashes segments
   into the CAS (duplicates are stored once), and writes the manifest.
   You can keep watching while it compiles.
3. **Library shelf.** The finished disc appears as a box on the shelf.
   A box set appears as one box, with its discs inside. Failed or partial
   compiles do not appear here at all; they live in a separate
   Needs Attention folder with the error and a retry button.
4. **Open the box.** Clicking a title opens the box to its inserts /
   detail page: cover, disc list, notes. For a TV DVD, the detail page
   lists its discs (for example, 4 discs) to choose from.
5. **Play.** Hitting Play inserts the disc: First Play, warnings,
   trailers, then the menu, exactly as the disc intended. Navigation is
   remote-style. Resume is on by default and toggleable — when on, the
   player picks up where you left off; when off, it always starts from
   First Play.

## Metadata

Priority order, always:

1. What the disc itself says (title, structure, chapters).
2. User uploads and designations (cover art, titles, notes).
3. Online databases, opt-in only, as an enhancement.

The collection works fully offline. Online metadata never overwrites what
the disc says or what the user set.

## Multi-user plan

Multi-user logins are essential, and they are in the plan 100% — they
just land in v2, not v1.

- **v1:** single admin. The database is designed for multi-user from day
  one (users table, per-user resume, per-user library access), so v2 is
  not a painful migration.
- **v2:** logins for household, friends, and family. Each user gets their
  own resume position per disc and their own access to the library.
  Roles distinguish household members from guests.

Do not design v1 shortcuts that assume there is only ever one user.

## Jellyfin / Plex plugin note

The standalone player is primary. Jellyfin and Plex cannot run a DVD
virtual machine — they expect a plain video file — so a plugin cannot be
the player.

A plugin, if built later, is a thin launcher: it lists your `.odvd`
discs inside Jellyfin or Plex and hands off to the ODVD player (or embeds
the web player). It does not replace the player, and it does not try to
shoehorn menus into their player model.

## Creator vs player

Creating `.odvd` files and playing them are two separate jobs, both part
of this project:

- **Creator:** the compiler worker (and, eventually, a standalone
  command-line tool) that turns a DVD into a `.odvd` bundle.
- **Player:** this platform — library, server, and web player — that
  consumes `.odvd` bundles.

They share the spec as their contract. You can compile on one machine
and play on another, or do both on the same server.

## Roadmap

- **v0.1 — Spec.** `odvd-spec` v0.1: manifest, segment format, CAS and
  hash rules, VM command set, versioning.
- **Parser prototype.** Parse 3 real DVDs (different studios/menu
  styles), dump titles, menus, buttons, and hidden titles, and prove the
  manifest can represent them.
- **MVP player.** Library shelf, box sets, inserts/detail pages, compile
  queue, web player with VM, resume, AC-3 + Opus proxy playback.
- **v2 — Multi-user.** Logins, per-user resume and access, household /
  guest roles.
- **Later, if wanted:** Jellyfin/Plex launcher plugin, hardware-encode
  profiles, additional metadata sources.

## Contributing

This project is in vision and spec stage. No application code yet.
Decisions are recorded in [ARCHITECTURE.md](ARCHITECTURE.md) (decisions
log) and in the `odvd-spec` repo. If a decision in either repo conflicts,
the spec wins on format questions and this repo wins on platform
questions — flag the conflict rather than silently picking a side.
