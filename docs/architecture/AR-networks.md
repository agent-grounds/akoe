# AR-networks: an adapter either reads the network's history or keeps a live session, and Akoe's own trait is all the actor sees

How each network is reached. Every adapter is a crate `akoe-net-<network>`
built on the library [§DA-network-libraries](../decisions/architectural/DA-network-libraries.md#da-network-libraries-akoe-builds-on-the-libraries-zeroclaw-wraps-not-on-zeroclaw) chose, behind the traits of
`akoe-net`. No library type crosses that boundary ([§FS-interfaces.6](../functional-spec/FS-interfaces.md#6-no-library-crate-and-no-mcp-server)).

## 1. Readers and keepers

Where the network keeps history Akoe can read, the adapter reads it; where it
does not, the adapter keeps the session ([§GRUND-akoe](../grund.md#grund-akoe-no-person-carries-a-conversation-into-the-work-loop-or-an-answer-back), [§FS-record.3.1](../functional-spec/FS-record.md#31-what-is-kept)):

| Shape | Networks | What vouches | Words in the store |
|---|---|---|---|
| **reader** | mail, Slack, Discord, Matrix | a read that completes during the request ([§FS-record.1.2](../functional-spec/FS-record.md#12-the-read-is-the-observation)) | mail: none; the others: cache |
| **keeper** | WhatsApp, Signal | a lease from the live receiving session ([§FS-record.3.2](../functional-spec/FS-record.md#32-the-vouching-lease)) | record |

A reader holds no connection between requests and needs no lease: the
completed read is the evidence. Only a keeper runs between requests.

**Example.** Slack's API is unreachable during a refresh. The `slack-ag` read
does not complete and both `slack-ag-direct` and `slack-ag-rooms` fail; the next
refresh that completes its read answers again, with nothing to backfill and no
gap, because Slack kept every message.

## 2. The adapter traits

`akoe-net` defines, in Rust, roughly:

```rust
trait Adapter {
    fn facts(&self) -> NetworkFacts;            // can_send, reports_edits, reports_deletions, …
}
trait Reader: Adapter {                         // mail, Slack, Discord, Matrix
    async fn observe(&mut self, scope: &Scope, cache: &mut dyn ReadCache)
        -> Result<Observation, ReadFailure>;    // complete, or an error naming what did not finish
}
trait Keeper: Adapter {                         // WhatsApp, Signal
    async fn run(&mut self, sink: EventSink) -> SessionEnd;   // until the connection ends
}
trait Sender: Adapter {
    fn render(&self, op: &Admitted) -> Frozen;  // the exact bytes, and the native id assigned
    async fn attempt(&mut self, frozen: &Frozen) -> Attempt;  // Accepted{network_id} | Rejected{why} | Unknown{why}
    async fn follow_up(&mut self, f: &FollowUp) -> Result<(), Transient>;  // a label, the Sent filing
}
```

`EventSink::commit(event)` returns only after the actor has committed the
event to the store. A keeper acknowledges a message to its network only after
`commit` returns, or else it passes the kill-and-replay proof
([§FS-record.3.5](../functional-spec/FS-record.md#35-a-kill-loses-no-received-message), [§AR-testing.5](AR-testing.md#5-kill-points)). The events are `Received`, `OwnSent` (the
owner's message from another client), `Edited`, `Deleted`, `Lease` (evidence for
the lease), `Frontier` (the committed catch-up position) and `Gap`. An
`Observation` carries the same facts for a reader, applied in one transaction.

`attempt` makes one submission and never retries: library-level retries,
reconnect-and-resend and connection pools that replay a request are turned off,
and the conformance kit asserts it for each adapter ([§AR-testing.3](AR-testing.md#3-adapter-conformance)). An
adapter classifies by where the failure fell: before any byte of the message
could have reached the network is `rejected`; after that point and without an
acknowledgement is `unknown`.

**Example.** The mail `Sender` opens the SMTP session, and the server answers
`RCPT TO` with 550: `rejected`, because `DATA` was never sent. The connection
resets after the final `.` and before the server's reply: `unknown`.

## 3. The lease

The actor keeps one lease per keeper account, built only from `Lease` events of
the session that is receiving ([§FS-record.3.2](../functional-spec/FS-record.md#32-the-vouching-lease)):

- **granted** when the session has connected, caught up to a committed
  frontier, and produced fresh network evidence;
- **renewed** by each new piece of network evidence on that same connection;
- **lost** when the newest evidence is older than the account's maximum lease
  age on a monotonic clock ([§FS-record.3.7](../functional-spec/FS-record.md#37-a-lease-has-a-maximum-age)), when the session ends, or when a
  `Gap` is found.

A `messages` request on a keeper account answers from the store only while the
lease is held and no gap is unacknowledged; otherwise it fails, naming why.

**Example.** `wa-me` exchanges a harmless query with WhatsApp every 60 seconds,
so its maximum lease age is 120 seconds. When the network is cut, no evidence
arrives; at 120 seconds the lease is lost, and the next refresh fails `wa-me:
cannot vouch: no network evidence for 121s`, which is [§GRUND-akoe.3](../grund.md#3-when-it-shrinks)'s
check 1.

## 4. Mail

The mail reader keeps no connection between requests. Each `observe`:

1. connects with TLS (a server that offers none is refused) and authenticates
   with the account's OAuth grant or app password ([§FS-privacy.1](../functional-spec/FS-privacy.md#1-pairing-is-a-persons-act));
2. for each folder of the declared scope ([§FS-record.7](../functional-spec/FS-record.md#7-declared-scope)), opens it with
   `EXAMINE`, never `SELECT`, and lists every UID since the scope's start date;
3. fetches headers with `BODY.PEEK` for UIDs the mail index does not hold, or
   holds under an older UIDVALIDITY;
4. threads every message by its headers ([§FS-names.2](../functional-spec/FS-names.md#2-threading-generic-mail)) and finds the
   conversations in the source's scope that are not retired;
5. fetches the bodies of those conversations' messages with `BODY.PEEK`, and
   holds them in memory for this answer only.

Any step that does not finish fails the whole observation
([§FS-record.1.2](../functional-spec/FS-record.md#12-the-read-is-the-observation)). The reader never issues `STORE`, `APPEND`, `EXPUNGE` or
`SELECT` while observing; the Sent filing and labels are follow-ups of an
accepted operation ([§AR-ledger.3](AR-ledger.md#3-the-attempt-protocol)), on their own connection.

The cost of step 5 grows with the conversations not retired, and nothing in Akoe
caps it ([§FS-record.4.2](../functional-spec/FS-record.md#42-nothing-is-deleted-or-cut-automatically)); the scope's start date and the person's
retirements bound it. A read that cannot finish within the source's timeout
fails like any incomplete read, and its cost is measured ([§AR-testing.7](AR-testing.md#7-measurements)).

**Example.** The owner reads a supplier's mail on the phone while `akoed` is
stopped. The next `observe` lists the folder's UIDs from the scope's start date,
not from the last UID it saw and not by `UNSEEN`, finds the message, and fetches
it with `BODY.PEEK`; it stays read on the phone, and nothing is flagged.

## 5. Slack, Discord and Matrix

These readers keep a cursor and a cache, so a read fetches what is new instead
of everything since the scope's start:

| Network | A read is | Edits and deletions |
|---|---|---|
| Slack | `conversations.history` and `conversations.replies` for each conversation in scope, from the cursor, plus a re-read of the account's edit window | found by comparing the edit window with the cache: changed words are a revision, a missing message a tombstone |
| Discord | each channel's messages after the last id in the cache, plus a re-read of the edit window | the same comparison |
| Matrix | one `/sync` from the persisted token, until it has caught up | edit relations and redactions, as the sync delivers them |

The cursor and the sync token are committed with what they cover, in one
transaction. Losing the cache costs one full read from the scope's start, and
changes no id ([§AR-store.4](AR-store.md#4-words-are-stored-only-where-the-network-keeps-none)). What a network cannot report is one of the
adapter's `NetworkFacts`, which `akoe config show` states
([§FS-record.6.3](../functional-spec/FS-record.md#63-what-a-network-cannot-report-is-stated)).

**Example.** A Discord message from three weeks ago is edited; `discord-ag`'s
edit window is seven days. The read does not see the edit, and the message keeps
its old words. `akoe config show` already says that Discord edits are seen only
within seven days.

## 6. The keepers

| Network | Library | Lease evidence | Own messages | Send acceptance |
|---|---|---|---|---|
| WhatsApp | `whatsapp-rust` | a harmless query answered on the linked socket, after the offline sync completes; presence is never set to available | own-message and device-sync events | the server's ack for the message id |
| Signal | `signal-cli` JSON-RPC, one child per account | the daemon's receive loop reporting the server, with Akoe's own subscription continuous | sync-sent events | `send` results with `SUCCESS` for each recipient |

Each keeper turns off everything that a contact or the owner's other devices
would see on reception: read receipts, typing, presence, automatic reactions
([§FS-record.5](../functional-spec/FS-record.md#5-receiving-has-no-side-effects)). A delivery acknowledgement the protocol requires is sent, after
commit.

**Example.** A message from +1 555 0142 arrives on `wa-me`. The adapter commits
it through the sink, then lets the library send the protocol's delivery ack. It
sends no read receipt, and the contact's phone shows two grey ticks, not blue.
