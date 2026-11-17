# Clean-room process

This project must be a clean-room reimplementation. The upstream card server that some operators build today has **no license**. We can study how the *game* behaves on the network. We cannot copy that upstream project.

If a contribution cannot show a clean provenance, it does not go in.

## The rule, in one paragraph

Specs are written in our own words from (1) traffic captured between the official game and a card server on cabinets the operator owns, and (2) public documentation. Implementers write new code from those specs and from this repository's documents. Implementers must not open the unlicensed upstream source, patches that include its context lines, decompiled or disassembled copies of it, or private trees derived from it. No code, file layout, class name, comment, or data from that upstream project may be copied into this repository.

## Roles

### Spec writers

Spec writers observe the game and write the behavior spec.

They may:

- Capture traffic from the operator's own cabinets while the official game talks to a card server the operator already runs.
- Watch the cabinet and any operator- or player-facing screens of the running system as a *user* (boot to attract mode, card login, save, menus, online settings, account or stats screens). Observation of a current browser UI is allowed only as a user, to learn what tasks exist. The replacement for those tasks in this project is the local admin API, the CLI, and the launchers, not a new web UI, and no upstream UI source may be copied.
- Read public documentation, including the licensed documents in [exvs2-matchmaking](https://github.com/ServerOmega/exvs2-matchmaking), [exvs2-launcher](https://github.com/ServerOmega/exvs2-launcher), and [exvs2-launcher-linux](https://github.com/ServerOmega/exvs2-launcher-linux).
- Write functional descriptions: message categories, when the cabinet sends them, what the cabinet needs in order to proceed, and what should be stored.

They must not:

- Paste source, comments, identifiers, file trees, or database typings from the unlicensed upstream project into a spec.
- Treat an upstream handler name, route table, or class model as the spec.
- Commit raw captures, card identifiers, secrets, or private host addresses.

A person who has recently read the upstream source should not be the sole implementer of the same area. If the same person must do both jobs in a small project, they should write the spec first, set it down, and implement only from the written spec, with a provenance note that says so. Prefer a second person to review the implementation against the spec only.

### Implementers

Implementers turn a written spec into code in this repository.

They may:

- Read this repository, its issues and pull requests, and the licensed public docs of the matchmaking and launcher projects.
- Ask spec writers for a clarification, which is then written back into the spec.
- Write tests from the spec, including fixtures built from *redacted* capture descriptions (expected request and response shapes with secrets removed).

They must not:

- Open the unlicensed upstream source tree, or any working copy of it.
- Open patches, pull requests, or diffs whose context is that upstream source.
- Open decompiled, disassembled, or reuploaded copies of that server.
- Port files, types, comments, or "just the idea of the layout" from memory of that codebase.
- Add game executables, assets, extracted keys, byte offsets, or hashes of official binaries.

If an implementer has already seen the upstream source, they must say so in the pull request and should avoid areas where that memory would do the design work. Reviewers may ask for a rewrite from the spec.

## Where the spec comes from

The behavior spec is a description of **what the game needs**, not a description of how any existing server is written.

Each section should be written as a *behavior spec to be verified by packet capture against the real game*. Early drafts may list message categories at a functional level, for example:

- Boot and network checks the cabinet runs before attract mode.
- Card registration, login, and save or load of player data.
- Online settings and event data the cabinet reads after it is up.
- Handoff to matchmaking (the configured matchmaking URL and the shared tag secret, compatible with the existing matchmaker).
- Submission of results or leaderboard data, if the cabinet sends them to the card server.
- Operator management tasks (status, card management with identifiers masked, backups, settings, logs, leaderboard and match history views) that the launchers and the CLI will perform through a local admin API. This server does not ship a web UI. Public community stats stay on the existing global leaderboards, not on this process.

Exact routes, field layouts, content types, and error handling are filled in from captures. If a public document and a capture disagree, the capture of the game wins, and the spec is updated.

The first drafts of those categories may live in this repo as ordinary documents once they contain no secrets. Raw captures never do.

## Provenance notes

Every spec section, and every implementation pull request, needs a short provenance note.

For a spec section, record:

- The date it was written or last checked.
- What was observed (for example: "cabinet boot to attract mode, local card server, HTTP traffic, 2026-11-16").
- Which public documents, if any, were used, with links.
- Who wrote it, and whether they have ever read the unlicensed upstream source.
- That identifiers and secrets were redacted.

For an implementation pull request, record:

- Which spec sections it implements.
- A statement that the author did not consult unlicensed upstream source while writing it.
- Any prior exposure to that source, if it exists.
- The Developer Certificate of Origin sign-off described below.

Reviewers should reject a change that looks copied from upstream, that introduces upstream names without a capture-based reason, or that lacks a provenance note.

## Contribution rules

1. Read this file before you open a pull request.
2. Do not add game files, assets, executables, teardowns of official binaries, byte offsets, or hashes of those binaries.
3. Do not add card identifiers, access codes, boot tokens, webhook URLs, admin API tokens, private server addresses, or raw packet captures.
4. Do not add source, patches, or verbatim layouts from the unlicensed upstream card server.
5. Keep secrets out of logs, sample config, and tests. Use placeholders.
6. Prefer small pull requests that map clearly to a spec section.
7. Sign off every commit with the Developer Certificate of Origin.

Sample config in the repo may use loopback addresses and placeholder names only. Never publish the addresses of private matchmaking hosts.

## Developer Certificate of Origin

This project uses the [Developer Certificate of Origin](https://developercertificate.org/), version 1.1. By adding a `Signed-off-by` line to a commit, you certify:

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.

Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

Add a line at the end of each commit message:

```
Signed-off-by: Your Name <your@email.example>
```

Use the name and email you intend to be known by. `git commit -s` adds the line for you.

If a tool helps write the change, also keep the project trailer when it applies:

```
Co-authored-by: Grok Bot <grok-bot@users.noreply.github.com>
```

A `Signed-off-by` on a commit that contains unlicensed upstream material does not make that material acceptable. The clean-room rules still apply.

## What stays out of git, always

| Material | Why |
|---|---|
| Unlicensed upstream source, patches with its context, decompiled binaries | No license to copy or redistribute |
| Game executables, assets, extracted keys, byte offsets, official hashes | Not ours to publish; not needed in a protocol spec |
| Raw packet captures | Can contain card identifiers and private addresses |
| Chip IDs, access codes, boot tokens, tag secrets, admin API tokens, webhook URLs | Secrets and personal player data |
| Private server host addresses | Do not publish operator infrastructure |

Local capture files and private spec notes should live outside this clone, or in ignored paths such as `captures/` and `spec-private/`. Those paths are listed in `.gitignore` on purpose.

## Reporting a problem

If you think a commit already in the repository copied unlicensed material, or leaked a secret, open a private report to the maintainers if you can, or an issue that does **not** repeat the leaked material. We will remove or rewrite the offending content and rotate any exposed secret.
