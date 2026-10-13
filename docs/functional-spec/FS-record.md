# FS-record: Akoe reads what a network keeps, keeps what it does not, and fails when it cannot vouch

What Akoe holds for each account, how it shows that an answer is complete, and
what receiving may never change. This is the receiving half of [§GRUND-akoe.1](../grund.md#1-what-it-removes).
Message ids and threads are [§FS-names](FS-names.md#fs-names-a-conversations-id-never-changes-a-room-names-a-venue-and-the-owner-is-one-person-across-accounts); the operation ledger is [§FS-sending.1](FS-sending.md#1-operations-and-the-ledger).

## 1. Mail is read at refresh

### 1.1 No listener and no body store

A mail account has no listener. Each `messages` call reads the account's
declared incoming and Sent scopes ([§FS-record.7](FS-record.md#7-declared-scope)) from the server at that moment.
Akoe keeps no copy of incoming message bodies; what it keeps for mail is the
record of [§FS-record.2.2](FS-record.md#22-record-and-cache).

**Example.** The owner's mailbox `owner@example.org` holds a supplier thread.
Between refreshes Akoe holds no connection to the server and has no file
containing the supplier's words; the next refresh reads them from the server.

A complete read is complete for the declared scope, which may begin at a start
date the person chose ([§FS-record.7](FS-record.md#7-declared-scope)). The read lists every message in scope
and fetches the words only of the conversations that are not retired. What one
read costs on a mailbox of ten thousand and of a hundred thousand messages is
measured on synthetic mailboxes before any real account is read, and stated
here.

Source: V-A04; settled 2026-10-13.

### 1.2 The read is the observation

A complete read of the declared scopes, made without marking anything, is the
evidence that the account is observed now. If the read cannot finish, the source
fails with a diagnosis and ephor shows the last good rows as stale. A partial
read is never answered as complete.

Because the read covers the whole scope, rather than the unread flag or the
mail that arrived while Akoe was running, it also finds mail the owner read on
the phone while Akoe was down.

**Example.** The server drops the connection after the Inbox is read and before
Sent is. The source fails with `mail-me: read of Sent did not finish`. Answering
with the Inbox alone would have hidden the owner's reply in Sent and shown the
thread as waiting.

**Example.** Akoe's host is off for an hour. A supplier's mail arrives and the
owner reads it on the phone. The next refresh lists it, because the read does
not start from the server's next UID or depend on `\Seen`.

Source: V-A04, V-A05, V-C05; ephor's FS-001-forge-interface.6, rule 1.

## 2. The per-account record

### 2.1 One store per account

Each account has exactly one transactional SQLite store, under
`~/.local/state/akoe/<account>/` ([§FS-privacy.2.1](FS-privacy.md#21-configuration-secrets-and-record)). A message and the cursor
that covers it change in one transaction, and so do an operation and its state.

**Example.** Akoe records an incoming Signal message and advances its receive
cursor past it in one transaction. A crash between the two writes leaves
neither, never a cursor past a message that is not in the record.

Source: V-A04.

### 2.2 Record and cache

The store holds two kinds of data. The **record** cannot be rebuilt from the
network and is backed up:

- the conversation ids Akoe issued, and their aliases ([§FS-names.1](FS-names.md#1-conversation-ids));
- the operation ledger ([§FS-sending.1](FS-sending.md#1-operations-and-the-ledger));
- the messages a keeper received ([§FS-record.3](FS-record.md#3-keepers)), which are the only record that
  they arrived;
- each account's verified remote identity ([§FS-names.4.1](FS-names.md#41-the-account-namespace)).

The record also holds each retirement, with the generation its conversation had
when it was retired ([§FS-record.4.1](FS-record.md#41-retirement-is-a-persons-act)), because no network can say what a person
retired.

The **cache** can be rebuilt from the network: the index of mail headers, and the
messages of the networks read from their history ([§FS-record.3.1](FS-record.md#31-what-is-kept)). Deleting
the cache changes no answer, only the cost of the next one, except that the read
which rebuilds it also sees edits and deletions older than the network's edit
window ([§FS-record.6.3](FS-record.md#63-what-a-network-cannot-report-is-stated)).

**Example.** Mail C arrives citing only B, and Akoe issues `k1`; ephor shows
row `mail-me:k1` and a run drafts a reply on it. Later B turns up citing an
older A, and A and B join `k1`, so the draft stays on its row. Had the record
been deleted and rebuilt, the thread would come back as `mail-me:k0` with no
draft, and an `unknown` send on `k1` would be forgotten, so a second press could
send twice.

Source: V-A04, V-A21, V-C10; settled 2026-10-13.

### 2.3 Backup is a person's act

Akoe writes a consistent copy of every account's record before each change to
the store's format, and whenever a person runs `akoe backup <dir>`. A copy holds
no session: a session is a key ([§FS-privacy.2.1](FS-privacy.md#21-configuration-secrets-and-record)), and a restored account is
paired again ([§FS-names.4.1](FS-names.md#41-the-account-namespace)). Where copies go after that, and whether they
are encrypted, is the owner's choice.

A record restored with `akoe restore <dir>` may lack operations admitted after
the copy was made, so a request that matched one of them would be sent again.
A restored account therefore sends nothing until a person runs `akoe
release-restore <account>`, having checked the network for what went out since
the copy.

**Example.** The disk holding `/var/lib/akoe` fails. The owner restores
yesterday's copy and pairs `mail-me` again. `p` on a draft is refused with
`mail-me: refused: restored from a copy of 2026-10-12; check what was sent since
and run akoe release-restore mail-me`. The owner finds this morning's reply in
Sent, resolves the draft in ephor as sent, and releases the account.

Source: V-A04, V-A21; settled 2026-10-13.

## 3. Keepers

### 3.1 What is kept

A network that cannot supply a conversation later is kept: Akoe holds the
linked session and writes each received message to the record. WhatsApp and
Signal are kept. Where a network keeps history that Akoe can read, Akoe reads
that history rather than keeping a copy: mail ([§FS-record.1](FS-record.md#1-mail-is-read-at-refresh)), Slack, Discord
and Matrix are read at refresh, and a read that completes during the request is
the observation ([§FS-record.1.2](FS-record.md#12-the-read-is-the-observation)), with no session and no lease.

**Example.** A WhatsApp message from +1 555 0142 arrives while ephor is not
refreshing. Akoe's WhatsApp session receives it and writes it to the `wa-me`
record; the next `messages` call answers from the record, because WhatsApp
offers no history Akoe could ask for later.

**Example.** Slack's API cannot be reached during a refresh. The `slack-ag`
read does not complete, and its sources fail; the next refresh that completes
answers again with nothing missing, because Slack kept every message.

Each keeper network passes the checks of [§GRUND-akoe.3](../grund.md#3-when-it-shrinks) on its own test pair. A
network whose library fails them leaves scope alone; a failure of Akoe's own
keeper core, its lease, its commit before acknowledgement or its replay after a
kill, removes every keeper.

Source: V-A01, V-A02, V-A04; settled 2026-10-13.

### 3.2 The vouching lease

A keeper source answers only while it holds a lease, granted from the real
receiving session: the network's own recent answer on that same connection, when
the connection started, how far it has caught up (its catch-up frontier), and
any known gaps. A restart starts without a lease and earns it again. A running
process is never a lease, nor is a probe on another connection or a local health
endpoint, nor the age of the newest message. Without a lease the source fails as
unable to vouch, and ephor shows its rows as stale rather than as nothing new.

**Example.** signal-cli is running and answers its local `/api/v1/check`, but its
link to Signal's servers has expired. The `signal-me` source fails as unable to
vouch, and ephor shows its rows stale.

Source: V-A05; ephor's FS-001-forge-interface.6, rule 1.

### 3.3 Per-network lease contracts

Each keeper is required to establish the lease as follows; these are
contracts Akoe must meet, not properties assumed of the libraries.

| Network | The lease is |
|---|---|
| WhatsApp | a harmless exchange on the linked socket, plus a completed sync; never setting presence to "available" |
| Signal | proven upstream receive health, plus continuity of Akoe's own subscription to the signal-cli daemon |

A network read from its history needs no lease ([§FS-record.3.1](FS-record.md#31-what-is-kept)). Were a live
session ever added for one, its lease would be stated here first: for Slack's
Socket Mode, the subscribed socket plus history reconciliation after a gap; for
Discord's gateway, `HEARTBEAT_ACK` with sequence and resume.

**Example.** signal-cli's receive loop stops reporting Signal's servers while
Akoe's process keeps running. Akoe loses the `signal-me` lease, and the source
fails until the loop reports the servers again.

Source: V-A05.

### 3.4 Gaps

A restart, reconnect or resume that reveals an interval Akoe cannot recover
makes the source fail, naming the account and the interval. "The network has
no catch-up" is not proof that nothing was missed. A gap that cannot be
backfilled stays a failure.

**Example.** The WhatsApp keeper restarts after its linked device was offline
longer than the network queues messages for it. The source fails with
`wa-me: messages between 2026-03-01T08:00Z and 2026-03-03T09:00Z cannot be
recovered`, and ephor shows `wa-me` rows as stale.

A person acknowledges a gap with `sudo -u akoe akoe ack-gap <account> <gap>`,
having looked at the network for what the interval held. The acknowledgement is
recorded, `akoe config show` lists it, and the source answers again. The forge
answer cannot say that messages may be missing, so nothing in it marks the gap;
the person's acknowledgement is the only way past it.

Source: V-A05; settled 2026-10-13.

### 3.5 A kill loses no received message

A message the network handed to Akoe is not lost if Akoe is killed before
recording it. Either Akoe commits it to the record before the network may
consider it delivered to Akoe, or the adapter is shown, by killing Akoe
between receipt and commit, to receive it again after the restart.

**Example.** On the test pair, Akoe is killed while a message from +1 555 0142
is in memory and not yet committed. After the restart the message is in the
record, because the network delivered it again.

Source: V-A03 (item 4), V-A24, V-A25.

### 3.6 Downtime

After downtime, mail is recovered from the server ([§FS-record.1.2](FS-record.md#12-the-read-is-the-observation)) and keepers
from the network's own queues and history. How long each network holds messages
for an offline linked device is unverified; the measured values belong in this
section.

**Example.** Akoe is down for six hours. On restart, the WhatsApp keeper
receives what the network queued for its linked device, and Signal what
signal-cli's daemon received meanwhile; anything beyond the network's window is a
gap ([§FS-record.3.4](FS-record.md#34-gaps)).

Until a network's window is measured on its test pair, the window is the longest
downtime a test has shown fully recovered on that network. Downtime longer than
that fails as a gap, even where the network may in fact have queued everything.

Source: V-A07; settled 2026-10-13.

### 3.7 A lease has a maximum age

A keeper's lease is lost when its newest network evidence is older than the
account's maximum lease age, measured on a clock that does not jump. The default
is twice the interval of the keeper's own harmless exchange with the network,
and an account may set it. Check 1 of [§GRUND-akoe.3](../grund.md#3-when-it-shrinks) asserts that a cut
connection fails the source within it.

**Example.** `wa-me` exchanges a harmless query with WhatsApp every 60 seconds,
so its maximum lease age is 120 seconds. The network is cut at 10:00:00; a
refresh at 10:01:30 still answers, and one at 10:02:01 fails with `wa-me: cannot
vouch: no network evidence for 121s`.

Source: V-A05; settled 2026-10-13.

## 4. Retirement and quotas

### 4.1 Retirement is a person's act

A conversation leaves Akoe's answers only when a person runs `akoe retire` on
it ([§FS-interfaces.7](FS-interfaces.md#7-commands-for-a-person)). Akoe infers no retirement, from age or from the owner having had the last
word. Retiring hides the conversation from the source's answers and changes
nothing in the remote mailbox or chat. A new inbound message reactivates it.

**Example.** The owner's last WhatsApp message to a contact is "I will check
tomorrow", and then nothing. On day 90 the contact writes. The row waits, with
"I will check tomorrow" still in its history and in the `answer` dossier.

Source: V-A07.

### 4.2 Nothing is deleted or cut automatically

Akoe deletes nothing automatically and caps no conversation: a reported
conversation carries every message Akoe holds for it, because ephor's answer
has no way to say that messages were left out. Purging a retired conversation's
words is a separate, deliberate act.

**Example.** A family group with 300 messages is reported with all 300. Cut to
the last 50, it would still wait on "Are you coming Sunday?", but a drafted
answer would read a history that silently starts in the middle.

A person purges a retired conversation's words with `sudo -u akoe akoe purge
<account> <conversation>`. It refuses a conversation that is not retired, or
that has an `unknown` operation no person has resolved. It deletes the messages
Akoe holds, and keeps the conversation's ids, its aliases and its operations;
each operation keeps a hash of its payload in place of the words, so an
identical request is still recognised ([§FS-sending.5.1](FS-sending.md#51-an-identical-request-returns-the-recorded-outcome)). A purged conversation
that a new message reactivates starts from that message.

Source: V-A07, V-C11; settled 2026-10-13.

### 4.3 A full quota fails the source

An account may declare a byte quota for its record. When the quota is full, the
source fails as a whole, naming the account and the bytes. Akoe never drops
messages to stay under it.

**Example.** `wa-me` declares a quota of 2 GiB. When it is reached, the source
fails with `wa-me: record quota 2147483648 bytes reached`, and every message
already recorded stays.

Source: V-A07.

## 5. Receiving has no side effects

Reading through Akoe leaves no trace a contact, or the owner's other clients,
can see: no read receipts, typing indicators, presence, automatic reactions, or
`\Seen` flags, and no label or flag written on reception. Each invariant holds
below the library boundary too, and each has a test. A transport-level delivery
acknowledgement that the protocol requires is kept distinct from a human read
receipt.

**Example.** A supplier's mail is read through ephor. It stays unread in the
owner's desktop client, and no read-receipt mail goes to the supplier.

Source: V-A16, V-A17.

## 6. Own messages, edits and deletions

### 6.1 The owner's own messages from other clients

A message the owner sends from any other client is captured as the owner's: for
mail from the Sent folder (or All Mail), for WhatsApp and Signal from
own-message and device-sync events. A message is the owner's when any of the
owner's identities wrote it ([§FS-names.4.2](FS-names.md#42-the-owners-identities-and-mine)).

**Example.** The owner answers a WhatsApp chat on the phone. The next refresh
reports that reply with `mine: true`, and ephor stops showing the chat as
waiting.

Source: V-A02, V-A16, V-C05.

### 6.2 Edits are revisions, deletions are tombstones

Where a network reports an edit, Akoe records a new revision of the message;
where it reports a deletion, a tombstone. Where a network cannot report them,
Akoe says so rather than implying that nothing was edited
([§FS-record.6.3](FS-record.md#63-what-a-network-cannot-report-is-stated)). An edit or a deletion advances the thread's generation
([§FS-sending.4](FS-sending.md#4-the-thread-generation)).

The forge answer reports a revised message in place, with the words of its
newest revision and its first time, author and `mine` unchanged. A deleted
message stays in place with the words `(message deleted)`. Each message also
carries Akoe's own keys `revision`, a number from 1, and `deleted`, a boolean;
ephor ignores keys it does not know, and `akoe thread` shows them.

**Example.** A contact on Signal edits "€100" to "€1,000". Akoe records
revision 2 of that message, and a draft made against revision 1 is refused as
stale ([§FS-sending.5.3](FS-sending.md#53-a-stale-never-sent-draft-is-refused)).

Source: V-A16, ruling V-D04; settled 2026-10-13.

### 6.3 What a network cannot report is stated

| Network | Edits | Deletions |
|---|---|---|
| mail | none: a sent mail is never edited | not reported: a mail deleted from the server is no longer read, and leaves the answer unmarked |
| WhatsApp, Signal | reported, as the session receives them | reported, as the session receives them |
| Slack, Discord | seen within the account's edit window | seen within the account's edit window |
| Matrix | reported, as the sync delivers them | reported, as redactions |

An edit window is how far back each read looks again for changes, set per
account. `akoe config show` states each account's line of this table in words.
ephor recognises a thread by its first message, so a mail conversation whose
first message is deleted from the server reaches ephor as a new thread.

**Example.** `akoe config show` reports `discord-ag: edits and deletions seen
within 7 days`. A message edited three weeks after it was sent keeps its old
words in the answer, as the owner was told.

Source: V-A16; settled 2026-10-13.

## 7. Declared scope

Which messages an account answers for is chosen when the account is activated,
from choices that each say what they leave out. No filter applies that nobody
chose. `akoe config show` reports the chosen scope. A chosen boundary is not a
claim that the mail it leaves out was irrelevant, and ephor grows no newsletter
policy.

| Network | Choices |
|---|---|
| mail | the folders or labels read; "all delivered mail", or "To or Cc without a List-Id", which leaves out Bcc-only mail and mailing lists |
| WhatsApp, Signal | direct chats, groups, or both; or a named list of either |
| Slack, Discord | every conversation the token can see, or a named list of them |
| Matrix | every room the account has joined, or a named list of them |

Every network's scope may also carry a start date. Nothing sent before it is
answered, which bounds the first read of a large mailbox; the date is shown and
chosen like any other boundary. A split account declares each of its scopes
under a name ([§FS-interfaces.1.2](FS-interfaces.md#12-each-source-has-its-own-block)).

**Example.** At activation the owner is offered "To/Cc and no List-Id", told
that it leaves out Bcc-only mail and mailing lists, and picks "all delivered
mail" instead. A supplier invoice sent through a list then appears, and
`akoe config show` reports the scope as "all delivered mail".

Source: V-A16; settled 2026-10-13.
