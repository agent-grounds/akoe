# FS-interfaces: ephor reads Akoe through the forge protocol, and programs through a small command line

The two ways in. ephor, the intake that [§GRUND-akoe](../grund.md#grund-akoe-no-person-carries-a-conversation-into-the-work-loop-or-an-answer-back) serves, reaches Akoe through
the forge protocol it already ships. Other programs reach it through a small
structured command line. There is nothing else. How the two fit into ephor's
loop, end to end, is [§FS-loop](FS-loop.md#fs-loop-a-conversation-goes-round-ephors-loop-and-its-answer-comes-back-through-akoe).

## 1. One executable for ephor

### 1.1 `akoe-forge` takes ephor's subcommands

ephor runs one executable, `akoe-forge`, for every Akoe source. Each source's
`command` is that executable's path, and ephor appends the protocol subcommand
as one argument. ephor does not split a `command` string, so the executable
takes the subcommands directly. Akoe declares and answers three: `capabilities`,
`messages` and `reply`. Any other exits 64.

**Example.** `{"provider": "mail-me", "command": "/usr/bin/akoe-forge", …}`
and `{"provider": "wa-me", "command": "/usr/bin/akoe-forge", …}` work side by
side. `"command": "akoe forge"` fails as a missing executable.

Source: V-A10, V-C13.

### 1.2 Each source has its own block

ephor hands each source's configuration block to `akoe-forge` verbatim, as the
request's `config`. The block names the Akoe account with the key `account`,
and, for a split account ([§FS-interfaces.2.2](FS-interfaces.md#22-a-split-for-ownership-or-for-a-selector-never-for-isolation)), the scope with the key `scope`,
which names a scope declared in that account's configuration. `provider` is
ephor's name for the source's rows, and Akoe never reads it. Each source has a
`provider` name of its own, because ephor keys a source's results and replies by
that name: two sources both named `akoe` would overwrite each other.

**Example.** The blocks `{"provider": "wa-me-direct", "command": "/usr/bin/akoe-forge",
"account": "wa-me", "scope": "direct"}` and `{"provider": "wa-me-rooms", …,
"account": "wa-me", "scope": "rooms"}` name the same account and different
scopes. `akoe-forge messages` answers each for its own scope.

Source: V-A10, V-C13; settled 2026-10-13.

### 1.3 Capabilities do not depend on configuration

ephor probes capabilities with an empty configuration, before every fetch,
reply and dry run, so `akoe-forge capabilities` declares the same surface for
every source and answers without reaching any network: `messages`, `replies`,
and `reply_reconciliation`. The last promises what the ledger already does: the
lookup of an earlier operation comes before any freshness test, the same
descriptor and words recover that operation, and a changed payload on a used
generation is refused ([§FS-sending.5](FS-sending.md#5-admission)). With it, ephor replays a send whose
answer it never received instead of holding it for a person.

What one account cannot do, it refuses with a diagnosis. An account that cannot
send, or whose `replies` setting is off ([§FS-sending.8](FS-sending.md#8-the-sender-allowlist-guards-against-mistakes)), puts no reply
descriptor on its threads, and ephor then offers its drafts as text to copy.

**Example.** `akoe-forge capabilities` given `{"config": {}}` answers
`{"messages": true, "replies": true, "reply_reconciliation": true}`. A `reply`
to a thread of a read-only Slack source exits non-zero with `slack-ag-rooms:
refused: this account cannot send`.

Source: V-A10; ephor's FS-001-forge-interface (reply reconciliation, 2026-10-08); settled 2026-10-13.

## 2. Sources

### 2.1 One source per account

Each account is its own ephor source. A source is ephor's unit of failure,
staleness and routing, so one network failing leaves the other accounts'
sources answering.

**Example.** The WhatsApp keeper loses its lease. `wa-me` fails and its rows
show stale; `mail-me` still answers, and its rows stay current.

Source: V-A09.

### 2.2 A split for ownership or for a selector, never for isolation

An account is split into more than one source only where ownership, or a
recipe's selector, must tell its scopes apart; never to isolate failures alone.
Sources split from one account share its connection and its credentials, and
fail together when those fail.

**Example.** `slack-ag-direct` and `slack-ag-rooms` share one Slack token. When
Slack's API cannot be reached, both fail. When one channel's history cannot be
read, only `slack-ag-rooms` fails. A recipe with `"when": {"sources":
["slack-ag-direct"]}` answers direct conversations and not channels.

Source: V-A09.

### 2.3 A budget of about ten sources

One owner has roughly six to ten sources; ten is a budget for configuration,
not a measurement. Akoe generates ephor's source blocks from its own
configuration with `akoe config sources`, which prints a JSON array of blocks,
each with `provider`, `command`, `account`, `scope` where the account is split,
and a `timeout_seconds` fit for the network. The output depends on Akoe's
configuration alone, so printing it again after an unrelated change gives the
same bytes for every source that did not change; ephor will not recover a saved
send through a block that changed. Akoe never writes ephor's configuration, and
whether a source is private, and where its rows go, stay ephor's
([§FS-privacy.5](FS-privacy.md#5-what-akoe-expects-of-ephor)). Akoe's acceptance includes a generated ten-source fixture.

**Example.** ephor's `status.json` holds blocks such as `mail-me`,
`wa-me-direct`, `wa-me-rooms`, `signal-me`, `slack-ag-direct`, `slack-ag-rooms`
and `discord-ag-rooms`, each pasted from `akoe config sources`, each with its own
`provider` and the same `command`.

Source: V-A10; settled 2026-10-13.

## 3. The forge answers

### 3.1 `messages`

`akoe-forge messages` answers the conversations in the source's declared scope
([§FS-record.7](FS-record.md#7-declared-scope)), except the retired ([§FS-record.4.1](FS-record.md#41-retirement-is-a-persons-act)). Each is whole: every
message Akoe holds for it, in order ([§FS-record.4.2](FS-record.md#42-nothing-is-deleted-or-cut-automatically)), each with its author
([§FS-names.4.4](FS-names.md#44-an-author-is-reported-by-address)), whether it is the owner's ([§FS-names.4.2](FS-names.md#42-the-owners-identities-and-mine)), the time it was
first sent, its words as last revised ([§FS-record.6.2](FS-record.md#62-edits-are-revisions-deletions-are-tombstones)), and its attachments
by name, type and size, never their contents; its room, absent for a direct
conversation ([§FS-names.3](FS-names.md#3-rooms)); its reasons ([§FS-names.3.2](FS-names.md#32-a-direct-conversation-has-no-room-and-no-reason)); and, where the
account can send, a reply descriptor that carries the thread's generation
([§FS-sending.4](FS-sending.md#4-the-thread-generation)). An account that is observed and quiet answers `[]`. When the
source cannot vouch for the account ([§FS-record.1.2](FS-record.md#12-the-read-is-the-observation), [§FS-record.3.2](FS-record.md#32-the-vouching-lease)), it
fails rather than answers.

ephor recognises a message by its author, time and words, and a thread by its
first message. So a message keeps its first-sent time across edits, a
conversation's order and first message never change, and no author has two
messages at the same instant in one conversation.

**Example.** A supplier thread is answered as
`{"id": "k1", "title": "Order 12", "updated_at": "2026-10-07T09:14:00Z",
"reasons": [], "threads": [{"messages": [{"author": "orders@supplier.example.org",
"mine": false, "text": "Can you confirm order 12?", "when":
"2026-10-07T09:14:00Z"}], "reply": {…}}]}`, where the `reply` object is
Akoe's own and opaque to ephor.

Source: V-A04, V-A05, V-A06, V-A07, V-C11, V-C15; ephor's FS-001-forge-interface (2026-10-08); settled 2026-10-13.

### 3.2 `reply`

`akoe-forge reply` takes ephor's request as it ships: the target, which is the
descriptor Akoe put on the thread, and the text. The forge request gains no
field. Akoe admits the request as [§FS-sending.5](FS-sending.md#5-admission) says and answers:

| Outcome | Answer |
|---|---|
| `accepted`, now or earlier | `{"status": "accepted"}` |
| `unknown`, now or earlier | `{"status": "unknown", "note": "<account>: operation <n> may have reached the network; resolve it with akoe resolve"}` |
| `rejected`; refused as blocked, stale, a used generation, or not allowed | a non-zero exit with a one-line diagnosis that names the operation or the newer message |

ephor refuses a stale draft itself before it calls Akoe, from what its last
refresh showed; Akoe's stale refusal ([§FS-sending.5.3](FS-sending.md#53-a-stale-never-sent-draft-is-refused)) is the floor beneath
it, for a message Akoe knows of and ephor has not yet fetched. What ephor does
with a definite refusal is [§FS-privacy.5.5](FS-privacy.md#55-a-definite-refusal-releases-the-saved-send).

**Example.** The supplier's "€1,000" reached Akoe's store after ephor's last
refresh. `p` on the €100 draft passes ephor's own check, and Akoe exits 1 with
`mail-me: stale: newer message from orders@supplier.example.org: "€1,000"` on
stderr. Nothing is sent.

Source: V-A08, V-A15, ruling V-D02; ephor's FS-001-forge-interface (typed reply outcomes, 2026-10-08); settled 2026-10-13.

## 4. The command line for programs

A program reaches Akoe by running a command that prints a structured result.
Four commands make the surface: send, read a conversation, operation status,
and wait for new mail. Each prints JSON on stdout. Their exit codes are fixed,
so a program that reads only the exit code never takes an `unknown` for success:

| Exit | Meaning |
|---|---|
| 0 | done; for a send, `accepted` |
| 2 | refused; nothing was admitted |
| 3 | `rejected` |
| 4 | the account cannot vouch now |
| 5 | no such account, conversation or operation |
| 6 | `unknown` |
| 69 | Akoe is not running, or cannot be reached from here |
| 75 | `akoe wait`: nothing new |

A refusal also prints its diagnosis on stderr. A consuming project keeps its own
records, approvals and stages; Akoe carries only the transport.

**Example.** crop-warden's program state runs `akoe wait --account
mail-sourcing --after <cursor>`, then `akoe thread` for the lead it is
following, and records lead 12 as sent only once `akoe send` exits 0.

Source: V-A17, V-A18, V-A19; settled 2026-10-13.

### 4.1 `akoe send`

`akoe send --account <account> --as <project> --key <key> --body-file <file>
[--label <label>]` and one destination:

- a new conversation: `--to <address>`, repeatable; for mail also `--cc
  <address>`, repeatable, and `--subject <text>`, which a chat account refuses;
- a reply within a conversation: `--reply-to <descriptor>`, the descriptor
  `akoe thread` printed, admitted as ephor's reply is
  ([§FS-sending.7](FS-sending.md#7-new-outbound-mail-takes-a-caller-key)).

The recipients, subject, label and body are the operation's payload, frozen
together: the same key with any of them changed is refused. Akoe sets the
sender, the date and the message id itself. The command runs under the caller's
key and the account's sender allowlist ([§FS-sending.8](FS-sending.md#8-the-sender-allowlist-guards-against-mistakes)), and prints
`{"operation", "key", "outcome", "accepted_at", "attempted_now",
"conversation"}`.

**Example.** `akoe send --account mail-sourcing --as crop-warden --key
lead-12-outreach-1 --to orders@supplier.example.org --subject "Seed potatoes"
--label suppliers --body-file F` prints `{"operation": 41, "key":
"lead-12-outreach-1", "outcome": "accepted", "accepted_at":
"2026-10-13T14:02:11Z", "attempted_now": true, "conversation": "k9"}` and exits
0. crop-warden stores `k9` against lead 12.

Source: V-A17, V-A20; settled 2026-10-13.

### 4.2 `akoe thread`

`akoe thread --account <account> <conversation>` prints the conversation as
JSON, with the replies that came back into it ([§FS-names.2](FS-names.md#2-threading-generic-mail)), its reply
descriptor where the account can send, and each message's revision and whether
it was deleted.

**Example.** Two days after the first mail to lead 12, `akoe thread --account
mail-sourcing k9` shows the supplier's reply, threaded by its `In-Reply-To`,
and the descriptor crop-warden passes to `--reply-to`.

Source: V-A17, V-A18.

### 4.3 `akoe operation`

`akoe operation --account <account> (--key <key> | <number>)` prints an
operation's recorded outcome, its finding if a person resolved it
([§FS-sending.3](FS-sending.md#3-unknown-blocks-the-thread)), and the evidence Akoe holds beside it.

**Example.** After a crash, crop-warden runs `akoe operation --account
mail-sourcing --key lead-12-outreach-1`, reads `"outcome": "unknown"` with exit
6, and records the lead as needing a person rather than sending again.

Source: V-A18; settled 2026-10-13.

### 4.4 `akoe wait`

`akoe wait --account <account> --after <cursor>` observes the account once, as
`messages` would, and exits 0 when a message from someone other than the owner
was first observed after the cursor, and 75 when none was. It prints the new
cursor and the conversations that moved; the caller keeps the cursor, so Akoe
keeps nothing per caller. It does not block. A read that does not complete is
never answered 75: it exits 4. A cursor Akoe can no longer compare, such as one
from before a mailbox's UIDVALIDITY changed, is answered 0, so the caller
looks.

**Example.** crop-warden's poll runs `akoe wait --account mail-sourcing --after
c:1841`. It exits 75 on a quiet day, and the program state ends without reading
anything. The next day it exits 0 with `{"cursor": "c:1847", "conversations":
["k9"]}`.

Source: V-A18; settled 2026-10-13.

## 5. Labels are the sender's act

A label is applied only as the sender's explicit act, with `akoe send --label`,
on the message being sent; its thread then joins the label's room
([§FS-names.3.3](FS-names.md#33-a-label-the-owner-applies-can-be-a-room)). Akoe never labels, flags or marks anything on reception. A
label that fails after the message was accepted is applied again; the message
is never sent again to carry it.

**Example.** lead 12's mail is accepted, and applying `suppliers` to it fails.
Akoe applies the label again on the sent message; it does not send lead 12's
mail a second time.

Source: V-A17.

## 6. No library crate and no MCP server

Akoe publishes no library crate and serves no MCP server. A library would
publish pre-1.0 internals and the network libraries' types as an API, and would
invite callers around the authority boundary. An MCP send tool would hand a
model the capability to send, where answers are proposals. Akoe's trait over the
networks stays internal.

**Example.** An agent's tool list offers no Akoe tool. A Rust program cannot
link Akoe; it runs `akoe send` as crop-warden's program states do.

Source: V-A18.

## 7. Commands for a person

A person's commands run at a terminal as Akoe's own user, `sudo -u akoe akoe
<command>`, behind the owner's password ([§FS-privacy.2.2](FS-privacy.md#22-akoes-own-unix-user)):

| Command | Does |
|---|---|
| `akoe pair <account>` | pairs an account ([§FS-privacy.1](FS-privacy.md#1-pairing-is-a-persons-act)) |
| `akoe retire <account> <conversation>` | retires a conversation ([§FS-record.4.1](FS-record.md#41-retirement-is-a-persons-act)) |
| `akoe purge <account> <conversation>` | deletes a retired conversation's words ([§FS-record.4.2](FS-record.md#42-nothing-is-deleted-or-cut-automatically)) |
| `akoe resolve <account> <operation> sent\|not-sent` | records a finding on an `unknown` ([§FS-sending.3](FS-sending.md#3-unknown-blocks-the-thread)) |
| `akoe ack-gap <account> <gap>` | acknowledges a gap so the source answers again ([§FS-record.3.4](FS-record.md#34-gaps)) |
| `akoe reset-identity <account>` | gives the account's conversations new ids ([§FS-names.4.3](FS-names.md#43-an-identity-reset)) |
| `akoe backup <dir>`, `akoe restore <dir>`, `akoe release-restore <account>` | [§FS-record.2.3](FS-record.md#23-backup-is-a-persons-act) |
| `akoe reload` | applies a changed configuration |

The configuration is edited with `sudoedit -u akoe`. Three commands read and
need no password: `akoe config show` reports each account's scope, senders and
what its network cannot report ([§FS-record.7](FS-record.md#7-declared-scope), [§FS-record.6.3](FS-record.md#63-what-a-network-cannot-report-is-stated)); `akoe config
sources` prints ephor's blocks ([§FS-interfaces.2.3](FS-interfaces.md#23-a-budget-of-about-ten-sources)); `akoe status` reports
whether each account vouches, and why not.

**Example.** The owner runs `sudo -u akoe akoe retire mail-me k1` once the
supplier thread is finished, and `akoe config show` to check that `mail-me`
reads "all delivered mail since 2026-10-01".

Source: V-A07, V-A16, V-A21; the owner's choice of 2026-10-13; settled 2026-10-13.
