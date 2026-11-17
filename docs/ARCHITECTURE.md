# Architecture

This is a proposed architecture for the clean-room card server. It is not an implementation, and it is not a description of any existing unlicensed server. Module names, file layouts, and class designs will be chosen when code is written, from the behavior spec, not from another codebase.

## Goals

- One codebase that publishes **self-contained, single-file** native builds for Windows and Linux.
- Operators can run a release without installing a SDK and without compiling.
- The Windows and Linux launchers can download a release, verify SHA-256, write a config file, start the process, and manage it through a local admin API.
- A command-line admin tool covers the same management tasks on a headless host.
- There is no web interface. Public community stats stay on the existing global leaderboards, not on this process.
- Protocol behavior comes from the clean-room spec in [CLEAN-ROOM.md](CLEAN-ROOM.md).
- Secrets and player identifiers stay out of logs, sample config, and the git history.
- Moving to this server, or switching any matchmaking server (including a third-party one), must not interfere with player stats. See [MIGRATION.md](MIGRATION.md) and [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).

## Runtime and HTTP stack

- **.NET 8 or .NET 9** (see the open question in [PLAN.md](PLAN.md)).
- **ASP.NET Core** on **Kestrel**. One process serves two endpoint groups:
  - The **game-facing card protocol**, on the listen address and port the cabinets already use.
  - A **local admin API**, bound to `127.0.0.1` by default, used by the launchers and the CLI.
- Listen addresses and ports come from the config file, not from compiled-in defaults that point at any private host. A typical local launcher setup already waits on a loopback port for the game-facing API; the exact bind values will be confirmed during M0 captures and documented as examples using loopback only.

No other web server is required in the 1.0 process. A reverse proxy is optional only in front of the game-facing API if an operator wants TLS at the edge. The admin API stays on loopback and is not published through a public site.

## Packaging

Releases use `dotnet publish` with self-contained, single-file output:

| Runtime identifier | 1.0 intent |
|---|---|
| `win-x64` | Required |
| `linux-x64` | Required |
| `linux-arm64` | Optional, if it does not delay 1.0 |

Each artifact should be a single executable (plus a sample config, the admin CLI if it is a separate binary, and a short run guide in the release notes), accompanied by a SHA-256 sum for every file. Trimmed or AOT builds are a later optimization only if they do not break the protocol, the admin API, or the CLI. The first releases should prefer a reliable self-contained build over the smallest possible binary.

The repository will eventually hold GitHub Actions that build those two required runtimes on every release tag, upload the artifacts, and publish the checksums. Until milestone M7, no binaries are published.

## Data storage

- **SQLite** as the default database, stored in a configured data directory.
- Access via **EF Core or Dapper** (decided at M2). Schema migrations are versioned in this repository and applied on startup, or by an explicit command, so an older data directory can move forward safely.
- The schema is designed here, from the behavior spec. It is not imported from any other server's source.
- Card identity fields the cabinet uses to recognize a card (referred to here as ChipId and AccessCode) are stored so that a migrated card, or a card whose cabinet has switched matchmaking servers, keeps the same identity. Those values are secrets: they are never logged, never used as example data, and never committed.
- Match history rows record which matchmaking server (label or host) was in use, as described in [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md). Player stats themselves are not partitioned by matchmaker.

Do not commit live database files. Tests use throwaway files or in-memory equivalents, with fake identifiers only.

## Configuration

A single config file next to the executable, or at a path passed on the command line, should cover:

- Game-facing listen URL (host and port).
- Admin API listen URL, defaulting to `127.0.0.1` on a port documented in the sample config.
- Data directory (database, admin token file, backups, and any generated files).
- Matchmaking URL, in the same operator-facing sense as the existing licensed matchmaker's configured URL.
- Tag secret for talking to that matchmaker. Prefer an environment variable or a separate file with restricted permissions over a world-readable config, and never log the value. The launcher swaps the matchmaking URL and that server's tag secret together when the operator picks a saved matchmaking server (any operator); see [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).
- Log level and log path.

Sample config in the repo uses placeholders and loopback addresses only. A real tag secret or admin token is never an example value.

The process should refuse to start if required values are missing, if the admin API would bind off loopback without an explicit later decision allowing that, or if the admin token file is missing or empty. It should redact secrets, admin tokens, and card identifiers in every log line.

## Admin API and CLI

Milestone M5. This server does not ship a web UI, a static site, or a browser-facing account app.

### Admin API

