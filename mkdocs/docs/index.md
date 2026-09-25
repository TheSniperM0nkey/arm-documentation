# Automatic Ripping Machine Developer Documentation

## What is the Automatic Ripping Machine?

The Automatic Ripping Machine (ARM) is a headless application that watches for an optical disc being inserted, works out what kind of disc it is, and rips it automatically — no prompts, no babysitting.

## The Vision

The Automatic Ripping Machine (A.R.M.) should be an all in one solution to build your own media library from your physical media collection that can be set up and used by anyone that reads a quickstart guide.

Granularity of the setup is a secondary objective intended to be used by power users that want to dive a bit deeper into what it can do.

## What A.R.M. Does, End to End

1. A disc is inserted; ARM detects it via `udev`.
2. It identifies the disc type — movie/TV (Blu-ray or DVD), music CD, or plain data — and picks the matching workflow.
3. **Video** discs get metadata lookup (OMDb, with TMDB support added later as an alternative provider), are ripped with MakeMKV, and are queued for transcoding with HandBrake — optionally hardware-accelerated via Intel QuickSync or AMD VCE.
4. **Audio** CDs are ripped with `abcde`, tagged and matched against MusicBrainz, with album art pulled in automatically.
5. **Data** discs are backed up as ISO images.
6. The disc is ejected and the drive becomes available for the next one — ARM supports several drives ripping in parallel.
7. Progress, logs, and job history are all visible through a web UI, and ARM can push notifications out to things like Discord, Slack, ntfy, IFTTT, and Pushbullet via Apprise.

## A.R.M. v2

A.R.M. v2 is the entire current, actively maintained architecture, spanning every release from `2.0.0` up to today's `2.24.0`. It superseded the original v1, which was a simpler set of bash scripts for basic rip automation.

The defining moment was `2.0.0`\: a complete rewrite of the project in Python, which also let ARM run as a non-root user for the first time. Everything since has been built on top of that foundation.


### How the pieces fit together

- **Language/runtime:** Python, designed to run headless and as a non\-root user.
- **Packaging:** typically run via Docker.
- **Web UI (`armui`):** a Flask + Bootstrap application added in `2.1`, backed by SQLite. It handles job monitoring, log viewing, a searchable job database, a persistent settings page (writing back to `arm.yaml`), and — since `2.2` — login/authentication for multiple users.
- **Ripping tools:** MakeMKV and HandBrake for video, `abcde` for audio.
- **Metadata providers:** OMDb (original), with TMDB added later as an alternative for movies; MusicBrainz for CD identification.
- **Notifications:** Apprise as the common layer, fanning out to most major chat/push services.

A.R.M. v2 is now in maintenance and not in active development. Current and Future development is focused on the v3 release.

## A.R.M. v3

A.R.M. v3 is a ground up rewrite of the system to fix structural problems in v2 that a refactor couldn't solve, while keeping the same core promise: insert a disc, walk away, get a finished file.

### Who v3 is for
Single-admin homelab users. One person running ARM for their own household. It is not designed to be a shared service, a multi-tenant platform, or a commercial product.

The user base in practice includes many data-hoarders — people who rip to preserve the raw bits, not just to feed a streaming app. This shapes retention defaults (keep raw forever) and session semantics (re-transcode from raw is a first-class operation).

### Problems v3 Is Built to Solve

1. **Resource isolation:** In v2, ripping, UI, and transcoding share one process tree, so heavy work in one area starves the others. v3 separates these into services so concurrent rips and a transcode don't degrade the UI.
2. **Batch-rip resumability:** In v2, a power loss mid-batch discards progress with no recovery path. v3 checkpoints finely enough that completed work survives a crash and unfinished work re-queues automatically on restart.
3. **Sessions:** In v2 ripping and transcoding is conflated into one irreversible pipeline; v3 separates them so a ripped disc can be re-transcoded later without re-ripping.

### Design Principles

1. **Bits first, metadata second, transcode third:** The three stages are decoupled. A rip succeeds the moment the bits are safely on disk and recorded in the DB. Metadata enrichment and transcoding are independent downstream stages that can fail, retry, or be re-run without touching the raw.
2. **One service, one responsibility:** Every container has a single reason to exist:
    - The Ripper exists to turn a disc into bytes on disk.
    - The Backend exists to own state and speak to the internet.
    - The Transcode container exists to turn one raw into one output.
    - The UI exists to render state and take commands.
    - No service does "a little bit of the other guy's job."
3. **Backend is the single internet boundary:** The Ripper and Transcode containers never talk to the internet. All external calls (TMDB/OMDB/MusicBrainz/Apprise/webhooks) originate from the Backend. This means workers are simpler to run (no API keys, no outbound firewall holes), and external credentials live in exactly one place.

4. **Postgres is the source of truth; stdout is the source of logs:** Durable state lives in Postgres. Logs are structured JSON emitted to stdout and appended to a shared volume. We do not invent a third persistence mechanism for state, and we do not hide debug data behind a query language.
5. **Crash\-safe by default:** Every long-running operation is checkpoint-able. A worker crash must not discard completed sub-work. A stale "in-progress" row with no live worker is the signal for "re-queue me."
6. **No sacred cows:** This is a greenfield rebuild. Any assumption inherited from v2 is open for review. When in doubt, re-decide from first principles rather than preserve an old shape.

### Explicit Non-Goals for v3.0

Explicit non-goals for v3.0 — not features that are not done yet, but features we have actively decided **not** to pursue:

- **No TrueNAS / iX Systems support.** Not a supported target.
- **No v2 → v3 data migration.** v2 stays on its own tag; v3 starts from a clean schema.
- **No multi-tenancy / RBAC.** Target user is a single homelab hobbyist. One admin.
- **No Kubernetes / Helm.** Docker Compose is the only supported deploy surface.
- **No in-backend transcoding.** Transcode always runs in a dedicated ephemeral container.

### Success Criteria for v3.0

- Five discs queued across two drives complete without manual intervention, including a simulated power-cut mid-batch that resumes cleanly.
- A ripped disc can have a new session (transcode preset) applied from the UI months later without re-ripping.
- A single PR can change a protocol payload and its two endpoints (ripper side and backend side) atomically.
- A bug-reporter can download one log file scoped to one job ID and attach it to a GitHub issue.
- A fresh install on a new host reaches the login screen in under 5 minutes.

### Current Status / Roadmap

**Shipped in v3:**

- Service split into FastAPI backend, Vue UI, Postgres, one ripper container per drive, and an ephemeral per-job transcoder, orchestrated with Docker Compose.
- Migrated from SQLite to Postgres with async SQLAlchemy/Alembic.
- Ripper and UI rewritten as new codebases with a pytest suite and a backend statement-coverage policy.
- Sessions, rip presets, and transcode presets implemented and user-editable in the UI (including music-to-FLAC/MP3, data copy, and ISO dump).
- One-command install via `install.sh`, including TLS certs and per-drive service blocks.
- Notifications via Apprise, configured from the UI.
- GPU transcoding (Intel QSV / AMD VAAPI / NVIDIA NVENC) via an opt-in overlay.

**In progress / ahead:**

- Stabilizing the alpha toward a v3.0 release (signed images for every supported platform, CI-built release tags).
- Ripping from an `.iso` source instead of a physical disc (designed, not yet built).
- TV-series-aware ripping (episode detection and naming) and further session ergonomics.