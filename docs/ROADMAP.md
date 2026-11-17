# Roadmap

Planning-only repository. Milestones below are the intended order of work. Each one should be testable on the operator's own cabinets before the next implementation milestone is treated as done. The current card server remains in daily use until this one can replace it for real play.

Related documents: [PLAN.md](PLAN.md), [CLEAN-ROOM.md](CLEAN-ROOM.md), [ARCHITECTURE.md](ARCHITECTURE.md).

## Milestones

### M0 — Capture tooling and behavior spec

**Goal.** Be able to record, redact, and describe how the official game talks to a card server, without putting any of that raw material in git, and without reading unlicensed upstream source in order to write the spec.

**Work**

- A documented, private capture setup for the operator's cabinets (what to record, where files live, how they are ignored).
- A redaction checklist: card identifiers, access codes, boot tokens, tag secrets, and private host addresses come out before any note is copied into a public spec.
- The first behavior spec sections, in our own words, at a functional level, each marked *to be verified by packet capture against the real game*.
- Provenance notes on those sections, as required by [CLEAN-ROOM.md](CLEAN-ROOM.md).

**Acceptance**

- A cab boot, a card login, and a short save can be captured on the operator's machine and reduced to a redacted description.
- The public spec lists the message categories needed for M1–M4 and does not contain secrets or upstream source.
- No server code is required to close M0.

### M1 — Boot and network checks

**Goal.** A cabinet reaches **attract mode** talking only to this server for the card-server role.

**Work**

- A minimal ASP.NET Core host, config file, and listen loop.
- Implementations of the boot and network-check behavior recorded in the spec.
- Enough logging (with redaction) to see why a cab stopped, if it does.
- Later features may still be stubbed only if the spec says the cabinet can reach attract mode without them. If it cannot, those replies have to be real enough for boot to finish.

**Acceptance, on a real cab**

- With this server started and the previous card server not in the path, the cabinet boots and reaches attract mode.
- The same test is repeated after a process restart.
- Failure cases in the spec (unreachable server, bad config) fail in a defined way and do not hang the cab forever if the game itself gives up.

### M2 — Card login, registration, and save

**Goal.** A player can register or log in a card, play enough for the game to save, and see that data again after a restart.

**Work**

- SQLite persistence and the first schema.
- Register, login, load, and save paths from the spec.
- No real card identifiers in tests or logs.

**Acceptance, on a real cab**

- A new test card can be registered and used.
- An existing test card the operator controls can log in (once M6 exists this includes migrated cards; before that, only cards created on this server).
- A change that the game saves is still there after the server and the cabinet have been stopped and started again.

### M3 — Online settings and events

**Goal.** The settings and event data the cabinet reads from the card server after it is up are served correctly, as specified from captures.

**Work**

- Spec sections for those reads, filled from M0 captures and any follow-up captures this milestone needs.
- Operator-facing config or data files for the values we decide should be editable, without copying another project's file format.
- Verification that the cabinet's visible behavior matches what was configured.

**Acceptance, on a real cab**

- After boot, the cabinet receives the settings or event data the spec says it needs.
- A controlled change in those values is visible on the cab the next time it reads them, without regressing M1 or M2.

### M4 — Matchmaking integration and leaderboards

**Goal.** Cabinets using this card server can play online through the **existing** matchmaker, and result or leaderboard data the spec assigns to the card server is submitted or stored correctly.

**Work**

- Config for the matchmaking URL and the shared tag secret, compatible with the existing licensed matchmaker.
- The handoff behavior the cabinet expects, as recorded in the spec.
- Leaderboard or result submission as specified, again from captures and public licensed docs, re-checked on a real cab.
- This server does not replace the matchmaker.

**Acceptance, on a real cab**

- A cabinet pointed at this card server can enter online play through the operator's existing matchmaker, using the configured URL and tag secret.
- A completed test match produces the result or leaderboard behavior the spec requires.
- Secrets used for this test are not committed and are not printed in public logs or issues.

### M5 — Web pages

**Goal.** The essential browser-facing features recorded in the spec work against this server.

**Work**

- Choose a UI stack independently of any unlicensed project.
- Implement the pages and actions the spec lists (for example the account tasks players already rely on, once those are specified from observation).
- Authentication and bind-address rules for local vs shared installs.

**Acceptance**

- The listed tasks can be completed in a browser against a local build.
- A real card created in M2 can be viewed or managed as the spec describes, without leaking identifiers in the repository or in default log levels.

### M6 — Migration from an existing database

**Goal.** An operator can import a *copy* of a database from the card server they use today, after a written mapping spec is reviewed.

**Work**

- A written field-level mapping spec, with provenance, treating the source file as an observed data format. No upstream source or typings are copied into that document.
- An offline import tool that never modifies the source file.
- Clear failure reports for rows it cannot map.
- Handling of secrets: the source copy stays private, and the tool does not write secrets into logs or into this repo.

**Acceptance**

- The mapping spec is reviewed under [CLEAN-ROOM.md](CLEAN-ROOM.md) before the tool is written.
- A test import of a private copy succeeds for the agreed scope (at least login-capable player save data).
- A cabinet can log in to a migrated test card and load its data.
- No source database, and no raw dump of one, is committed.

### M7 — 1.0 releases and CI

