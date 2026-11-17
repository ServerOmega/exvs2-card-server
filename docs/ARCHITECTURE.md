# Architecture

This is a proposed architecture for the clean-room card server. It is not an implementation, and it is not a description of any existing unlicensed server. Module names, file layouts, and class designs will be chosen when code is written, from the behavior spec, not from another codebase.

## Goals

- One codebase that publishes **self-contained, single-file** native builds for Windows and Linux.
- Operators can run a release without installing a SDK and without compiling.
- The Windows and Linux launchers can download a release, verify SHA-256, write a config file, and start the process.
- Protocol behavior comes from the clean-room spec in [CLEAN-ROOM.md](CLEAN-ROOM.md).
- Secrets and player identifiers stay out of logs, sample config, and the git history.

## Runtime and HTTP stack

- **.NET 8 or .NET 9** (see the open question in [PLAN.md](PLAN.md)).
- **ASP.NET Core** on **Kestrel**. One process serves the cabinet-facing HTTP API and, from milestone M5, the operator and player web UI.
- Listen address and port come from the config file, not from compiled-in defaults that point at any private host. A typical local launcher setup already waits on a loopback port; the exact bind values will be confirmed during M0 captures and documented as examples using loopback only.

No other web server is required in the 1.0 process. A reverse proxy (nginx, Caddy, IIS) is optional for operators who want TLS at the edge.

## Packaging

Releases use `dotnet publish` with self-contained, single-file output:

| Runtime identifier | 1.0 intent |
|---|---|
| `win-x64` | Required |
| `linux-x64` | Required |
| `linux-arm64` | Optional, if it does not delay 1.0 |

Each artifact should be a single executable (plus a sample config and a short run guide in the release notes), accompanied by a SHA-256 sum for every file. Trimmed or AOT builds are a later optimization only if they do not break the protocol or the web UI. The first releases should prefer a reliable self-contained build over the smallest possible binary.

The repository will eventually hold GitHub Actions that build those two required runtimes on every release tag, upload the artifacts, and publish the checksums. Until milestone M7, no binaries are published.

## Data storage

- **SQLite** as the default database, stored in a configured data directory.
- Access via **EF Core or Dapper** (decided at M2). Schema migrations are versioned in this repository and applied on startup, or by an explicit command, so an older data directory can move forward safely.
- The schema is designed here, from the behavior spec. It is not imported from any other server's source.

Do not commit live database files. Tests use throwaway files or in-memory equivalents, with fake identifiers only.

## Configuration

A single config file next to the executable, or at a path passed on the command line, should cover:

- Listen URL (host and port).
- Data directory (database and any uploaded or generated files).
- Matchmaking URL, in the same operator-facing sense as the existing licensed matchmaker's configured URL.
- Tag secret for talking to that matchmaker. Prefer an environment variable or a separate file with restricted permissions over a world-readable config, and never log the value.
- Log level and log path.
- Later: web UI bind and authentication settings.

Sample config in the repo uses placeholders and loopback addresses only. A real tag secret is never an example value.

The process should refuse to start if required values are missing, and should redact secrets and card identifiers in every log line.

## Web UI

Milestone M5. Served by the same process (static files, a small hosted UI, or similar). It should cover the account and operator tasks that the behavior spec records from real use, such as inspecting a card and managing friends or teams, once those are specified.

The UI stack is an open question. It must be chosen independently, not by reproducing another project's pages, component tree, or client framework.

Until M5, a healthy process and a simple status page or health endpoint are enough for the launchers to know the server is up.

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
- Submit leaderboard or result data as the M4 spec requires, without assuming a particular private host.

It should not embed a second matchmaker unless a later, separate decision says otherwise.

## Migration from an existing card server database (M6)

This is an **explicit later milestone**, not something the first boot or card-login builds try to do.

Before any importer is written:

1. Write a **mapping spec** that describes the on-disk data to be read, the destination schema, and the field-level mapping, with provenance notes.
2. Treat the existing database as an observed data format, documented from a local copy the operator owns, not as an excuse to import upstream source, typings, or comments.
3. Review that mapping spec under the clean-room rules. Only then write the tool.

The tool must run offline, against a copy of the source database, leave the source file untouched, and refuse to commit or upload that file. Identifiers that are secrets must be handled as secrets. A cab logging in to a migrated card, on a test copy, is the acceptance test. Details are in [ROADMAP.md](ROADMAP.md).

## Logging, health, and operations

- Structured or plain-text logs with a consistent redaction policy (card identifiers, access codes, boot tokens, tag secrets, and similar values).
- A health or ready check the launchers can use in addition to a raw TCP connect, once it is useful.
- Graceful shutdown so in-flight saves are not cut off mid-write.
- No automatic phone-home. Updates are pulled by the launchers or by the operator when they choose a release.

## What the launchers expect

The [Windows launcher](https://github.com/ServerOmega/exvs2-launcher) and the [Linux launcher](https://github.com/ServerOmega/exvs2-launcher-linux) should, from M7, treat a GitHub Release of this project as the card server they install:

1. Download the artifact for the current OS.
2. Verify the published SHA-256.
3. Unpack or place the executable and a config file (the launcher already knows how to store a matchmaking URL and a tag secret locally).
4. Start the executable and wait until it is accepting connections.
5. Never compile, and never fetch unlicensed upstream source.

Until those releases exist, the launchers keep their current install path for the existing server.

## Tests, at the architecture level

- Unit tests for config, redaction, schema migrations, and pure protocol codecs once those exist.
- Conformance tests built from the behavior spec, using redacted fixtures, never raw captures.
- A parity matrix run on real cabinets, described in [ROADMAP.md](ROADMAP.md).

No test in this repository may require a game executable, an unlicensed source tree, or a live private server address.
