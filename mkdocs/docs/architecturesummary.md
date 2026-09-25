# ARM v3 — Architecture Documentation Summary

This is a condensed summary of all eleven documents in `docs/arch/` on the [`integration/all-prs`](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs) branch. The full documents are the source of truth for implementation detail; this is an orientation guide to what's in each one and how they connect.

## Quick summary

ARM v3 is a **multi-container, Python-first** system built around a job/session state machine, per-drive ripper containers, and ad-hoc transcode containers. The Backend is the brain; everything else is a worker that talks to the Backend over REST + WebSocket.

| Service | Image | Lifetime | Role |
|---|---|---|---|
| **UI** | `arm-ui` | Long-running | SPA (Vite-built) served by nginx; consumes Backend API + WS |
| **Backend** | `arm-backend` | Long-running | FastAPI: job/session state machine, internet adapters, WS hub, spawns transcoders |
| **Ripper** | `arm-ripper` | Long-running, one per drive | Bound to a single `/dev/sr*`; identifies disc, rips to `/raw`, reports to Backend |
| **Transcode** | `arm-transcode` | Ad-hoc, one per transcode | Spawned by Backend; optional GPU pass-through; reports progress, exits |
| **DB** | `postgres:18` | Long-running | Source of truth for all state |

## How the Documents Fit Together

This section sets out how the documents all fit together to define the architecture of the Automatic Ripping Machine.