- Local HTTP API bound to `127.0.0.1` by default.
- Token-authenticated. The process creates or reads a token file in the server data directory, with permissions restricted to the service user. The token is never written to logs, sample config, or this repository.
- Used by the [Windows launcher](https://github.com/ServerOmega/exvs2-launcher) and the [Linux launcher](https://github.com/ServerOmega/exvs2-launcher-linux) for:
  - Card management (lists and exports mask identifiers by default).
  - Leaderboard and match history views for cards this operator hosts.
  - Settings get and set.
  - Logs.
  - Backups and restores.
  - Health and status, so the launcher can tell that the process is ready.
- The API is not a public stats service. This operator's public community leaderboards stay on the existing global leaderboard deployment (a public page or Discord), which this server may submit to as [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md) describes. Submissions to a third-party matchmaker's board are opt-in per saved server.

### Command-line admin tool

A fallback for headless or Linux-only installs, and for operators who manage the server over SSH:

- Either `exvs2-card-server admin ...` or a separate `exvs2-cardctl` (see the open question in [PLAN.md](PLAN.md)).
- Talks to the same local admin API, using the token file, so launcher and CLI cannot drift apart.
- Commands at minimum: status, cards list (masked), backup, restore, export, settings get, settings set, logs.

Remote administration in 1.0 is "SSH to the host and run the CLI," not "expose the admin API on a public address."

Until M5, a health or ready check on loopback is enough for the launchers to know the server is up.

## Process model and OS integration

The 1.0 server is a long-running console process that can also run as a service.

- **Linux:** a systemd unit in the repo, with the executable path, config path, data directory, and a dedicated user. Restart-on-failure is expected. Optional: a short install note for running it under a user session during development.
- **Windows:** a Windows Service option, and later an optional tray helper if the launcher does not already cover "start with the cabs." For the first releases, "run the executable" plus a documented service install is enough.

The server must not start the game, the live monitor, or any game DLL. Launching cabinets stays the launcher's job. The launcher starts this process, waits until the listen port accepts connections, then starts the rest of the cab.

## Matchmaking integration

This process is a card server, not a matchmaker. It should:

- Accept the operator's matchmaking URL and tag secret from config.
- Give cabinets whatever boot-time or settings data the behavior spec says they need in order to use that matchmaker.
- Stay compatible with the existing licensed matchmaker, including the way that project already expects a matchmaking URL and a shared tag secret to be presented.
- Submit leaderboard or result data as the M4 spec requires, without assuming a particular private host. Submissions to this operator's board carry the last-matchmaker label as metadata and keep a single per-card identity, as in [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md). Third-party boards are opt-in and must never receive ChipId, AccessCode, or other secrets.
- Leave player stats untouched when the matchmaking URL or saved server changes. A third-party matchmaker is untrusted: validate what it sends, never let it write stats directly, and rate-limit that traffic.

It should not embed a second matchmaker unless a later, separate decision says otherwise.

## Migration from an existing card server database

Preserving player stats when an operator moves to this server is a **gating requirement for 1.0**. The procedure, mapping spec rules, dry-run reports, parity checks, freeze window, and rollback path are in [MIGRATION.md](MIGRATION.md).

Before any importer is written:

1. Write a **mapping spec** that describes the on-disk data to be read, the destination schema, and the field-level mapping, with provenance notes.
2. Treat the existing database as an observed data format, documented from a local copy the operator owns, not as an excuse to import upstream source, typings, or comments.
3. Review that mapping spec under the clean-room rules. Only then write the tool.

The tool must run offline, against a copy of the source database, leave the source file untouched, and refuse to commit or upload that file. Switching to this server is opt-in per operator and reversible by pointing the launcher back at the previous server.

## Logging, health, and operations

- Structured or plain-text logs with a consistent redaction policy (card identifiers, access codes, boot tokens, tag secrets, admin API tokens, and similar values).
- A health or ready check the launchers can use in addition to a raw TCP connect, once it is useful, on the admin API or a dedicated loopback endpoint.
- Graceful shutdown so in-flight saves are not cut off mid-write.
- No automatic phone-home. Updates are pulled by the launchers or by the operator when they choose a release.

## What the launchers expect

The [Windows launcher](https://github.com/ServerOmega/exvs2-launcher) and the [Linux launcher](https://github.com/ServerOmega/exvs2-launcher-linux) should, from M7, treat a GitHub Release of this project as the card server they install:

1. Download the artifact for the current OS.
2. Verify the published SHA-256.
3. Unpack or place the executable and a config file (the launcher already knows how to store a matchmaking URL and a tag secret locally, and must write those two as a pair when the operator picks a saved matchmaking server; each server has its own secret, which is never sent to another server).
4. Start the executable and wait until it is accepting connections.
5. Use the local admin API (token from the data directory) for card management, leaderboard and match history views, settings, logs, and backups.
6. Never compile, never fetch unlicensed upstream source, and never open a web UI on this server to manage it.

Until those releases exist, the launchers keep their current install path for the existing server.

## Tests, at the architecture level

- Unit tests for config, redaction, schema migrations, and pure protocol codecs once those exist.
- Conformance tests built from the behavior spec, using redacted fixtures, never raw captures.
- Importer dry-run and parity tests against synthetic copies of a source database, never against a live original and never with real identifiers committed.
- A parity matrix run on real cabinets, described in [ROADMAP.md](ROADMAP.md), including stats parity after migration and a matchmaking-server-switch round trip (own LA → own EU → a third-party test server → back to own LA).

No test in this repository may require a game executable, an unlicensed source tree, or a live private server address.
