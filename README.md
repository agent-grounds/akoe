# Akoe

Akoe is a message gateway. It turns the owner's mail and chat accounts, and an
organization's channels, into ephor forge sources, and it gives other programs
a small command line to send and read through the same accounts. It decides
nothing about work: where a conversation belongs, whether it waits, and what to
answer stay with ephor, the consuming project and the person. [§GRUND-akoe](docs/grund.md#grund-akoe-no-person-carries-a-conversation-into-the-work-loop-or-an-answer-back)

## Status

Specifications and architecture, no code. The owner confirmed on 2026-10-07
that the grund is real and serves the workspace's intake rather than adding a
reason of its own, that Akoe is a project of its own under agent-grounds, and
that it builds on the libraries ZeroClaw wraps ([§DA-network-libraries](docs/decisions/architectural/DA-network-libraries.md#da-network-libraries-akoe-builds-on-the-libraries-zeroclaw-wraps-not-on-zeroclaw)). On
2026-10-13 the owner chose every network the specs name for the first version,
a daemon reached through local sockets ([§DA-local-daemon](docs/decisions/architectural/DA-local-daemon.md#da-local-daemon-the-owners-processes-reach-akoe-through-a-daemon-and-unix-sockets-not-through-sudo)), logs without
content, and app passwords over TLS for mail without OAuth; every point the
founding record left open is settled. The order of work is
tracked in agent-grounds/agent-grounds#15. No real account is added until agents
are confined; synthetic test accounts carry the proofs until then
([§FS-privacy.4.3](docs/functional-spec/FS-privacy.md#43-no-real-account-before-confinement)).

## Where to start

1. [docs/grund.md](docs/grund.md): why Akoe exists, what it does not do, when
   it shrinks, and why it is a project of its own. [§GRUND-akoe](docs/grund.md#grund-akoe-no-person-carries-a-conversation-into-the-work-loop-or-an-answer-back)
2. The functional specs, [docs/functional-spec/](docs/functional-spec/README.md):
   - [§FS-record](docs/functional-spec/FS-record.md#fs-record-akoe-reads-what-a-network-keeps-keeps-what-it-does-not-and-fails-when-it-cannot-vouch): what Akoe keeps per account, and how it shows an answer is
     complete;
   - [§FS-sending](docs/functional-spec/FS-sending.md#fs-sending-each-authorized-operation-is-admitted-once-attempted-at-most-once-and-recorded-as-accepted-rejected-or-unknown): how an operation is admitted once and recorded as accepted,
     rejected or unknown;
   - [§FS-names](docs/functional-spec/FS-names.md#fs-names-a-conversations-id-never-changes-a-room-names-a-venue-and-the-owner-is-one-person-across-accounts): conversation ids, rooms and identities;
   - [§FS-interfaces](docs/functional-spec/FS-interfaces.md#fs-interfaces-ephor-reads-akoe-through-the-forge-protocol-and-programs-through-a-small-command-line): `akoe-forge` for ephor, and the command line for programs;
   - [§FS-privacy](docs/functional-spec/FS-privacy.md#fs-privacy-a-person-pairs-akoes-own-user-holds-the-keys-and-no-real-account-enters-before-agents-are-confined): pairing, custody, and confinement;
   - [§FS-loop](docs/functional-spec/FS-loop.md#fs-loop-a-conversation-goes-round-ephors-loop-and-its-answer-comes-back-through-akoe): one conversation through ephor's loop and back, end to end.
3. The architecture, [docs/architecture/](docs/architecture/README.md), from
   [§AR-overview](docs/architecture/AR-overview.md#ar-overview-one-daemon-running-as-akoes-own-user-owns-every-account-thin-clients-carry-requests-to-it): the daemon, its sockets, the ledger, the store, the
   adapters, the forge, the tests and the crates.
4. The decision records [§DA-network-libraries](docs/decisions/architectural/DA-network-libraries.md#da-network-libraries-akoe-builds-on-the-libraries-zeroclaw-wraps-not-on-zeroclaw) and [§DA-local-daemon](docs/decisions/architectural/DA-local-daemon.md#da-local-daemon-the-owners-processes-reach-akoe-through-a-daemon-and-unix-sockets-not-through-sudo).

Every rule ends with `Source:`, naming the entries of the founding agora's
verdict (V-C certain, V-A agreed) and its rulings (V-D) that it rests on, and
says when a point was settled after it. Examples are synthetic.
