# Switching matchmaking servers

This document is the plan for pointing a cabinet at **any** matchmaking server — one the operator runs themselves (for example LA or EU) or one run by another person or community — **without breaking card data, match history, or the operator's global leaderboard identity**.

"Region" is not the unit of switching. A saved **matchmaking server** is: a label, a MatchUrl, and the tag secret that operator gave you. LA and EU are only examples of servers this operator already runs. The same rules apply to a third-party matchmaker.

This is not a database migration. Player stats live on the **player's own card server**. Changing which matchmaker a cabinet uses must not rewrite that data, move it off this server, or expose it beyond what the match protocol already needs. Moving from the previous card server software to this one is covered in [MIGRATION.md](MIGRATION.md).

Related documents: [PLAN.md](PLAN.md), [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md).

## What stays put

Player stats live on the operator's card server, in the database this process owns:

- Cards and identity fields (ChipId and AccessCode)
- Profiles
- Unlocks
- Play counts
- Win/loss
- Class/rank
- Match history (each row records which matchmaking server was in use, described below)
- Leaderboard submission state

Changing the matchmaking URL, or picking a different saved server in the launcher, **never** rewrites those records and **never** copies the card database to the matchmaker. A player who plays on the operator's LA server, switches to the operator's EU server, then to a third-party test server, and switches back to LA still has the same card, the same totals, and the same history they had before the first switch, plus any matches they played in between.

Card data is not sent to a matchmaker except insofar as the game's own match protocol, as recorded in the behavior spec, already requires a value in a given message. This server must not add profiles, unlocks, play counts, ChipId, AccessCode, tag secrets, or other card-server records to that traffic.

## Each saved server is a MatchUrl and tag secret pair

Each saved matchmaking server has its own MatchUrl and its own tag secret. The secret is given by **that** server's operator. It is stored only on this machine (in this server's config, or a restricted file, or an environment variable — never in the public repo, never in the shared launcher server list if that list is synced or exported, and never in logs).

- The launcher already keeps a saved list of matchmaking servers. When the operator picks one, the launcher must write **both** the URL and that server's tag secret into this server's config (or the equivalent settings the admin API exposes).
- It must not leave a previous server's secret paired with a new URL, or the other way around.
- A secret is never sent to a different matchmaker, and is never reused as an example or default for another saved server.
- This server hands cabinets the configured URL and uses the configured secret, as the existing licensed matchmaker already expects.
- Private host addresses and real secrets are never examples in this repository. Labels such as "LA", "EU", or "third-party test" are fine in docs and in stored metadata.

## Leaderboards

There are two kinds of board, and they are not mixed.

### The operator's own global leaderboard

Community leaderboards this operator already runs (the existing public page or Discord, not a site on this card server) identify a card once.

- The identity used to submit or update an entry is the preserved card identity, the same one [MIGRATION.md](MIGRATION.md) keeps unchanged. Public pages still must not display ChipId, AccessCode, or other secrets.
- Which matchmaking server was last used is **metadata on a submission**, not part of the identity key. A player who moves from LA to EU, or from either of those to a third-party matchmaker, does not get a second row, and their existing totals are not reset.
- Resubmission is **idempotent**: a submit or resubmit with the same card identity updates the existing entry. It does not append a duplicate and does not zero the row.
- After a server switch, this server may resubmit the card's current cumulative stats with updated matchmaker metadata, so the board reflects the move without creating a new identity.

The recommended display rule, to be confirmed or replaced before M4 leaderboard work is considered done (see [PLAN.md](PLAN.md)):

- A card's optional **last matchmaker** label is the saved-server label (or host name, never a private address in public material) of its most recent accepted submission to this operator's board.
- Cumulative stats remain a single entry for that card. They are not split, reset, or copied when the last-matchmaker label changes.
- A filtered view may show entries whose current label matches, or matches whose history label matches, without inventing a second identity.
- Switching back to a previous server, followed by the idle-time resubmit below, can set that label back. Totals do not rewind.

### Third-party matchmaker leaderboards (opt-in)

A third-party matchmaker may run its own board. Submitting to it is **opt-in per saved server, default off**. Turning on submission for one server does not turn it on for others.

- These submissions are separate from the operator's own global leaderboard. They must not replace, split, or reset the single per-card entry on this operator's board.
- Never send ChipId, AccessCode, tag secrets, admin tokens, or raw card records.
- If the protocol or that board's API allows a stable identifier at all, send only a **derived pseudonymous id** computed locally on this card server. A reasonable design is a keyed hash of the preserved card identity with a secret that never leaves this card server. The third party must not be able to recover ChipId or AccessCode from it.
- Prefer a distinct derived id **per third-party server**, so two communities cannot correlate the same player across boards unless the operator explicitly opts into a shared identity.
- If the protocol has no safe way to submit without sending secrets or raw identifiers, do not submit. Opt-in stays off.

## Match history records which server was used

Each match history row stored on this card server records the matchmaking server that was in use at the time of the match (the saved-server label and/or host name, never a private host address in public material or logs that leave the machine).

