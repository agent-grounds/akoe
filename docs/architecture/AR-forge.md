# AR-forge: akoe-forge maps ephor's forge protocol onto Akoe's sockets and adds nothing of its own

How `akoe-forge` meets ephor's out-of-process forge contract, as ephor's
FS-001-forge-interface states it at the ephor commit Akoe pins
([§AR-testing.6](AR-testing.md#6-the-ephor-contract)). The behavior it must show is [§FS-interfaces.1](../functional-spec/FS-interfaces.md#1-one-executable-for-ephor) to
[§FS-interfaces.3](../functional-spec/FS-interfaces.md#3-the-forge-answers); this is how. The crate is `akoe-forge`; it depends on
`akoe-rpc` and `akoe-model` and on nothing that touches a store or a network.

## 1. Invocation

ephor runs `akoe-forge <subcommand>` with one JSON request on stdin and reads
one JSON value from stdout. The request's `config` is the source's whole
status.json block, verbatim; `akoe-forge` reads its `account` and `scope` keys
([§FS-interfaces.1.2](../functional-spec/FS-interfaces.md#12-each-source-has-its-own-block)) and never parses `provider`, which is ephor's name for
the rows. It sends the request to `akoed` and prints the answer.

| Subcommand | Socket | Answer |
|---|---|---|
| `capabilities` | none | `{"messages": true, "replies": true, "reply_reconciliation": true}`, printed without contacting `akoed` |
| `messages` | `read.sock` | the conversations ([§AR-forge.2](AR-forge.md#2-messages)) |
| `reply` | `send.sock` | the outcome ([§AR-forge.3](AR-forge.md#3-reply)) |
| any other | none | exit 64, `akoe-forge: unsupported subcommand <name>` |

`capabilities` stays offline because ephor probes it, with an empty
configuration and a fixed 15-second budget, before every fetch, every reply and
every dry run ([§FS-interfaces.1.3](../functional-spec/FS-interfaces.md#13-capabilities-do-not-depend-on-configuration)).

A failure is a non-zero exit and one line on stderr that starts with the source
name and a word ephor's line picker looks for: `failed`, `refused`, `cannot`,
`unable to`. When `akoed` itself cannot be reached, the line says `connection
refused`, so ephor marks the source unreachable rather than broken.

**Example.** `akoed` is stopped for an upgrade. A refresh's `akoe-forge messages`
for `mail-me` exits 69 with `mail-me: akoed connection refused at
/run/akoe/read.sock`; ephor shows `mail-me` unreachable and its last rows stale.

## 2. `messages`

`akoe-forge` prints `akoed`'s conversations as ephor's JSON:

| ephor field | Akoe's value |
|---|---|
| `id` | the conversation's reported id ([§FS-names.1](../functional-spec/FS-names.md#1-conversation-ids)) |
| `title` | mail: the first message's subject; a chat: the contact's or the room's name |
| `updated_at` | the time of the newest message, revision or tombstone |
| `room` | the room ([§FS-names.3](../functional-spec/FS-names.md#3-rooms)); absent for a direct conversation |
| `reasons` | [§FS-names.3.2](../functional-spec/FS-names.md#32-a-direct-conversation-has-no-room-and-no-reason)'s words |
| `threads` | exactly one thread: every message in order, and the reply descriptor where the account can send |
| message `author` | the author's address in [§FS-names.4.4](../functional-spec/FS-names.md#44-an-author-is-reported-by-address)'s form |
| message `mine` | [§FS-names.4.2](../functional-spec/FS-names.md#42-the-owners-identities-and-mine) |
| message `when` | the time the message was first sent, never an edit's time |
| message `text` | the newest revision's words; `(message deleted)` for a tombstone |
| message `attachments` | name, media type and size of each, never the contents |
| message `revision`, `deleted`, `author_name` | Akoe's own keys, which ephor ignores and `akoe thread` shows |

ephor recognises a message by its author, `when` and text, and a thread by its
first message ([§FS-record.6.2](../functional-spec/FS-record.md#62-edits-are-revisions-deletions-are-tombstones)). So `akoe-forge` keeps the order of a
conversation's messages, never drops its first message, and keeps `when` unique
for each author within it: two messages from one author at the same instant are
told apart by adding a millisecond to the later one in native order, the same on
every refresh.

**Example.** A Signal contact edits "€100" to "€1,000". The next answer reports
the message with the same author and `when` and the words "€1,000", with
`"revision": 2`. ephor sees an edit of a message its draft saw and refuses the
draft as stale before calling Akoe.

## 3. `reply`

`akoe-forge` sends the block's account and the request's `target` and `text` to
`akoed`, which refuses a target whose account is not the block's, admits the
rest as [§AR-ledger.2](AR-ledger.md#2-admission-is-one-transaction-in-the-accounts-actor) says, and maps the result:

| Akoe's result | `akoe-forge` prints | ephor then |
|---|---|---|
| accepted now, or found accepted | `{"status": "accepted"}`, exit 0 | confirms the saved send and retires the draft |
| `unknown` now, or found `unknown` | `{"status": "unknown", "note": "<account>: operation <n> may have reached the network; resolve it with akoe resolve"}`, exit 0 | holds the saved send for the person |
| `rejected`, `blocked`, `stale`, used generation, replies off | non-zero exit, one diagnosis line ([§FS-interfaces.3.2](../functional-spec/FS-interfaces.md#32-reply)) | keeps the saved send pending ([§FS-privacy.5.5](../functional-spec/FS-privacy.md#55-a-definite-refusal-releases-the-saved-send)) |

The note on `unknown` and every diagnosis name the operation's number, so the
person can find it in `akoe operation`.

**Example.** `p` on a draft: the SMTP session times out after the final `.`.
`akoed` records operation 10 `unknown`; `akoe-forge` prints `{"status":
"unknown", "note": "mail-me: operation 10 may have reached the network; resolve
it with akoe resolve"}`; ephor holds the send and shows the note.

## 4. The reply descriptor

The descriptor Akoe puts on a thread is the JSON object
`{"akoe": 1, "account": "<account UUID>", "conversation": <internal key>,
"generation": <n>}`. It names no network id and has no key `provider`, which
ephor reserves. It changes exactly when the conversation's generation does
([§AR-ledger.5](AR-ledger.md#5-the-generation)), because ephor treats a changed descriptor as a moved thread
and refuses drafts made before it.

`akoed` refuses a descriptor whose account UUID is not the account's current
one, which is what a pairing to a different identity leaves behind
([§FS-names.4.1](../functional-spec/FS-names.md#41-the-account-namespace)), and one whose `akoe` version it does not know.

**Example.** Conversation 17 at generation 3 carries `{"akoe": 1, "account":
"0b6e…", "conversation": 17, "generation": 3}` on every refresh until a message
arrives; then it carries generation 4, and ephor stales the drafts made at 3.

## 5. Source blocks

`akoe config sources` prints the status.json block for every source in Akoe's
configuration, in the order the configuration lists them:
`{"provider": "<name>", "command": "<path of akoe-forge>", "account":
"<account>", "scope": "<scope>", "timeout_seconds": <n>}`, with `scope` left out
for an account that is not split ([§FS-interfaces.2.3](../functional-spec/FS-interfaces.md#23-a-budget-of-about-ten-sources)). The output depends on the configuration alone, so
printing it again after an unrelated change gives the same bytes: ephor refuses
to recover a saved send when its source's block has changed.

**Example.** `akoe config sources` prints seven blocks, from `mail-me` to
`discord-ag-rooms`. The owner adds an eighth account and prints again; the
first seven are byte-identical, and saved sends on them stay recoverable.
