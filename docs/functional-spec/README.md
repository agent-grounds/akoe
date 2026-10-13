# Functional specs

What Akoe does, one area per file. Each H1 declares an `FS-<slug>` ID, and its
numbered sections are citable. Every section ends with the verdict entries or
rulings it rests on (`Source:`). A point the founding record left open and that
was settled on 2026-10-13, by the owner or by this repository's own pass over
the record, says so in its `Source:` line; none is left open.

| ID | Subject |
|---|---|
| [§FS-record](FS-record.md#fs-record-akoe-reads-what-a-network-keeps-keeps-what-it-does-not-and-fails-when-it-cannot-vouch) | the per-account record, mail read at refresh, keepers and their lease, retirement, receiving without side effects |
| [§FS-sending](FS-sending.md#fs-sending-each-authorized-operation-is-admitted-once-attempted-at-most-once-and-recorded-as-accepted-rejected-or-unknown) | operations, the ledger, outcomes, the thread generation, the stale refusal, the five cases, the allowlist |
| [§FS-names](FS-names.md#fs-names-a-conversations-id-never-changes-a-room-names-a-venue-and-the-owner-is-one-person-across-accounts) | conversation ids, threading, rooms, accounts and the owner's identities |
| [§FS-interfaces](FS-interfaces.md#fs-interfaces-ephor-reads-akoe-through-the-forge-protocol-and-programs-through-a-small-command-line) | `akoe-forge` for ephor, sources, the command line for programs |
| [§FS-privacy](FS-privacy.md#fs-privacy-a-person-pairs-akoes-own-user-holds-the-keys-and-no-real-account-enters-before-agents-are-confined) | pairing, custody, the user and its sockets, logs, confinement, and what Akoe expects of ephor |
| [§FS-loop](FS-loop.md#fs-loop-a-conversation-goes-round-ephors-loop-and-its-answer-comes-back-through-akoe) | one conversation from the network through ephor's feed and its answer back, end to end |

This index is navigational — citations should target the spec ID directly,
never this file.
