# exvs2-card-server

A **clean-room** card server for *Mobile Suit Gundam: Extreme Vs. 2 Infinite Boost* (EXVS2 IB).

The goal is a server we can license and ship as **ready-to-run native builds for Windows and Linux**, so operators do not have to compile anything. The [Windows launcher](https://github.com/ServerOmega/exvs2-launcher) and the [Linux launcher](https://github.com/ServerOmega/exvs2-launcher-linux) will download a release, check its SHA-256, and start it.

This server is **headless**. It speaks the game-facing card protocol and a local admin API. Operators manage it from the launchers, or from a command-line tool on a machine with no desktop. It does not ship a web interface. Public community stats stay on the existing global leaderboards (a public page or Discord), not on this process.

This project is **not affiliated with**, endorsed by, or derived from Bandai Namco, Sunrise, or the unlicensed upstream card server that some people build today. It does not include game files, assets, executables, or any code from that upstream project. You must already have a legally obtained copy of the game.

Licensed under the **GNU General Public License v3.0** — see [LICENSE](LICENSE). Using AGPL-3.0 instead is an open question; see [docs/PLAN.md](docs/PLAN.md#open-questions).

## Status

**Planning.** This repository holds design and process documents only. There is no server code yet, and there are no binaries.

| Document | What it covers |
|---|---|
| [docs/PLAN.md](docs/PLAN.md) | Why this exists, how the work is staged, and open questions |
| [docs/CLEAN-ROOM.md](docs/CLEAN-ROOM.md) | Clean-room process, contribution rules, and DCO sign-off |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Proposed stack, packaging, config, admin API, CLI, and services |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Milestones M0–M7, parity tests, risks, and launcher downloads |
| [docs/MIGRATION.md](docs/MIGRATION.md) | Moving player data from an existing card server without losing stats |
| [docs/spec/CUSTOM-MATCH-ROOMS.md](docs/spec/CUSTOM-MATCH-ROOMS.md) | Draft behaviour spec for Custom Match rooms (LM LoadRoom / RegisterRoom, the cab's room tag, shared rooms across card servers) |
| [docs/FINDINGS-2026-11-17.md](docs/FINDINGS-2026-11-17.md) | 17 Nov 2026 findings: cross-server rooms working on real cabinets, Dual Select vs One rules, networking errors (21-01 / 21-11 / 21-32), the LM's 16-seat limit, LM resource use, the stand-in LM direction, problems and fixes |
| [docs/MATCHMAKING-SERVERS.md](docs/MATCHMAKING-SERVERS.md) | Switching any matchmaking server (including third-party ones) without touching card data or resetting the operator's leaderboard |

People should keep using their current card server until this one can replace it. Each milestone has to pass against real cabinets before the next one starts in earnest. A 1.0 release cannot ship until migrated player stats have been verified against the previous server.

## Why this exists

Releases of the current matchmaking project cannot ship a compiled card server, because the upstream server it patches has **no license**. Operators have to build that server themselves.

A server written from scratch, under a license we control, can publish prebuilt Windows and Linux binaries. The rule is clean room: we may study how the *game* talks on the network, and we may use public documentation, but we must not copy code, file layouts, class names, comments, or data from the unlicensed upstream project. The process is in [docs/CLEAN-ROOM.md](docs/CLEAN-ROOM.md).

## Community

Join the [EXVS2 matchmaking Discord](https://discord.gg/YENXAmjkYX) for guides, troubleshooting, and updates.

## Related projects

- Matchmaking server: [ServerOmega/exvs2-matchmaking](https://github.com/ServerOmega/exvs2-matchmaking)
- Windows launcher: [ServerOmega/exvs2-launcher](https://github.com/ServerOmega/exvs2-launcher)
- Linux launcher: [ServerOmega/exvs2-launcher-linux](https://github.com/ServerOmega/exvs2-launcher-linux)

This card server is meant to work with the existing matchmaker (including the configured matchmaking URL and shared tag secret). It does not replace that matchmaker. It can point a cabinet at any matchmaking server, including ones other people run. Changing which matchmaker a cabinet uses must not alter player stats or expose card data beyond the match protocol; see [docs/MATCHMAKING-SERVERS.md](docs/MATCHMAKING-SERVERS.md).

## License

Copyright (C) 2026 ServerOmega.

The original work in this repository is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE).

**Not covered by this license, and not included in this repository:**

- The unlicensed upstream card server and any of its code, names, comments, or data. All rights remain with its author. Nothing from it is relicensed here.
- The game, its data, images, executables, and trademarks (Bandai Namco / Sunrise). You must supply your own legally obtained game installation.
