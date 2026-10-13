# AR-store: one SQLite file per account, its tables split into record and cache

How [§FS-record.2](../functional-spec/FS-record.md#2-the-per-account-record) is laid out. The crate is `akoe-store`; it owns the schema,
its migrations and every query, and nothing else in Akoe writes SQL. The
account's actor holds the only connection ([§AR-overview.3](AR-overview.md#3-account-actors)).

## 1. The file

Each account's store is `~/.local/state/akoe/<account>/store.sqlite` under
Akoe's home, in WAL mode with `synchronous = FULL`, so a committed transaction
survives a power cut. The schema version is in `PRAGMA user_version`; `akoed`
migrates forward on start and refuses a store newer than itself. A migration
never drops a record table's rows.

**Example.** `akoed` 0.3 starts on a store at version 2 and migrates it to 3 in
one transaction before recovery runs. `akoed` 0.2, started on that store
afterwards, refuses with `mail-me: store version 3 is newer than this akoe`.

## 2. The record tables

The record cannot be rebuilt from the network ([§FS-record.2.2](../functional-spec/FS-record.md#22-record-and-cache)):

| Table | One row per | Holds |
|---|---|---|
| `account` | store | the internal UUID, the verified remote identity, the identity epoch ([§FS-names.4](../functional-spec/FS-names.md#4-accounts-and-identity)) |
| `conversation` | conversation | an internal integer key that never changes, the reported id for the current identity epoch, the room, the generation, retirement (who, when, and the generation it was retired at) |
| `alias` | native form | a native id (a `Message-ID`, a JID, a LID, a Gmail thread id) and the conversation it belongs to |
| `merge` | late-found join | two conversation keys that are one thread ([§FS-names.1.4](../functional-spec/FS-names.md#14-merged-threads-keep-both-ids)) |
| `observed` | message seen | the conversation, the native id, the author, `mine`, the sent time, the observation sequence; for a kept network also the words ([§AR-store.4](AR-store.md#4-words-are-stored-only-where-the-network-keeps-none)) |
| `revision` | edit | the message, the revision number, the words where they are stored, the time |
| `tombstone` | deletion | the message, the time |
| `operation`, `follow_up` | operation | [§AR-store.3](AR-store.md#3-the-ledger-tables) |
| `gap` | unrecoverable interval | the interval, when found, and who acknowledged it ([§FS-record.3.4](../functional-spec/FS-record.md#34-gaps)) |
| `cursor` | adapter cursor | a keeper's committed frontier, Matrix's sync token |

The reported id is derived from the internal key and the epoch, so an identity
reset ([§FS-names.4.3](../functional-spec/FS-names.md#43-an-identity-reset)) changes reported ids by moving the epoch and leaves every
internal key, alias and operation as it was.

**Example.** Conversation 17 is reported as `k17` in epoch 1. After a reset it is
reported as `k17b` in epoch 2; operation 41, which names conversation 17, is
still found by a descriptor issued in epoch 1, because the descriptor carries
the key 17 ([§AR-forge.4](AR-forge.md#4-the-reply-descriptor)).

## 3. The ledger tables

`operation` holds one row per operation with the fields of [§AR-ledger.1](AR-ledger.md#1-an-operations-states):
number, kind, state and its timestamps, the conversation key and generation or
the caller key, the request identity and its hash, the frozen bytes, the
assigned native id, the network's id, the caller's uid and project, the
diagnosis, and the resolution. A unique index on the caller key and an index on
(conversation, generation) serve the lookup of [§AR-ledger.2](AR-ledger.md#2-admission-is-one-transaction-in-the-accounts-actor). `follow_up` holds
each label or Sent filing owed by an accepted operation and its state.

The ledger is record. An identity reset, a retirement and a purge leave it
untouched ([§FS-sending.10.2](../functional-spec/FS-sending.md#102-an-identity-reset-keeps-the-ledger)).

## 4. Words are stored only where the network keeps none

Where the words live depends on the network ([§FS-record.3.1](../functional-spec/FS-record.md#31-what-is-kept)):

| Network | Words in the store | Class |
|---|---|---|
| Mail | none; read from the server at each refresh | the `observed` rows are record, the mail index is cache |
| WhatsApp, Signal | every received message and revision | record |
| Slack, Discord, Matrix | every message the session or a history read delivered | cache |

The cache tables are `mail_index` (folder, UIDVALIDITY, UID, `Message-ID`,
references, flags) and `chat_cache` (a session network's messages). Deleting
every cache table changes no answer, only the cost of the next one; a test
deletes them and compares ([§AR-testing.4](AR-testing.md#4-end-to-end-tests-prove-the-functional-spec)).

**Example.** The `slack-ag` store is copied to a new host without its
`chat_cache`. The first refresh reads Slack's history for each declared
conversation, refills the cache, and answers the same rows with the same ids,
because the ids, aliases and generations are in the record.

## 5. Quota, backup and purge

- **Quota.** Before each commit that grows the record, the actor compares the
  file's size to the account's declared quota. A full quota fails the source
  ([§FS-record.4.3](../functional-spec/FS-record.md#43-a-full-quota-fails-the-source)), the commit that would have passed it is not made, and a
  keeper's session stops as if Akoe were down: what the network queues
  meanwhile arrives once the quota is raised, and what it does not queue is a
  gap ([§FS-record.3.4](../functional-spec/FS-record.md#34-gaps)).
- **Backup.** `sudo -u akoe akoe backup <dir>` writes a consistent copy of every
  store with `VACUUM INTO`, while `akoed` runs; `akoed` also writes one beside
  the store before each migration. Sessions are not in a backup; pairing again
  restores them ([§FS-record.2.3](../functional-spec/FS-record.md#23-backup-is-a-persons-act)).
- **Restore.** `akoe restore <dir>` puts a backup's stores in place with their
  accounts stopped, and marks each restored account so that it admits no
  operation until a person runs `akoe release-restore <account>`: operations
  admitted after the backup was taken are not in it, and a second press on one
  of them would send again.
- **Purge.** `akoe purge` refuses unless the conversation is retired and has no
  `unknown` operation without a finding. It deletes the words from `observed`
  and `revision`, and from each of its operations' frozen bytes, and keeps the
  rows, the ids, and each operation's identity hash, so an identical request is
  still found ([§FS-record.4.2](../functional-spec/FS-record.md#42-nothing-is-deleted-or-cut-automatically)).

**Example.** `wa-me` declares a 2 GiB quota and reaches it while recording a
message from +1 555 0142. The commit is not made, the WhatsApp session stops,
and `wa-me` fails with its quota diagnosis. The owner raises the quota and
reloads; the session reconnects and receives what WhatsApp queued for it,
including that message.