**Goal.** Prebuilt, no-compile releases for Windows and Linux, with checksums, that the launchers can install.

**Work**

- GitHub Actions that build self-contained single-file artifacts for `win-x64` and `linux-x64` (and `linux-arm64` if that was chosen).
- A SHA-256 file published with every release.
- A sample config, a systemd unit, and a documented Windows service install.
- Launcher changes (in their own repos) to download the artifact, verify the hash, write config, and start the process, instead of building any other server.

**Acceptance**

- A tagged release produces both required artifacts and checksums on GitHub Actions without a local manual build.
- A clean Windows machine and a clean Linux machine can run the artifact from the release, with no SDK installed.
- The Windows launcher and the Linux launcher can install that release, verify SHA-256, and start the server until a cab can still complete the M1–M4 checks.
- Operators are told they can stop building the unlicensed server *only* after those checks pass.

## Parity test matrix

Use this as the living checklist for "can this replace the current card server for a real cab?" Every cell is a real-cabinet test unless marked otherwise. Fill it during the matching milestone; do not invent results in advance.

| # | Check | Milestone | Windows host | Linux host | Notes |
|---|---|---|---|---|---|
| 1 | Server starts from a release-style published build and listens on the configured loopback port | M1 / M7 | | | Launcher can wait on the port |
| 2 | Cabinet boots to attract mode | M1 | | | Previous card server not in the path |
| 3 | Network check failure is visible and recoverable after a fix | M1 | | | |
| 4 | New card registration | M2 | | | Test card only |
| 5 | Login of a card created on this server | M2 | | | |
| 6 | Save, restart server and cab, data still there | M2 | | | |
| 7 | Online settings or events the cab reads match the configured values | M3 | | | |
| 8 | Matchmaking URL and tag secret are accepted by the existing matchmaker | M4 | | | No private host addresses in reports |
| 9 | Cabinet can queue and complete a test match through that matchmaker | M4 | | | |
| 10 | Result or leaderboard submission matches the spec | M4 | | | |
| 11 | Essential web-UI tasks from the spec | M5 | | | |
| 12 | Migrated test card logs in and loads | M6 | | | Private source DB, never committed |
| 13 | Launcher downloads the release, verifies SHA-256, and starts the server | M7 | | | Windows and Linux launchers |
| 14 | Clean machine, no SDK, artifact from GitHub Releases only | M7 | | | |

A row is done only when it has been run, not when the code exists.

## How the launchers will download releases

Today the launchers cannot ship a compiled copy of the unlicensed card server, so they have to steer operators through a build step.

Once M7 is in place, both launchers should switch their "install or update the card server" flow to this project:

1. Look up the latest 1.0 (or operator-pinned) GitHub Release of `ServerOmega/exvs2-card-server`.
2. Download the artifact for `win-x64` or `linux-x64` (or `linux-arm64` when it exists).
3. Verify the SHA-256 against the published sum. Refuse to run on a mismatch.
4. Place the executable in the operator's chosen server directory and merge or write a config file, reusing the matchmaking URL and tag secret the launcher already stores locally.
5. Start the process and wait until it accepts connections, then start the rest of the cab as they do today.

The Linux launcher's Wine or Proton work stays on the game side. This server should run as a native Linux process. The Windows launcher should run the Windows artifact natively, not through a compatibility layer.

Until M7 is accepted, launcher work in those repos should not promise a no-build card server.

## Risks

### Legal

- The unlicensed upstream project must not be copied, in whole or in part. The process in [CLEAN-ROOM.md](CLEAN-ROOM.md) is the mitigation, and it only works if people follow it.
- This project is not affiliated with Bandai Namco or Sunrise. It must not ship game files, assets, or trademarks, and it does not grant anyone a license to the game.
- A clean-room server that speaks the game's network protocol is still a large project to review. If a mistake is found, we will remove or rewrite the offending material.

### Protocol gaps

- Captures may miss rare message types, error paths, event windows, or first-boot cases.
- A cab that reaches attract mode (M1) can still fail later when a neglected message appears.
- Mitigation: stage the work, keep the current server available, and extend the spec when a real cab shows a gap, rather than guessing from memory of another codebase.

### Encryption, signatures, and keys

- Traffic or saved payloads may be compressed, checksummed, signed, or encrypted.
- Extracted keys, official executable bytes, offsets, and hashes must never be committed, even if they would make a codec easier to write.
- If interoperability needs material that we cannot legally redistribute, we stop and document the limitation in the spec, rather than hide a key in the repo.
- Describe any such mechanism in the public spec only at a high level; keep private research notes out of git.

### Effort

- This is a large replacement, not a small tool. M1 through M4 are the critical path for real play. M5–M7 make it something other people can run without a build step.
- Until parity is honest, asking the community to switch would be premature. The current server stays the production path.

### Compatibility with the existing matchmaker

- The handoff has to stay compatible with the licensed matchmaker as it exists, including the matchmaking URL and the shared tag secret.
- Changes on either side should be negotiated in the public, licensed repos, not by special-casing a private host.

## Out of scope unless revisited

- Redistributing the game, official assets, or any unlicensed server.
- Replacing the matchmaker, the launchers, or the live monitor.
- Starting game processes or loading game DLLs from this server.
- Publishing private capture archives.
- A guaranteed schedule. Milestones move when real cabinets pass the checks, not when a date arrives.
