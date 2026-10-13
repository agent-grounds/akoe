# AR-overview: one daemon, running as Akoe's own user, owns every account; thin clients carry requests to it

How Akoe is put together. One long-running daemon, `akoed`, holds every
session, store and ledger as Akoe's own Unix user ([§FS-privacy.2.2](../functional-spec/FS-privacy.md#22-akoes-own-unix-user)). ephor and
programs, which run as the owner, reach it through two thin clients that hold
nothing: `akoe-forge` for ephor ([§FS-interfaces.1](../functional-spec/FS-interfaces.md#1-one-executable-for-ephor)) and `akoe` for programs
and people ([§FS-interfaces.4](../functional-spec/FS-interfaces.md#4-the-command-line-for-programs), [§FS-interfaces.7](../functional-spec/FS-interfaces.md#7-commands-for-a-person)). The decision and its
alternatives are [§DA-local-daemon](../decisions/architectural/DA-local-daemon.md#da-local-daemon-the-owners-processes-reach-akoe-through-a-daemon-and-unix-sockets-not-through-sudo).

## 1. The parts

```text
 owner's uid                         │  akoe's uid
                                     │
 ephor ──argv/stdin──▶ akoe-forge ───┼──▶ /run/akoe/read.sock ─┐
                                     │                         │
 programs ───────────▶ akoe ─────────┼──▶ /run/akoe/send.sock ─┼──▶ akoed ──▶ account actors ──▶ adapters ──▶ networks
                                     │                         │                 │
 person (sudo -u akoe) ▶ akoe ───────┼──▶ /run/akoe/admin.sock ┘                 └──▶ one SQLite store per account
```

| Part | Runs as | Holds | Lives |
|---|---|---|---|
| `akoed` | `akoe` | sessions, credentials, stores, configuration | a system service, always |
| `akoe-forge` | ephor's uid | nothing | one call |
| `akoe` | the caller's uid | nothing | one call |
| `signal-cli` | `akoe` | one Signal account's linked device | a child of `akoed`, one per Signal account |

The clients never open a store, read a credential or speak to a network. Every
fact they print comes from `akoed` over a socket ([§AR-rpc](AR-rpc.md#ar-rpc-clients-speak-one-json-request-per-connection-over-three-sockets-whose-paths-are-the-permissions)).

**Example.** ephor's refresh runs `/usr/bin/akoe-forge messages` as the owner.
`akoe-forge` reads ephor's request on stdin, sends `messages` for account
`mail-me` over `read.sock`, and prints `akoed`'s answer. It never sees the
IMAP password, which only `akoed` can read.

## 2. Users, groups and sockets

`akoed` runs as the system user `akoe`, whose home is `/var/lib/akoe`; that
home is the `~` of [§FS-privacy.2.1](../functional-spec/FS-privacy.md#21-configuration-secrets-and-record). Its sockets live in `/run/akoe/`, which
systemd creates for it:

| Socket | Mode and group | Carries |
|---|---|---|
| `read.sock` | `0660 akoe-read` | `messages`, `thread`, `operation`, `wait`, `status`, `config show`, `config sources` |
| `send.sock` | `0660 akoe-send` | `reply`, `send` |
| `admin.sock` | `0600 akoe` | the person's commands: `pair`, `retire`, `purge`, `resolve`, `ack-gap`, `reset-identity`, `backup`, `restore`, `release-restore`, `reload` |

The owner is in `akoe-read` and `akoe-send`. A confined worker's sandbox
denies `/run/akoe` ([§FS-privacy.4.2](../functional-spec/FS-privacy.md#42-what-confinement-means)); the split lets a deployment grant reading
without sending. A person's command runs as `akoe` through `sudo -u akoe`, at a
terminal and behind the owner's password, which is what makes it a person's
act ([§FS-privacy.1](../functional-spec/FS-privacy.md#1-pairing-is-a-persons-act), [§FS-record.4.1](../functional-spec/FS-record.md#41-retirement-is-a-persons-act)).

`akoed` reads each connection's peer credentials (`SO_PEERCRED`) and records the
caller's uid on every operation it admits. The uid is evidence for a person
reading the ledger, never an authorization: the socket's permissions are the
authorization.

**Example.** An agent in a sandbox that hides `/run/akoe` runs `akoe send …`;
the client cannot connect and exits 69 with `akoe: /run/akoe/send.sock: no such
file or directory`. The owner at a terminal runs `akoe retire mail-me k1` and is
told to run it as `sudo -u akoe akoe retire mail-me k1`.

## 3. Account actors

Each configured account is one actor inside `akoed`: a task that owns the
account's store connection, its adapter and, for a keeper, its lease
([§AR-networks.3](AR-networks.md#3-the-lease)). Every read and write of that account goes through the
actor's inbox, so one account's store has exactly one writer and admission
([§AR-ledger.2](AR-ledger.md#2-admission-is-one-transaction-in-the-accounts-actor)) needs no lock beyond the actor itself. Accounts share nothing
but the process: one account's adapter failing, or its store filling its quota,
leaves the other actors answering ([§FS-interfaces.2.1](../functional-spec/FS-interfaces.md#21-one-source-per-account)).

A source ([§FS-interfaces.2](../functional-spec/FS-interfaces.md#2-sources)) is one account, or one named scope of a split account.
A request names the account and the scope; `akoed` routes it to the account's
actor, which answers for that scope.

**Example.** `wa-me-direct` and `wa-me-rooms` both route to the `wa-me` actor.
When the WhatsApp socket drops, the actor loses its lease and both sources fail;
the `mail-me` actor goes on answering.

## 4. What one request does

### 4.1 `messages`

1. `akoe-forge` sends `messages {source}` on `read.sock`.
2. The account's actor establishes that it can vouch: for a reader, by a read
   of the network that completes now ([§AR-networks.4](AR-networks.md#4-mail), [§AR-networks.5](AR-networks.md#5-slack-discord-and-matrix)); for a
   keeper, by its current lease and the absence of an unacknowledged gap.
3. The actor writes what the read observed (issued ids, aliases, generations,
   revisions, the owner's messages) in one transaction, then builds the answer
   from the store and, for mail, from the bodies just read, which are never
   written down ([§FS-record.1.1](../functional-spec/FS-record.md#11-no-listener-and-no-body-store)).
4. `akoe-forge` prints the answer in ephor's shape ([§AR-forge.2](AR-forge.md#2-messages)).

If step 2 fails, the actor answers an error naming the account and the cause,
and `akoe-forge` exits non-zero with it ([§FS-record.1.2](../functional-spec/FS-record.md#12-the-read-is-the-observation)).

### 4.2 `reply` and `send`

1. The client sends the request on `send.sock`.
2. The actor admits it, or refuses it, in one transaction ([§AR-ledger.2](AR-ledger.md#2-admission-is-one-transaction-in-the-accounts-actor)).
3. The actor marks the operation `attempting`, commits, and hands the frozen
   bytes to the adapter for one attempt ([§AR-ledger.3](AR-ledger.md#3-the-attempt-protocol)).
4. The adapter returns `accepted`, `rejected` or `unknown`; the actor records it,
   with the generation it advances, in one transaction.
5. The client prints the outcome.

The attempt belongs to `akoed`, not to the client. ephor killing a slow
`akoe-forge` at its timeout ends the client only; the attempt finishes and its
outcome is recorded, and ephor's replay later reads it ([§FS-sending.5.1](../functional-spec/FS-sending.md#51-an-identical-request-returns-the-recorded-outcome)).

**Example.** An SMTP submission takes 40 seconds and ephor's timeout is 30.
ephor kills `akoe-forge`; `akoed` records operation 7 `accepted` at second 40.
The owner presses `p` again; ephor replays the saved request, and the actor
answers operation 7's acceptance without opening a session.

## 5. Startup and recovery

On start, `akoed` loads the configuration, opens each account's store, and runs
the ledger's recovery before it accepts any request: an operation still marked
`attempting` becomes `unknown`, and one still `admitted` becomes `rejected` with
"not attempted: Akoe stopped before the attempt" ([§AR-ledger.4](AR-ledger.md#4-recovery-after-a-crash)). Each
keeper then starts without a lease and earns one ([§FS-record.3.2](../functional-spec/FS-record.md#32-the-vouching-lease)). Sockets
open last, so no client is answered from a state recovery has not settled.

`akoed` never resubmits anything on start or at any other time
([§FS-sending.9](../functional-spec/FS-sending.md#9-what-akoe-never-does-when-sending)). An adapter that loses its connection reconnects with a backoff;
that is a receiving session reconnecting, never an operation retried.

**Example.** The host loses power while operation 12 is mid-submission. On
boot, recovery records operation 12 `unknown`, the WhatsApp actor starts without
a lease, and `wa-me` fails as unable to vouch until its first completed sync.

## 6. Logs carry no content

`akoed` logs to the journal. A log line names an account, a source, an internal
conversation key (an integer), an operation number, a count, a duration or an
outcome. It never carries a message's words, a subject, an address, a number, a
network id, a pairing code or a token ([§FS-privacy.2.3](../functional-spec/FS-privacy.md#23-logs-carry-no-content)). A diagnosis returned to
the caller may quote content, because the caller could read it anyway; it is
not written to the log.

**Example.** A stale refusal prints `mail-me: stale: newer message from
orders@supplier.example.org: "€1,000"` to the caller, and the journal records
`mail-me conv=17 op=- refused=stale`.
