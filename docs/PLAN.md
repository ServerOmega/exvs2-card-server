# Plan

This is the project plan for a clean-room EXVS2 Infinite Boost card server that ships prebuilt native Windows and Linux binaries. It is a plan only. Implementation starts after the clean-room process in [CLEAN-ROOM.md](CLEAN-ROOM.md) is in place and the first behavior spec work in milestone M0 has something to implement against.

Companion documents:

- [CLEAN-ROOM.md](CLEAN-ROOM.md) — who may write specs, who may write code, and what must never enter the repo
- [ARCHITECTURE.md](ARCHITECTURE.md) — proposed stack and packaging
- [ROADMAP.md](ROADMAP.md) — staged milestones, tests, and risks

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

## What this repository is

- A licensed, from-scratch card server (code comes later).
- Planning and process documents (this commit).
- Later: self-contained Windows and Linux builds, a config file, a database, optional service wrappers, a web UI, and a migration tool.

## What this repository is not

- A fork, copy, or relabel of the unlicensed upstream card server.
- A place for game files, assets, executables, byte offsets, or hashes of official binaries.
- A place for packet captures, card identifiers, access codes, boot tokens, webhook URLs, or host addresses of private servers.
- A replacement for the matchmaker, the launchers, or the game.

## Delivery model

From milestone M7, a release is a set of self-contained single-file builds plus a SHA-256 file. An operator, or the Windows or Linux launcher, should be able to:

1. Download the build for their OS.
2. Verify the published SHA-256.
3. Provide a config file (listen address, data directory, matchmaking URL, tag secret).
4. Start the process, or install the systemd unit / Windows service.

There is no compile step and no requirement to own a copy of any other card server's source.

Until M7, this repo does not publish binaries, and the launchers should keep using the current "download and build" path for the existing server.

## Work order

1. Agree the clean-room process and keep it visible to every contributor ([CLEAN-ROOM.md](CLEAN-ROOM.md)).
2. Milestone M0: capture tooling, redaction rules, and the first behavior spec sections, written in our own words at a functional level and marked "to be verified against the real game."
3. Implement against the spec in order: boot and network checks, card login and save, online settings, matchmaking handoff and leaderboard submission, web pages, then a migration tool with its own written mapping spec.
4. Only then cut 1.0 releases and point the launchers at them.

Details and acceptance tests are in [ROADMAP.md](ROADMAP.md).

## Open questions

These should be decided before the first implementation milestone (M1), except where noted.

1. **GPL-3.0 vs AGPL-3.0.** This repo starts under GPL-3.0, consistent with the matchmaking and launcher projects. AGPL-3.0 is a reasonable alternative for a network service: it would require operators who let others talk to a modified server over a network to offer the corresponding source. We should pick one license before M1 code lands, and not change it casually after people start shipping builds. If we switch, we do it before accepting outside implementation work.

2. **.NET 8 vs .NET 9.** [ARCHITECTURE.md](ARCHITECTURE.md) allows either. .NET 8 is the current long-term support release; .NET 9 may be attractive if its support window still covers 1.0. Pick before the first public build, and pin the version in the repo.

3. **EF Core vs Dapper** for SQLite access. EF Core is faster to evolve while the schema is moving. Dapper is thinner if we want every query explicit. A choice can wait until M2, when we first persist card data.

4. **Web UI technology.** The UI is milestone M5. The choice should be made on its own merits (simple static pages, a small SPA, or a .NET UI), not by copying the look, structure, or stack of any existing unlicensed web UI.

5. **How operators authenticate to the web UI**, and whether a local-only install can stay open on loopback with no login.

6. **linux-arm64.** Optional in the architecture. Decide whether 1.0 includes it or only `win-x64` and `linux-x64`.

7. **STUN and related network helpers.** The matchmaker already covers a large part of online play. This server should not grow those features unless captures show the cabinet expects something from the card server itself.

8. **Migration scope (M6).** Whether the first import tool covers only player save data, or also operator settings, friends, teams, and similar account data. A written mapping spec is required before any importer is written, either way.

## Immediate next step

After these planning documents are published, the first real work is **M0**: a private capture setup on the operator's own cabinets, a redaction checklist, and the start of a behavior spec that implementers can use without opening any upstream source. No server code, and no captures, belong in this repository during M0.
