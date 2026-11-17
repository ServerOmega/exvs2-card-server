# Plan

This is the project plan for a clean-room EXVS2 Infinite Boost card server that ships prebuilt native Windows and Linux binaries. It is a plan only. Implementation starts after the clean-room process in [CLEAN-ROOM.md](CLEAN-ROOM.md) is in place and the first behavior spec work in milestone M0 has something to implement against.

Companion documents:

- [CLEAN-ROOM.md](CLEAN-ROOM.md) — who may write specs, who may write code, and what must never enter the repo
- [ARCHITECTURE.md](ARCHITECTURE.md) — proposed stack, packaging, admin API, and CLI
- [ROADMAP.md](ROADMAP.md) — staged milestones, tests, and risks
- [MIGRATION.md](MIGRATION.md) — moving player data from an existing card server without losing stats
- [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md) — switching any matchmaking server without touching card data

## Problem

A card server is the service a cabinet talks to when it boots, when a player uses a card, and when the cabinet needs online settings, save data, and a pointer to matchmaking. Today many operators run a patched build of an upstream card server that **has no license**. Because of that:

- The public matchmaking project cannot ship a compiled card server in its releases.
- Operators have to obtain that upstream tree and build it themselves.
- The Windows and Linux launchers cannot honestly offer a one-click, no-compile server install.

We need a server whose license we control, so a GitHub Release can contain ready-to-run builds for `win-x64` and `linux-x64`, with published SHA-256 sums, that the launchers can download and start.

## Approach

Write a new server from scratch under GPL-3.0 (AGPL-3.0 is an open question below).

The protocol behavior will be specified from:

1. Packet captures of the **game itself** talking to a card server, taken from cabinets the operator already owns and is allowed to run.
2. Public documentation, including the licensed docs in [exvs2-matchmaking](https://github.com/ServerOmega/exvs2-matchmaking) and the launchers, used only at a functional level and always re-checked against captures.

Implementers work from those specs. They must not consult the unlicensed upstream source, patches that contain its context, or decompiled copies of it. See [CLEAN-ROOM.md](CLEAN-ROOM.md).

The work is staged so each milestone is testable on real cabinets. The current server stays in production until this one can take a cabinet from boot, through card login and save, to a match on the existing matchmaker.

This repo implements the **card server** role only. Matchmaking stays in [exvs2-matchmaking](https://github.com/ServerOmega/exvs2-matchmaking). This server must be able to hand a cabinet the configured matchmaking URL and work with the existing shared tag secret, so operators can keep the matchmaker they already run.

The server is **headless**. Cabinets talk to the game-facing card protocol. Operators talk to a local admin API from the Windows or Linux launcher, or to a command-line admin tool on a headless host. There is no web interface on this process. Public community stats stay on the existing global leaderboards (a public page or Discord), not on this server.

Moving to this server, or switching a cabinet between any matchmaking servers (this operator's or a third party's), must not interfere with player stats. Those rules are in [MIGRATION.md](MIGRATION.md) and [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md). Verified stats parity is a gate for 1.0.

## What this repository is

- A licensed, from-scratch card server (code comes later).
- Planning and process documents (this commit).
- Later: self-contained Windows and Linux builds, a config file, a database, optional service wrappers, a local admin API, a command-line admin tool, and a migration tool that preserves player stats.

## What this repository is not

- A fork, copy, or relabel of the unlicensed upstream card server.
- A place for game files, assets, executables, byte offsets, or hashes of official binaries.
- A place for packet captures, card identifiers, access codes, boot tokens, webhook URLs, admin API tokens, or host addresses of private servers.
- A web application or a public stats site.
- A replacement for the matchmaker, the launchers, or the game.

## Delivery model

From milestone M7, a release is a set of self-contained single-file builds plus a SHA-256 file. An operator, or the Windows or Linux launcher, should be able to:

1. Download the build for their OS.
2. Verify the published SHA-256.
3. Provide a config file (game-facing listen address, admin listen address, data directory, matchmaking URL, tag secret).
4. Start the process, or install the systemd unit / Windows service.
5. Manage it from the launcher, or with the command-line admin tool on a headless host.

There is no compile step, no web UI to host, and no requirement to own a copy of any other card server's source.

Until M7, this repo does not publish binaries, and the launchers should keep using the current "download and build" path for the existing server.

## Work order

1. Agree the clean-room process and keep it visible to every contributor ([CLEAN-ROOM.md](CLEAN-ROOM.md)).
2. Milestone M0: capture tooling, redaction rules, and the first behavior spec sections, written in our own words at a functional level and marked "to be verified against the real game."
3. Implement against the spec in order: boot and network checks, card login and save, online settings, matchmaking handoff and leaderboard submission, then the admin API, CLI, and launcher integration.
4. Build the migration tool from a written mapping spec, and prove stats parity on real cabinets ([MIGRATION.md](MIGRATION.md)). This is a **gating requirement for 1.0**: a release cannot be called 1.0 until a migrated card shows the same stats as it did on the previous server, and a matchmaking-server switch (including to a third-party matchmaker and back) leaves those stats untouched ([MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md)).
5. Only then cut 1.0 releases and point the launchers at them.

Details and acceptance tests are in [ROADMAP.md](ROADMAP.md).

## Open questions

These should be decided before the first implementation milestone (M1), except where noted.

1. **GPL-3.0 vs AGPL-3.0.** This repo starts under GPL-3.0, consistent with the matchmaking and launcher projects. AGPL-3.0 is a reasonable alternative for a network service: it would require operators who let others talk to a modified server over a network to offer the corresponding source. We should pick one license before M1 code lands, and not change it casually after people start shipping builds. If we switch, we do it before accepting outside implementation work.

2. **.NET 8 vs .NET 9.** [ARCHITECTURE.md](ARCHITECTURE.md) allows either. .NET 8 is the current long-term support release; .NET 9 may be attractive if its support window still covers 1.0. Pick before the first public build, and pin the version in the repo.

3. **EF Core vs Dapper** for SQLite access. EF Core is faster to evolve while the schema is moving. Dapper is thinner if we want every query explicit. A choice can wait until M2, when we first persist card data.

4. **CLI shape.** The admin tool can be a subcommand (`exvs2-card-server admin ...`), a separate `exvs2-cardctl` binary, or both (the subcommand as the default, with a thin alias if it helps packaging). Decide before M5.

5. **Admin API bind beyond loopback.** The default is `127.0.0.1` only, with a token file in the data directory. For 1.0, remote operators should SSH to the host and use the CLI (or run the launcher on that same machine). Binding the admin API to a non-loopback address is out of scope unless a later decision adds TLS and a tighter access model.

6. **linux-arm64.** Optional in the architecture. Decide whether 1.0 includes it or only `win-x64` and `linux-x64`.

7. **STUN and related network helpers.** The matchmaker already covers a large part of online play. This server should not grow those features unless captures show the cabinet expects something from the card server itself.

8. **Migration scope (M6).** [MIGRATION.md](MIGRATION.md) requires the first import to preserve cards, profiles, unlocks, play counts, win/loss, class/rank, match history, and leaderboard submission state, with identity fields left unchanged. Whether that first tool also covers operator settings, friends, teams, and similar account data can be decided in the mapping spec, but stats-bearing player data is not optional.

9. **Last-matchmaker metadata on the operator's board.** [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md) recommends keeping one entry per card on this operator's global leaderboard, with an optional last-matchmaker label as metadata only. Third-party board submission is opt-in per saved server and, if used at all, may send only a derived pseudonymous id. Confirm or replace those rules before M4 leaderboard work is considered done.

## Immediate next step

After these planning documents are published, the first real work is **M0**: a private capture setup on the operator's own cabinets, a redaction checklist, and the start of a behavior spec that implementers can use without opening any upstream source. No server code, and no captures, belong in this repository during M0.