- **00** sets the "why"; 
- **01** sets the "what" (topology); 
- **02**, **03**, and **04** are the operational core — lifecycle, wire contract, and schema — and cross\-reference each other constantly (a session's output path template in 02 is a column in 04, resolved over the REST calls in 03)
- **05** and 06 are the non\-functional backbone that every service is built against regardless of feature;
- **07** tracks what's deliberately still undecided;
- **08** governs the meta\-process of shipping this rebuild without breaking the existing release; 
- **09** documents engineering practice as it was actually built, superseding the original plan in 05; and 
- **10** is the one forward\-looking proposal in the set, explicitly labeled as not yet real.

## 00 — Vision, Goals, and Principles

This is covered in the [Overview](index.md) section of this site.

Defines who v3 is for (single\-admin homelab users, often "data\-hoarders" who rip to preserve raw bits), and the two structural v2 problems that justified a rebuild rather than a refactor: 
- Shared\-process resource contention between ripping/transcoding/UI, 
- No recovery path when a batch rip is interrupted. 

A third goal, Sessions (rip once, transcode many times), rides alongside these core structural issues in v2. 

This document lays out the six design principles behind v3:
- Bits\-first pipeline staging, 
- One\-service\-one\-responsibility, 
- Backend as the sole internet boundary, 
- Postgres as the single source of truth, 
- Crash\-safety by default,
- "No sacred cows" inherited from v2

Plus the concrete success criteria that define a shippable v3.0.

## 01 — System Architecture Overview

The container topology: 
 
 - Vue SPA (`arm-ui`, stateless, nginx\-served) talks over HTTPS/WSS to `arm-backend` (FastAPI, the only service holding durable state or internet access), which spawns one long\-running `arm-ripper-<serial>` container per enrolled optical drive, and one ephemeral `arm-transcode-<uuid>` container per transcode job via the Docker socket.
 
 - `arm-db` is Postgres 18, read/written by every service. 
 
 - Shared volumes (`/raw`, `/media`, `/logs`) are how rippers and transcoders hand off files; all coordination otherwise goes through Backend.

The full data flow end to end: 

Disc insertion → drive\-status polling detects it → ripper scans and calls Backend to identify (TMDB/OMDB/MusicBrainz) → on a hit the job is `identified`, on a miss it either blocks (`awaiting_user_id`, default) or — in opt\-in placeholder mode — rips immediately and defers identity → ripping proceeds per\-track with WS progress → on completion the disc ejects and an auto\-session may queue or it will wait for the user to select a session → Backend spawns a transcode container → output lands in `/media`. 

Also documents the drive\-polling mechanism (`ioctl(CDROM_DRIVE_STATUS)` every 2s, no udev dependency), and the image, replica count, inputs/outputs, and state of each service.

## 02 — Job Lifecycle & Crash Recovery

This is the document behind one of the project's two main objectives. It defines the four runtime entities:

- **Job** (one disc insertion), 
- **Track** (the checkpoint unit: one title/song/dump), 
- **Session Application** (the durable record of "apply session S to job J"), and 
 - **Transcode Task** (one raw\-track\-to\-output operation)
 
 It also includes the reusable **Session** template that composes a rip preset, a transcode preset, and an output\-path convention.

The document defines the state machines for **jobs** (`created → identified → ripping → ripped | ripped_partial`, with an `awaiting_user_id` detour and `abandoned`/`failed` terminal states) and **tracks** (`queued → in_progress → done | failed`, no per\-track claim needed since one ripper owns one drive). 

Crash recovery is deliberately coarse: **an interrupted rip restarts entirely from scratch** — every track resets to `queued`, `/raw/<job_id>/` is wiped, MakeMKV reruns. This is because MakeMKV can't resume mid\-title. "Done" in the DB doesn't guarantee fsync'd durability, and interruptions are rare enough that the cost is acceptable. Two paths trigger this: a Backend\-startup sweep (catches jobs stuck `ripping` when Backend was down) and a ripper\-startup probe (catches the reverse). 

Transcode tasks, by contrast, use real per\-task claims and heartbeats, because multiple ephemeral transcoders genuinely compete for queued work and a multi\-hour transcode batch losing all progress is a worse trade.

This document also explains why sessions were split into three tables (rip preset / transcode preset / session) versus v2's single bundled concept to support home\-movie rips, ISO dumps, and multiple encodings from one source; the full output\-path/naming convention for movies, TV, and music (deliberately mechanical, "honest" names — no guessed episode numbers); and the concurrency\-safety mechanisms (atomic `.arm-inprogress` → rename, apply\-time collision checks, a DB\-level partial unique index) that prevent two sessions from clobbering each other's output files.

## 03 — Protocol: REST \+ WebSocket Contract

Two transport methods are allowed between services: **REST** (JSON, request/response), used for anything that must durably land (state transitions, CRUD), and **WebSocket** (used only for streaming live progress and async push). There is no direct DB access between services, no file\-based IPC. All schemas are Pydantic models in `packages/arm_common/schemas/` are exported as OpenAPI and imported by both producer and consumer so a protocol change is one PR.

This document covers the full REST surface for three client relationships:

 - **Ripper ↔ Backend** (register, identify, resume\-after\-crash, per\-track complete/fail, job\-complete), 
 - **UI ↔ Backend** (auth, config, drives, jobs, sessions, transcodes, logs),
 - **Transcoder ↔ Backend** (register, claim, heartbeat, complete/fail)
 
 It also covers the single WebSocket endpoint's topic list (per\-job progress, typed events, per\-drive command channels, per\-task command channels), including that progress ticks are intentionally never persisted (fire\-and\-forget telemetry, throttled to 1 Hz) whilst typed events are.
 
 The document explains the REST\-vs\-WS split as two different cost models: state transitions must be acknowledged and retried, progress can simply be superseded by the next tick. Inter\-worker calls (rippers and transcoders never talk to each other) and Backend\-initiated calls to rippers (Backend pushes WS events instead; rippers are never HTTP servers) are explicitly ruled out.

## 04 — Data Model

This document covers the ARM data model Postgres 18 is the source of truth. All durable state lives here. There is no SQLite fallback, and no sidecar state stores.

The document sketches the logical data model. Exact column types, indexes, and constraints land in the first Alembic migration — this is the shape, not the DDL.

**Conventions:**

- Primary keys are ULIDs (text, lexicographically sortable) prefixed with the entity name: `job_01HXYZ…`, `track_01HXYZ…`. This makes log lines and URLs legible.
- Every table has `created_at` and `updated_at` (`TIMESTAMPTZ NOT NULL DEFAULT now()`).
- Soft-delete is avoided. If a row needs to hide, a status column handles it.
- Foreign keys are enforced at the DB layer; no ORM-only relationships.
- Enums are stored as plain `VARCHAR` columns; the `StrEnum` class in [packages/arm_common/arm_common/enums.py](../../packages/arm_common/arm_common/enums.py) is the source of truth and validation runs at write time through the SQLModel/Pydantic layer. No native Postgres `CREATE TYPE` enums. Rationale: adding/removing/renaming values via `ALTER TYPE` is awkward in transactional migrations, and several schema-diff tools (Atlas, Bytebase) gate enum-diff features behind paid tiers — `VARCHAR` keeps the door open to those tools on their free tiers. The tradeoff we accept: a bad value inserted by raw `psql` or a rogue migration is not caught by the DB; the app is the only guardrail. This is acceptable for a single-admin homelab system.

The document goes on to Walk through each core table: 
- `users` (single admin today, schema ready for more), 
- `config` (singleton runtime settings, plaintext secrets), 
- `drives` (one row per detected optical drive, ripper container created on enroll), 
- `jobs` and `tracks` (the rip\-side state machine, including forward\-compatible\-but\-unpopulated `disc_fingerprint`/`aacs_disc_id` columns reserved for a future community lookup database), 
- `rip_presets` / `transcode_presets` / `sessions` three\-table split,
- `transcode_tasks` (the one table that does carry claim/heartbeat columns, since ephemeral transcoders genuinely compete for work). 

Each table's rationale is given alongside its fields, not just the shape.

## 05 — Cross\-Cutting Concerns

This document covers the things that touch every service: configuration, authentication, secrets, logging, observability, code quality, testing, and the shared Python package.

- **Repo layout** defines ARM as a monorepo with a shared pyton package at the repository root. A key property of the design is that a single PR can change a Pydantic schema in arm_common and its producers + consumers in services/*. Protocol changes are always atomic.
- **Configuration** is two\-tier: bootstrap values in `.env` (DB credentials, the shared service token, PUID/PGID/CDROM\_GID), and everything else — API keys, retention policy, Apprise URLs — in the `config` DB table, edited from the UI. No YAML files (the v2 `arm.yaml` pattern is retired).
- **Auth** is JWT\-based (not cookies, to sidestep consent\-law complications behind arbitrary reverse proxies), argon2id\-hashed passwords, a forced password change on first login, 7\-day non\-refreshing tokens, and no MFA — all explicitly scoped to "single\-user homelab." Service\-to\-service auth uses one shared long\-lived bearer token, with authorization scoping at the endpoint level as the mitigation for the shared\-credential tradeoff.
- **Transport is TLS everywhere, no exceptions**  - an internal CA generated once per install signs leaf certs for every service, including Postgres itself; the deployment is explicitly LAN\-only and never internet\-exposed, which is why the document deliberately rules out Let's Encrypt.
- **Secrets** (third\-party API keys) are stored in plaintext DB rows — Postgres dumps must be treated as sensitive — and are redacted from logs.
- **Logging** is structured JSON to stdout plus a per\-service rotated file, with a per\-job log slice and a bug\-report zip endpoint; there is no metrics/tracing stack in v3.0 by design (the typed `events` table is the nearest thing).
- **Notifications** run through Apprise with native pass\-through URLs — a deliberate rejection of v2's hand\-maintained per\-service dictionary, which went stale as new notification services appeared.
- **Code quality** is one pre\-commit stack (ruff \+ strict mypy) gating all four Python packages equally.

## 06 — Deployment

Docker Compose on Linux (Engine ≥ 24, Compose v2 ≥ 2.20) is the **only** supported target — Unraid, Synology, other NAS\-appliance GUIs, TrueNAS, and Kubernetes are explicitly out of scope (a decision finalized 2026\-06\-05). 

Everything installs under a single prefix (`~/arm/` by default) with a fixed layout (`.env`, `docker-compose.yml`, `certs/`, `db/`, `raw/`, `media/`, `logs/`) — nothing compiles on the host, images are pulled pre\-built.

This document covers: 
- The build chain (fresh images on upstream bases, no inherited v2 `arm-dependencies` image or `phusion/baseimage`, `tini` as PID 1 for zombie\-reaping, digest\-pinned bases, SBOM \+ Cosign\-signed images, weekly rebuilds for security patches); 
- The compose topology (rippers and transcoders are *not* declared as compose services — the backend creates ripper containers dynamically on drive enrollment and transcode containers per job, both over the Docker socket, which is why they sit outside the `docker compose` project namespace); 
- The SCSI\-generic device\-pairing requirement MakeMKV needs;
- Why host\-side auto\-mount must be disabled via a udev rule for eject to work on desktop hosts; 
- The full `PUID`/`PGID` file\-ownership model (and the hard rule that v3 never recursively `chown`s a user\-mounted volume, a direct reaction to real v2 data\-loss bugs); 
- The privilege matrix (nothing runs `--privileged`); 
- Install/update/uninstall/backup procedures; and
- The platform\-specific notes ruling out ripping under Docker Desktop on macOS/Windows.

## 07 — Open Questions & Deferred Decisions

This document is deliberately short. Most first\-draft open questions have been resolved and folded into the main docs. The one live entry (OQ\-1) is whether the current DB\-as\-queue pattern (`SELECT … FOR UPDATE SKIP LOCKED`) should eventually be replaced with a real broker (Redis\+RQ, NATS). This has been explicitly deferred until a concrete pain point emerges, since it's adequate for the expected scale of 1–4 drives / 1–4 concurrent transcodes. 

The document also states the process for resolving future OQs: fold the decision into the relevant doc and delete the entry, relying on git history rather than an archive folder.

## 08 — v2 Isolation and Cutover Plan

This is the process document governing how v3 was built without destabilizing v2. It defines the isolation rule (v3 developed inside its own subtree, no v2 file touched during development), how the two stacks coexist on one host (distinct compose project namespaces, `armv3-` vs `arm-` container prefixes, noting there is one unavoidable resource conflict if both try to bind the same optical drive at once), and the mechanical steps of the eventual cutover PR: move v3 to repo root, retire v2 files, rewrite user\-facing docs, retire v2 CI workflows. 

The document also explicitly states what the cutover is *not* — not a data migration, not a branch rename, not a soft/gradual migration — and lists the readiness criteria that gate it (fresh\-install\-to\-login under 5 minutes, a real disc\-ISO end\-to\-end rip, a crash\-recovery drill pass, at least one real BD/DVD/CD rip on contributor hardware, green CI, and final maintainer sign\-off).

## 09 — Testing Philosophy (as\-built)

This document ecords what was actually built, which diverged from the original plan in 05. Five principles anchor it: 
1. Tests run with zero infrastructure (`uv run pytest`, no Docker/Postgres/network), 
2. Coverage is a floor for confidence rather than a vanity metric (100% Backend statement coverage, with exclusions required to carry a written justification), 
3. Only true I/O boundaries are faked — never the code under test, 
4. Speed is a feature (the 600\+\-test Backend suite runs in \~15s), 
5. Determinism beats fidelity where they conflict (GPU/Docker\-socket\-dependent paths are pinned to one deterministic branch rather than tested against real hardware in CI).

Two tiers actually exist for the Backend: 
- **Tier 1:** fast tests against an in\-memory fake SQLAlchemy session for exhaustive router/logic coverage,
- **Tier 2** a real\-DB end\-to\-end harness that boots the actual FastAPI app against file\-backed SQLite (standing in for Postgres, with the explicit accepted gap that Postgres\-only dialect features and real Alembic migrations aren't covered here).

A **contract tier** (OpenAPI\-drift checking, per\-endpoint request/response contract tests) and a **Big Buck Bunny integration rig** (a real, legally\-redistributable ISO fixture for testing the actual rip pipeline) are specified but not yet built as of this document — the closest existing approximation is in `devtools/iso-smoke.sh`.

## 10 — ISO\-Source Ripping (proposed, design only)

This is the only document in this set marked explicitly **not built**. 

It distinguishes between "**ISO as output**" (already shipped — `rip_presets.output_mode='iso'` dumps a physical disc to an image file) from "**ISO as source**" (ripping *from* an existing `.iso` file through the normal pipeline, no physical disc involved) — this document is only about the latter.

The proposed shape reuses two things nearly verbatim: the existing scan/identify/rip/transcode pipeline (already source\-blind — it routes `iso:<path>` vs `dev:<path>` at the scan layer) and the existing ephemeral\-container spawn mechanism already used for transcoding, rather than inventing a new long\-running `arm-ripper-iso` daemon.

Today, ISO\-source ripping exists only as an internal test hook (`ARM_MANUAL_TRIGGER_ISO`) that ripped once on startup and then idled — the opposite of the ephemeral design being proposed — which is why `devtools/iso-smoke.sh` had to grow into a large hand\-rolled orchestrator around it. The eventual front door (UI file upload spawning an ephemeral ISO ripper per upload) is explicitly deferred; this document locks in the architecture, not the upload UX.

