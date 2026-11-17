# Migration

This document is the plan for moving an operator from the card server they run today to this clean-room server **without interfering with player stats**. It is also a **gating requirement for 1.0**: a release cannot be called 1.0 until the parity checks below have passed on real cabinets.

Switching matchmaking servers (any operator, for example this operator's LA or EU servers or a third-party matchmaker, and back) is a different operation. That path must also leave stats intact, but it does not copy a database. See [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).

Related documents: [PLAN.md](PLAN.md), [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md), [CLEAN-ROOM.md](CLEAN-ROOM.md).

## Rules that cannot be relaxed

1. **Player stats are preserved.** After import, a card shows the same profile, unlocks, play counts, win/loss, class/rank, and match history the previous server showed.
2. **The importer never writes the original database.** It reads only a copy. Rollback is "keep using the previous server."
3. **Card identity does not change.** The fields the cabinet uses to recognize a card, referred to here as ChipId and AccessCode, are copied as-is. They are not regenerated, re-encoded, or replaced. A card that logged in before must still match after the switch.
4. **Global leaderboard identity stays the same.** Submissions after import use the same card identity and the same submission keys, so this operator's community entries do not duplicate or reset. Which matchmaking server was used is metadata, not part of the identity. Resubmission is idempotent. Details are in [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).
5. **Switching is opt-in and reversible.** An operator chooses when to point a launcher at this server. Pointing it back at the previous server, with the original database untouched, must still work.
6. **No upstream source.** The mapping spec treats the existing database as an observed data format, documented from a copy the operator owns. It does not copy typings, comments, schema dumps taken from unlicensed source, or file layouts from that project.
7. **No secrets in git.** Source databases, raw dumps, real ChipId or AccessCode values, tag secrets, and admin tokens stay out of this repository and out of logs.

## Mapping spec

Before any importer is written, write a field-level mapping spec and review it under [CLEAN-ROOM.md](CLEAN-ROOM.md).

The spec is derived from **read-only** inspection of a copy of the operator's own existing database, plus what real cabinets have already shown in captures. It must describe:

- Each source table or store that holds cards, profiles, unlocks, play counts, win/loss, class/rank, match history, or leaderboard submission state.
- The destination table or store in this server's schema.
- The field-level mapping, including types, units, and any allowed transformation (for example, a timestamp epoch). Identity fields have **no** transformation.
- Fields that cannot be mapped, and what the tool will do with them (skip with a reported warning, or fail the import).
- Provenance: when the copy was inspected, that it was a copy, who wrote the spec, and that no unlicensed source was consulted.

The mapping spec may live in this repository only after secrets and real identifiers have been removed. Private notes and the database copy itself stay outside git, or in an ignored path.

## Importer

- Runs offline, as a command on the admin CLI (or a dedicated import subcommand).
- Takes the path to a **copy** of the source database. If it can detect that it has been pointed at a live data directory, it should refuse, or at least require an explicit "this is a copy" confirmation after explaining the risk.
- Never opens the original file for write, never deletes it, and never replaces it.
- Applies this server's schema to a new or empty destination data directory, then inserts mapped rows.
- Fails closed on unmapped stats-bearing fields rather than dropping them silently.

## Dry-run report

A dry-run is mandatory before a real import. It writes no destination data (or writes only to a throwaway directory that is discarded). The report includes counts per table and per card for at least:

- Cards
- Profiles
- Unlocks
- Play counts
- Win/loss
- Class/rank
- Match history
- Leaderboard submission state

It also lists rows it cannot map, with enough context to fix the mapping spec, and without printing secrets or raw identifiers. A real import is not accepted unless its counts match this report.

## Parity checks

Before and after a real import, compute:

- The dry-run counts above, on the source copy (before) and on the destination database (after).
- Checksums or other aggregate fingerprints over each mapped, stats-bearing collection, taken in a defined order, with identity fields included so a rewritten ChipId or AccessCode cannot hide. Do not publish those checksums alongside real identifiers.

The import fails if counts or checksums do not match, unless the mapping spec has already documented a precise, reviewed reason (for example, a known unused source column that is not stats-bearing). "Close enough" is not accepted for player stats.

## Side-by-side verification on real cabinets

Counts and checksums are not enough. After a successful import of a private copy:

1. Keep the previous server available, still pointed at the **untouched original** database (or at a second copy of it, never at the file the importer read if there is any doubt).
2. Point one test cabinet at this server and log in a migrated test card the operator controls.
3. Log in the same card on the previous server.
4. Compare the in-game stats screen and any related menus: profile, unlocks, play counts, win/loss, class/rank, and match history.
5. Confirm the card is recognized as the same card (no re-registration prompt, no empty save).
6. After a test match, confirm that a leaderboard submission updates the existing community entry on this operator's board rather than creating a second one or resetting totals. Use the identity and idempotency rules in [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).

Only a pass on this comparison closes the M6 stats-parity row in [ROADMAP.md](ROADMAP.md).

## Freeze window and switch procedure

A production switch happens only when cabinets are idle: no card login in progress, no unsaved play, no queue, and no match. The operator should announce a short freeze, then:

1. Stop the previous card server, so it cannot write more rows after the copy is taken.
2. Copy the database to a private working file. Leave the original where it is, read-only for the rest of this procedure.
3. Run the dry-run against the copy. Fix mapping problems, if any, and repeat until the report is clean.
4. Import the copy into a new data directory for this server.
5. Verify counts and checksums.
6. Start this server against the new data directory. Do not yet point every cabinet at it.
7. Run the side-by-side cabinet checks on a test cab.
8. If those pass, point the launcher at this server (game-facing URL, admin API, and the already-configured matchmaking URL and tag secret pair).
9. Keep the previous server and its original database until the operator is satisfied. There is no deadline in this document for deleting them.

If any step fails, stop. The rollback is to start the previous server again and point the launcher at it. Because the original database was never written, player stats are where they were at the freeze.

## Rollback

Rollback does not try to write data back into the previous server.

- Leave the original database untouched.
- Start the previous server.
- Point the launcher at it.
- This server and its imported copy can be kept for another attempt.

Switching to this server is therefore reversible for as long as the operator keeps the previous server and its original database.

## Global leaderboard continuity

Migration must not create a second community identity for a card, and must not reset an existing entry.

- Submission keys are derived from the same preserved card identity, not from a new identifier this server invents.
- Which matchmaking server was used is metadata on a submission, not part of the identity. See [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md).
- After import, an idempotent resubmission of the card's current cumulative stats is allowed so the community board stays in sync. A resubmit with the same identity updates the existing row. It does not append a duplicate and does not zero totals.

Private host addresses of those boards are never written into this repository.

## Compatibility tests

The following rows in the [ROADMAP.md](ROADMAP.md) parity matrix are the official checks for this document. They must be run, not just coded, before 1.0:

- Migrated test card logs in; dry-run counts and checksums match the import.
- Stats parity on the in-game stats screen, side by side with the previous server.
- Rollback to the previous server still works.
- A later matchmaking-server-switch round trip (own LA → own EU → a third-party test server → back to own LA), described in [MATCHMAKING-SERVERS.md](MATCHMAKING-SERVERS.md), still shows the same stats and the same identity on this operator's leaderboard.

## What stays out of this repository

- Source databases and any raw dump or copy of one.
- Real ChipId, AccessCode, or other identifier values.
- Mapping notes that still contain those values.
- Private host addresses, tag secrets, and admin tokens.

Tests in this repo use synthetic source files with fake identifiers only.