- History is append-only with respect to completed matches. Switching servers does not delete, rewrite, or move old rows.
- A player who played on LA, then EU, then a third-party server, then LA again keeps all of those matches, each tagged with the server they were played on.
- Views in the launcher and the CLI can show that label next to each match.

## Trust model

A matchmaker, especially one run by someone else, is **untrusted**.

- This card server must validate anything it receives from a matchmaker: expected fields, types, sizes, and ranges. Unexpected fields are ignored. Malformed or contradictory payloads are dropped and logged without secrets.
- A matchmaker must **never** be allowed to write card stats, profiles, unlocks, play counts, win/loss, class/rank, or match history directly.
- If a match result arrives via the matchmaker, it can be used only after validation, and only as the behavior spec allows. It is not an authoritative overwrite of cumulative totals.
- Rate-limit traffic to and from matchmakers, including `/status` probes, result or leaderboard submissions, and any other matchmaker-facing calls.
- The operator's own remote matchmaker (for example a VPS) is still a separate trust domain for card data: stats stay on this card server.

## Compatibility

A matchmaking server should advertise a protocol version on `GET /status` (the licensed matchmaker already includes a version field in that response).

- The launcher and this server record the protocol version range they support.
- If `/status` is missing, unreachable, or reports a version outside that range, the launcher **warns** before the switch is applied. A known-incompatible version should not be applied silently.
- Until `/status` exists on a given server, the launcher may still save the entry and show a ping by other means, but it must not claim the server is a compatible matchmaker.

## When a switch is allowed

A switch is allowed only when cabinets are **idle**:

- No cabinet is in a matchmaking queue.
- No cabinet is in a match.
- No card login or save is in progress.

An in-progress queue or match cannot be moved to another matchmaker. The launcher, and the admin settings-set path, must refuse a MatchUrl or tag-secret change until those conditions are met. If a cab is busy, the operator waits or stops the queue first.

## Switching back

Switching back to a previous saved server restores prior matchmaking configuration with no data loss:

- Card data was never moved, so it does not need to be restored.
- The launcher writes that server's URL and that server's tag secret as a pair. Secrets for other servers stay local and are not sent.
- An idle-time idempotent resubmit may update the last-matchmaker label on the operator's own board. It does not reset totals or create a duplicate.
- Match history still contains the matches played on the other servers, tagged as such.
- Opt-in third-party board settings stay as the operator left them; switching back does not silently start or stop those submissions.

After the round trip, the card's stats screen matches what the operator would expect from "the same card, plus any matches played while away."

## Suggested operator procedure

1. Confirm every cab is idle.
2. In the launcher, pick the destination saved matchmaking server (any operator). The launcher sets MatchUrl and that server's tag secret together, and warns if `/status` reports an incompatible protocol version.
3. Confirm this server is using the new pair (CLI or launcher status), and that the previous server's secret is not still configured.
4. Optionally resubmit current cumulative stats so the operator's own board's last-matchmaker label updates. Leave third-party board submission off unless the operator has chosen it for that server.
5. Start play on the new server.
6. To go back, or to move to any other saved server, repeat from step 1.

No database copy and no freeze of the card server itself is required for a matchmaking-server switch, because the card database is not the thing that is changing.

## Compatibility tests

The official check is the matchmaking-server-switch row in the [ROADMAP.md](ROADMAP.md) parity matrix. It must be run on real cabinets before 1.0. The required round trip is:

**own LA → own EU → a third-party test server → back to own LA.**

1. Record stats and the operator's global leaderboard entry for a test card the operator controls, while the cab is configured for the operator's own LA server.
2. With cabs idle, switch to the operator's own EU server (URL and that server's tag secret as a pair). Confirm the in-game stats screen is unchanged.
3. Complete a test match on EU if possible. Confirm match history gained a row tagged with that EU server, totals went up by that match only, and the operator's community entry was updated rather than duplicated or reset.
4. With cabs idle, switch to a **third-party test** matchmaking server (URL and the secret that operator provided; the LA and EU secrets must not be sent). Confirm the stats screen is still unchanged, and that ChipId, AccessCode, and tag secrets are not present in traffic to that matchmaker beyond what the match protocol requires. If a test match is possible, history records that server's label or host, and the operator's global entry is still the same identity. Third-party board submission stays off unless this step is explicitly testing opt-in, in which case only a derived pseudonymous id is sent.
5. With cabs idle, switch back to the operator's own LA server. Confirm stats still include the matches played while away, history still tags those matches with the servers they were played on, the operator's community entry is still the same identity, and nothing was lost.
6. Confirm that a switch attempted during a queue or match is refused, and that an incompatible `/status` protocol version produces a launcher warning.

Reports of this test must not include private host addresses, tag secrets, or card identifiers.

## What stays out of this repository

- Private matchmaking host addresses, including any VPS addresses.
- Real tag secrets, admin tokens, and card identifiers.
- Raw leaderboard dumps that include those values.

Clean-room rules are unchanged: see [CLEAN-ROOM.md](CLEAN-ROOM.md).
