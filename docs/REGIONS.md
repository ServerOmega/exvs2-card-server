# Region switching

This document is the plan for moving a cabinet or a player between matchmaking regions (for example LA to EU, or EU back to LA) **without breaking card data, match history, or global leaderboard identity**.

This is not a database migration. Player stats live on the operator's card server. Changing which matchmaker a cabinet uses must not rewrite that data. Moving from the previous card server software to this one is covered in [MIGRATION.md](MIGRATION.md).

Related documents: [PLAN.md](PLAN.md), [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md).

## What stays put

Player stats live on the operator's card server, in the database this process owns:

- Cards and identity fields (ChipId and AccessCode)
- Profiles
- Unlocks
- Play counts
- Win/loss
- Class/rank
- Match history (each row keeps a region field, described below)
- Leaderboard submission state

Changing the matchmaking URL, or picking a different saved region in the launcher, **never** rewrites those records. A player who plays in LA, switches to EU, and switches back to LA still has the same card, the same totals, and the same history they had before the first switch, plus any matches they played in between.

## Matchmaking URL and tag secret travel together

Each region has its own tag secret. The matchmaking URL and the tag secret are a pair.

- The launcher already keeps a saved list of matchmaking servers. When the operator picks one, the launcher must write **both** the URL and that region's tag secret into this server's config (or the equivalent settings the admin API exposes).
- It must not leave a previous region's secret paired with a new URL, or the other way around.
- This server hands cabinets the configured URL and uses the configured secret, as the existing licensed matchmaker already expects.
- Private host addresses and real secrets are never examples in this repository. Region names such as LA and EU are fine in docs and in stored metadata.

## Global leaderboard identity is per card, not per region

Community leaderboards (the existing public page or Discord, not a site on this card server) identify a card once.

- The identity used to submit or update an entry is the preserved card identity, the same one [MIGRATION.md](MIGRATION.md) keeps unchanged.
- Region is **metadata on a submission**, not part of the identity key. A player who moves from LA to EU does not get a second row, and their existing totals are not reset.
- Resubmission is **idempotent**: a submit or resubmit with the same card identity updates the existing entry. It does not append a duplicate and does not zero the row.
- After a region switch, this server may resubmit the card's current cumulative stats with the new region metadata, so the board reflects the move without creating a new identity.

### Which region a card is shown under

The recommended rule, to be confirmed or replaced before M4 leaderboard work is considered done (see [PLAN.md](PLAN.md)):

- A card's **display region** is the region of its most recent accepted leaderboard submission.
- Cumulative stats remain a single entry for that card. They are not split, reset, or copied when the display region changes.
- A regional board may filter to entries whose current display region matches, or to matches whose history region matches, without inventing a second identity.
- Switching back to a previous region, followed by the idle-time resubmit below, sets the display region back to that previous region. Totals do not rewind.

"Best rank" is not the default, because it would keep showing a player in a region they have left.

## Match history keeps a region field

Each match history row stored on this card server records the region that was in use at the time of the match (a region name or saved-server identifier, never a private host address).

- History is append-only with respect to completed matches. Switching region does not delete, rewrite, or move old rows.
- A player who played in LA, then EU, then LA again keeps all three groups of matches, each tagged with the region they were played in.
- Views in the launcher and the CLI can show the region next to each match.

## When a switch is allowed

A region switch is allowed only when cabinets are **idle**:

- No cabinet is in a matchmaking queue.
- No cabinet is in a match.
- No card login or save is in progress.

An in-progress queue or match cannot be migrated to another region. The launcher, and the admin settings-set path, must refuse a MatchUrl or tag-secret change until those conditions are met. If a cab is busy, the operator waits or stops the queue first.

## Switching back

Switching back to a previous region restores prior matchmaking configuration with no data loss:

- Card data was never moved, so it does not need to be restored.
- The launcher writes the previous URL and that region's tag secret as a pair.
- An idle-time idempotent resubmit updates the display region on the community board. It does not reset totals or create a duplicate.
- Match history still contains the matches played in the other region, tagged as such.

After the round trip, the card's stats screen matches what the operator would expect from "the same card, plus any matches played while away."

## Suggested operator procedure

1. Confirm every cab is idle.
2. In the launcher, pick the destination saved matchmaking server. The launcher sets MatchUrl and that region's tag secret together.
3. Confirm this server is using the new pair (CLI or launcher status).
4. Optionally resubmit current cumulative stats so the community board's display region updates.
5. Start play in the new region.
6. To go back, repeat from step 1 with the previous saved server.

No database copy and no freeze of the card server itself is required for a region switch, because the card database is not the thing that is changing.

## Compatibility tests

The official check is the region-switch row in the [ROADMAP.md](ROADMAP.md) parity matrix. It must be run on real cabinets before 1.0:

1. Record stats and the community leaderboard entry for a test card the operator controls, while the cab is configured for LA.
2. With cabs idle, switch to EU (URL and tag secret as a pair). Confirm the in-game stats screen is unchanged.
3. Complete a test match in EU if possible. Confirm match history gained a row tagged EU, totals went up by that match only, and the community entry was updated rather than duplicated or reset.
4. With cabs idle, switch back to LA. Confirm stats still include the EU match, history still shows it as EU, the community entry is still the same identity, and the display region returns to LA after the resubmit.
5. Confirm that a switch attempted during a queue or match is refused.

Reports of this test must not include private host addresses, tag secrets, or card identifiers.

## What stays out of this repository

- Private matchmaking host addresses, including any VPS addresses.
- Real tag secrets, admin tokens, and card identifiers.
- Raw leaderboard dumps that include those values.
