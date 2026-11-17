# Spec draft: Custom Match rooms

Status: **draft behaviour spec**, 17 November 2026 (America/Los_Angeles). It describes what the **game** needs. It is
not a description of how any existing server is written. Verify every field against packet captures before
implementing.

## Provenance

- **Sources:** behaviour seen on real cabinets and a Live Monitor (LM) on 16–17 Nov 2026; request and reply shapes
  from the game executable's own protobuf descriptors (published as `analysis/proto/from_exe/` in
  [exvs2-matchmaking](https://github.com/ServerOmega/exvs2-matchmaking)); official VS.Mobile help pages for what
  rooms offer players; and the licensed documents in exvs2-matchmaking, especially `docs/CUSTOM_ROOMS.md` and
  `docs/FINDINGS-2026-11-17.md`.
- **Author:** Grok Bot, an AI assistant working for ServerOmega. This author **has read** the unlicensed upstream
  card server while writing the matchmaker patches. Under [CLEAN-ROOM.md](../CLEAN-ROOM.md), this text is written in
  our own words from game behaviour only, and it contains no upstream identifiers, routes, config keys or layouts.
  An implementer who has never seen the upstream source should implement from this spec and from captures, and a
  reviewer should check the result against this spec only.
- **Redaction:** this spec contains no room codes, card ids, access codes, chip ids, serials, tokens or addresses.

## What the player sees

1. A player makes a room (name, solo or team, open and close time, class range, fes rule, member limit) and gets a
   short room code to share. Friends join with that code.
2. On the cabinet, the player touches the card and chooses **Custom Match**. The cabinet searches only with
   players from that same room.
3. Players in one room may use **different card servers**, as long as those card servers share one matchmaker.

## Messages the game sends

| Who | Message | Needs from the server |
|---|---|---|
| LM, after a card touch on its Custom Match screen | **LoadRoom** (id 418): card identity (access code + chip id) | The rooms this card belongs to. **The reply status must be 1**, or the LM shows no rooms. Each room has an id, a registration code, a room kind, a matching type (solo / team / mix), a start and end time in **Unix seconds**, and display fields (name, class and win-rate bounds, fes rule, timer, limits). |
| LM | **RegisterRoom** (id 419): card + chosen room id | Store that room as the card's **selected room**. A reply status of 0 means OK. |
| Cabinet | **PreLoadCard** (card load) | When the card has a selected room, include its **matching tag** (the room). **If no room is selected, leave the matching tag out.** The cabinet then skips Custom Match. |
| Cabinet | Matchmaking entry for a custom-room world | Handled by the matchmaker. Worlds 6–9, 24 and 26–33 carry the room id, and the matchmaker must keep each room id in its own queue. |

### Room kinds the LM shows

- Kind **2** (private): a list of up to 7 rooms.
- Kind **4** (public): one special slot (the yellow パブリックルーム box). It shows only the time, mode and class.
- The LM skips kinds 1 and 3. It also drops any room whose start–end window doesn't include the LM's current time. If
  nothing remains, it shows its "no room" dialog.

### Fields the cabinet actually uses

Most room options are display-only on the LM. The cabinet itself uses the open and close window, solo vs team,
the fes rule and the room id. Treat the rest as display data until a capture shows otherwise.

## What a card server must provide

- **Selected room per card**, settable in two ways: from the LM (RegisterRoom) and from the operator or player tools
  (launcher or admin API). **The last selection wins.** Creating or joining a room should also select it.
  Leaving or closing a room clears the selection. This is what lets one LM serve cabinets on different card
  servers, because nobody has to touch a card on "their" LM.
- **A shared room registry** that every card server on the same matchmaker can reach. Room lookups by code must not
  put the code in a URL (use a request body), and logs must mask codes. Calls between servers use the existing
  shared tag secret.
- **A configurable time zone** for open and close times and the daily reset. Do not use the host's UTC clock. A UTC
  daily reset closed rooms at 4 PM Pacific in testing.
- **Flexible time input** (`HH:mm`, `HH:mm:ss`, 12-hour AM/PM, `24:00`). Browsers and locales differ.

## Matchmaker behaviour (for reference)

- Each room is its own queue.
- **Dual Select is a 2v2 rule.** Each player picks two units and plays one of them in a normal 2v2, so a room with
  no CPU partner waits for 4 humans.
- **One (0–4) and One XU (0–4) are 1v1 rules.** Two solo players in such a room are paired right away, one per side,
  with no CPU.

## Verified so far

- 17 Nov 2026, 12:33 PM PT: 4 cabinets on 2 card servers with 1 LM, in one room. The matchmaker formed a 4-human 2v2
  with no CPU, and every cabinet got "match found" within about 3 s.
- Not yet verified on cabinets: 1v1 pairing in a One-rule room, and whether results save after a custom-room match.
