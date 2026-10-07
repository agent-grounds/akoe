# FS-interfaces: ephor reads Akoe through the forge protocol, and programs through a small command line

The two ways in. ephor, the intake that [§GRUND-akoe](../grund.md#grund-akoe-no-person-carries-a-conversation-into-the-work-loop-or-an-answer-back) serves, reaches Akoe through
the forge protocol it already ships. Other programs reach it through a small
structured command line. There is nothing else.

## 1. One executable for ephor

### 1.1 `akoe-forge` takes ephor's subcommands

ephor runs one executable, `akoe-forge`, for every Akoe source. Each source's
`command` is that executable's path, and ephor appends the protocol subcommand
(`capabilities`, `messages`, `reply`) as one argument. ephor does not split a
`command` string, so the executable takes the subcommands directly.

**Example.** `{"provider": "mail-me", "command": "/usr/local/bin/akoe-forge"}`
and `{"provider": "wa-me", "command": "/usr/local/bin/akoe-forge"}` work side by
side. `"command": "akoe forge"` fails as a missing executable.

Source: V-A10, V-C13.

### 1.2 Each source has its own block

ephor hands each source's configuration block to `akoe-forge` verbatim, as the
request's `config`. The block names the Akoe account, and the scope for a split
account ([§FS-interfaces.2.2](FS-interfaces.md#22-a-split-for-ownership-or-for-a-selector-never-for-isolation)), that the call is about. Each source has a
`provider` name of its own, because ephor keys a source's results and replies by
that name: two sources both named `akoe` would overwrite each other.

**Example.** The `wa-me-direct` and `wa-me-rooms` blocks name the same account
and different scopes. `akoe-forge messages` answers each for its own scope.

Source: V-A10, V-C13.

- **Open:** the keys by which a block names its account and scope, or whether
  the provider name itself is the account's name.

### 1.3 Capabilities do not depend on configuration

ephor probes capabilities with an empty configuration, so `akoe-forge
capabilities` declares the same surface for every source: `messages` and
`replies`, the two rows ephor's DF-003-chat-is-a-forge-capability added for a
chat gateway. What one account cannot do, it refuses with a diagnosis. An
account that cannot send puts no reply descriptor on its threads, and ephor then
offers its drafts as text to copy.

**Example.** `akoe-forge capabilities` given `{"config": {}}` answers
`{"messages": true, "replies": true}`. A `reply` to a thread of a read-only
Slack source exits non-zero with `slack-ag-rooms: this account cannot send`.

Source: V-A10.

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
Sources split from one account share its connection, and fail together when it
fails.

**Example.** `slack-ag-direct` and `slack-ag-rooms` share one Slack connection.
When the socket drops, both fail. When one channel's history cannot be
reconciled, only `slack-ag-rooms` fails. A recipe with `"when": {"sources":
["slack-ag-direct"]}` answers direct conversations and not channels.

Source: V-A09.

### 2.3 A budget of about ten sources

One owner has roughly six to ten sources; ten is a budget for configuration,
not a measurement, and nobody has measured how that many perform. Akoe
generates ephor's source blocks from its own configuration, and its acceptance
includes a generated ten-source fixture.

**Example.** ephor's `status.json` holds blocks such as `mail-me`,
`wa-me-direct`, `wa-me-rooms`, `signal-me`, `slack-ag-direct`, `slack-ag-rooms`
and `discord-ag-rooms`, each generated from `~/.config/akoe`, each with its own
`provider` and the same `command`.

Source: V-A10.

- **Open:** the command that prints the generated blocks.

## 3. The forge answers

### 3.1 `messages`

`akoe-forge messages` answers the conversations in the source's declared scope
([§FS-record.7](FS-record.md#7-declared-scope)), except the retired ([§FS-record.4.1](FS-record.md#41-retirement-is-a-persons-act)). Each is whole: every
message Akoe holds for it ([§FS-record.4.2](FS-record.md#42-nothing-is-deleted-or-cut-automatically)), each with its author and whether it
is the owner's ([§FS-names.4.2](FS-names.md#42-the-owners-identities-and-mine)); its room, absent for a direct conversation
([§FS-names.3](FS-names.md#3-rooms)); its reasons ([§FS-names.3.2](FS-names.md#32-a-direct-conversation-has-no-room-and-no-reason)); and, on each thread where the
account can send, a reply descriptor that carries the thread's generation
([§FS-sending.4](FS-sending.md#4-the-thread-generation)). When the source cannot vouch for the account
([§FS-record.1.2](FS-record.md#12-the-read-is-the-observation), [§FS-record.3.2](FS-record.md#32-the-vouching-lease)), it fails rather than answers.

**Example.** A supplier thread is answered as
`{"id": "k1", "title": "Order 12", "updated_at": "2026-10-07T09:14:00Z",
"reasons": [], "threads": [{"messages": [...], "reply": {...}}]}`, where the
`reply` object is Akoe's own and opaque to ephor.

Source: V-A04, V-A05, V-A06, V-A07, V-C11, V-C15.

### 3.2 `reply`

`akoe-forge reply` takes ephor's request as it ships: the target, which is the
descriptor Akoe put on the thread, and the text. The forge request gains no
field. Akoe admits the request as [§FS-sending.5](FS-sending.md#5-admission) says and answers an empty
object only when the operation is `accepted`, now or earlier. Every other
result is a non-zero exit with a one-line diagnosis: `rejected`, `unknown`,
blocked by an `unknown`, stale with the newer message named, or a used
generation.

**Example.** `p` on a stale draft exits 1 with
`mail-me: stale: newer message from orders@supplier.example.org: "€1,000"` on
stderr, and nothing is sent.

Source: V-A08, V-A15, ruling V-D02.

## 4. The command line for programs

A program reaches Akoe by running a command that prints a structured result.
Four commands make the surface: send, read a conversation, operation status,
and wait for new mail. Each prints JSON on stdout; a refusal is a non-zero exit
with a diagnosis on stderr. A consuming project keeps its own records,
approvals and stages; Akoe carries only the transport.

**Example.** crop-warden's program state runs `akoe wait --account
mail-sourcing`, then `akoe thread <key>` for the lead it is following, and
records lead 12 as sent only once `akoe send` reports `accepted`.

Source: V-A17, V-A18, V-A19.

### 4.1 `akoe send`

`akoe send --account <account> --as <project> --key <key> [--label <label>]
--body-file <file>` opens a new conversation through the account, under the
caller's key ([§FS-sending.7](FS-sending.md#7-new-outbound-mail-takes-a-caller-key)) and the account's sender allowlist
([§FS-sending.8](FS-sending.md#8-the-sender-allowlist-guards-against-mistakes)). It prints the operation's outcome and the new conversation's
key.

**Example.** `akoe send --account mail-sourcing --as crop-warden --key
lead-12-outreach-1 --label suppliers --body-file F` prints the outcome
`accepted` and the conversation key that crop-warden stores against lead 12.

Source: V-A17.

- **Open:** how the recipients and the subject are given.

### 4.2 `akoe thread`

`akoe thread <key>` prints the conversation with that key as JSON, with the
replies that came back into it ([§FS-names.2](FS-names.md#2-threading-generic-mail)).

**Example.** Two days after the first mail to lead 12, `akoe thread <key>` shows
the supplier's reply, threaded by its `In-Reply-To`.

Source: V-A17, V-A18.

### 4.3 Operation status

A command prints an operation's recorded outcome, by its caller key or its
number.

**Example.** After a crash, crop-warden asks for `lead-12-outreach-1` and reads
`unknown`; it records the lead as needing a person rather than sending again.

Source: V-A18.

- **Open:** the command's name.

### 4.4 `akoe wait`

`akoe wait --account <account>` exits 0 when something new came in and 75 when
nothing did.

**Example.** crop-warden's poll runs `akoe wait --account mail-sourcing`. It
exits 75 on a quiet day, and the program state ends without reading anything.

Source: V-A18.

- **Open:** what "new" is measured from, and how long the command waits.

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

Three commands are a person's, run at a terminal: `akoe pair <account>` pairs an
account ([§FS-privacy.1](FS-privacy.md#1-pairing-is-a-persons-act)), `akoe retire <conversation>` retires a conversation
([§FS-record.4.1](FS-record.md#41-retirement-is-a-persons-act)), and `akoe config show` reports each account's scope
([§FS-record.7](FS-record.md#7-declared-scope)).

**Example.** The owner runs `akoe retire mail-me:k1` once the supplier thread is
finished, and `akoe config show` to check that `mail-me` reads "all delivered
mail".

Source: V-A07, V-A16, V-A21.
