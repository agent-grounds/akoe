# FS-sending: each authorized operation is admitted once, attempted at most once, and recorded as accepted, rejected or unknown

How Akoe sends, and what it never does when sending. This is the outbound half
of [§GRUND-akoe.1](../grund.md#1-what-it-removes). Who may ask Akoe to send is [§FS-sending.8](FS-sending.md#8-the-sender-allowlist-guards-against-mistakes) and
[§FS-privacy.4](FS-privacy.md#4-confinement-is-a-precondition-not-a-feature); the command surfaces are [§FS-interfaces](FS-interfaces.md#fs-interfaces-ephor-reads-akoe-through-the-forge-protocol-and-programs-through-a-small-command-line).

## 1. Operations and the ledger

An operation is one authorized outbound message: its account, its destination,
and its full payload. The destination is a thread's reply descriptor for a reply
([§FS-interfaces.3.2](FS-interfaces.md#32-reply)), or a caller's key for a new conversation ([§FS-sending.7](FS-sending.md#7-new-outbound-mail-takes-a-caller-key)).
Akoe freezes the payload when it admits the operation and writes the operation
to the account's ledger ([§FS-record.2.2](FS-record.md#22-record-and-cache)) before any attempt. Every later change
to the operation is written in the same transaction as the fact behind it.

**Example.** The owner posts "Thanks, confirmed." to a supplier thread. Akoe
writes operation 7, with the thread, its generation and those exact bytes, to
the `mail-me` ledger, and only then opens the SMTP session.

Source: V-A04, V-A08.

## 2. One attempt, three outcomes

Akoe makes at most one attempt per operation: one submission of the frozen
message to the network. The attempt ends in one of three outcomes:

- `accepted`: the network acknowledged that it accepted the message;
- `rejected`: the network definitively refused it, or the attempt failed before
  the message could reach the network;
- `unknown`: the attempt may have reached the network and no acknowledgement
  came.

Success is reported only on `accepted`, never on a queue append or a handoff to
a local daemon. `accepted` promises acceptance, not delivery. An operation found
in flight after a crash or kill is recorded `unknown` and is not attempted
again. A request to send the words of a `rejected` operation again is a new
operation ([§FS-sending.5.1](FS-sending.md#51-an-identical-request-returns-the-recorded-outcome)).

**Example.** Three SMTP attempts end three ways. The connection is refused
before `MAIL FROM`: `rejected`. The server answers the message data with a 550:
`rejected`. The session times out after the final `.` with no reply: `unknown`.

Source: V-A01, V-A02, V-A08.

## 3. `unknown` blocks the thread

While any operation on a thread is `unknown`, Akoe admits no operation on that
thread and refuses with a diagnosis naming the operation. Only a person resolves
an `unknown`; Akoe never resolves one by itself and never resubmits. A message
Akoe can correlate with the operation by a native id, such as the Message-ID it
set, is shown beside the operation as evidence. Correlation is not resolution,
and absence from the Sent folder does not prove that the network did not accept
the message. After a person resolves an `unknown`, sending the same words again
is a new operation.

**Example.** Operation 10, "See you Monday", is `unknown`. Every reply to that
thread is refused with `blocked: operation 10 may have been accepted`. A message
with operation 10's Message-ID later appears in Sent; Akoe shows it beside
operation 10, and the thread stays blocked until a person resolves it.

Source: V-A01, V-A08.

- **Open:** the command a person resolves an `unknown` with, the findings it
  records, and whether a finding of "accepted" advances the generation
  ([§FS-sending.4](FS-sending.md#4-the-thread-generation)).

## 4. The thread generation

Each thread has a generation, a counter that advances on:

- every message observed in the thread, whoever wrote it, including the owner's
  own from other clients ([§FS-record.6.1](FS-record.md#61-the-owners-own-messages-from-other-clients));
- every operation Akoe records as `accepted` on the thread, at the moment it
  records it, before any read shows the message, because SMTP submission alone
  files no copy in the mailbox Akoe reads;
- every revision, where the network reports edits ([§FS-record.6.2](FS-record.md#62-edits-are-revisions-deletions-are-tombstones)).

The reply descriptor Akoe puts on a thread carries the generation the thread
had when ephor read it. A draft made from that read keeps that generation.

**Example.** The supplier's mail H offers €100, and ephor drafts "I accept €100"
at generation 1. Before posting it, the owner types `ephor reply ID "Can you
deliver by Friday?"`, and Akoe records that reply accepted: the generation
becomes 2. Nothing new arrives from the supplier. When the owner presses `p` on
the €100 draft, Akoe refuses it as stale and names "Can you deliver by Friday?"
([§FS-sending.5.3](FS-sending.md#53-a-stale-never-sent-draft-is-refused)).

Source: ruling V-D04, V-A08.

- **Open:** whether Akoe's own accepted message, read back later and correlated
  by its native id, advances the generation a second time. This spec's reading:
  each message counts once.
- **Open:** whether a tombstone ([§FS-record.6.2](FS-record.md#62-edits-are-revisions-deletions-are-tombstones)) advances the generation.

## 5. Admission

Akoe tests each request in the order below. The lookup of [§FS-sending.5.1](FS-sending.md#51-an-identical-request-returns-the-recorded-outcome)
comes before every refusal.

### 5.1 An identical request returns the recorded outcome

A request with the same destination and generation, or the same caller key, and
the same payload as an admitted operation is that operation. Akoe sends
nothing and answers with its recorded outcome: `accepted` is success; `unknown`
is refused as blocked ([§FS-sending.3](FS-sending.md#3-unknown-blocks-the-thread)). An identical request for a `rejected`
operation is admitted as a new operation, subject to [§FS-sending.5.2](FS-sending.md#52-a-blocked-thread-admits-nothing) to 5.4,
because a definite failure may be tried again.

**Example.** "Thanks, confirmed." was accepted as operation 7 and the
acknowledgement was lost. The same descriptor and the same words arrive again;
Akoe answers with operation 7's acceptance and opens no SMTP session.

Source: V-A01, V-A08.

### 5.2 A blocked thread admits nothing

Any `unknown` operation on the thread refuses the request ([§FS-sending.3](FS-sending.md#3-unknown-blocks-the-thread)).

**Example.** Operation 10 on the thread is `unknown`. A different reply,
"Sorry, wrong date", is refused and names operation 10.

Source: V-A08.

### 5.3 A stale never-sent draft is refused

A request that matches no admitted operation, and whose generation is older than
the thread's current generation, is refused. The refusal names the thread's
newest message, so the person can read what changed.

**Example.** The supplier's mail H offers €100, and ephor drafts "I accept €100"
at generation 1. A refresh fetches the supplier's new mail M, "€1,000", and the
generation becomes 2. The owner runs `ephor reply ID`; Akoe refuses with
`stale: newer message from orders@supplier.example.org: "€1,000"` and sends
nothing.

Source: ruling V-D02, ruling V-D04, V-A14.

### 5.4 A used generation refuses a different payload

A request whose generation already carries an admitted operation with a
different payload is refused, naming that operation.

**Example.** Operation 12, "See you Monday", is being attempted at generation 6.
A request at generation 6 with "See you Tuesday" is refused and names operation
12.

Source: V-A08.

- **Open:** whether a `rejected` operation uses its generation. This spec's
  reading is that it does not, so a corrected draft can be posted after a
  definite failure.

### 5.5 There is no override

Akoe offers no flag and the forge request no field to send a stale draft
anyway. To send the old words, the person types them again, or drafts again.
The typed reply carries the thread's current generation and replies to the
newest message.

**Example.** After the refusal above, the owner still wants to accept €100 and
types `ephor reply ID "I accept €100"`. It is admitted at generation 2 and goes
out as a reply to M, in the same mail conversation.

Source: ruling V-D02.

## 6. The five cases

The thread below is a supplier thread at generation 3. Cases (b), (d) and (e)
rely on ephor's change E6a: ephor saves each draft's original reply target, and
a pending intent for each typed reply, before calling Akoe, and a refresh
never replaces a saved target with the latest one. Akoe's rules alone answer
two byte-identical requests identically.

| Case | What happens | ephor sends | Akoe answers | Attempts |
|---|---|---|---|---|
| (a) | "Thanks, confirmed." is accepted as operation 7 (generation becomes 4); the acknowledgement to ephor is lost; the owner refreshes and presses `p` again | the saved target, generation 3, same text | operation 7's acceptance; ephor retires the draft | 1 |
| (b) | the owner types "Thanks" at 14:02, accepted as operation 8 and confirmed; after a refresh, types it again on purpose at 14:10 | the current target, generation 4, same text | admits operation 9 and sends it | 2, as intended |
| (c) | `p` admits operation 10, which ends `unknown`; the owner refreshes and presses `p` again | the saved target, generation 3, same text | refused: blocked until a person resolves operation 10 | 1 |
| (d) | as (a), but before the second `p` the supplier writes "See you Monday" (generation 5) and the feed refreshes | the saved target, generation 3, same text | operation 7's acceptance, found before the stale test | 1 |
| (e) | the owner types "Thanks, see you Monday", accepted as operation 11; the acknowledgement is lost; the supplier writes; the owner types the same command again | the pending intent's saved target, generation 3, same text | operation 11's acceptance | 1 |

Without the saved target, case (d) would reach Akoe with generation 5's
descriptor, which matches no operation, and the words would be sent a second
time.

Source: V-A08, V-C12.

- **Open:** case (b) assumes ephor gives a new typed command the thread's
  current descriptor. If the 14:10 command reached Akoe with the generation-3
  descriptor, Akoe would answer it as case (a) and send nothing. The verdict
  does not say whether ephor refreshes the target before a new typed command.

## 7. New outbound mail takes a caller key

`akoe send` opens a new conversation and takes a key from its caller, unique
within the account. The key identifies the operation: a request with a key
already in the ledger and the same payload is answered by [§FS-sending.5.1](FS-sending.md#51-an-identical-request-returns-the-recorded-outcome), and
one with a different payload is refused. Akoe returns the new conversation's
key, through which the caller reads replies ([§FS-interfaces.4](FS-interfaces.md#4-the-command-line-for-programs)).

**Example.** crop-warden runs `akoe send --account mail-sourcing --as
crop-warden --key lead-12-outreach-1 --body-file F`, and the session times out:
`unknown`. crop-warden's program state runs again after a restart with the same
key and the same file; Akoe answers `unknown` and sends nothing.

Source: V-A08, V-A17.

- **Open:** whether `akoe send` can also reply within an existing
  conversation, and how a caller names one.

## 8. The sender allowlist guards against mistakes

Each account's configuration lists the projects allowed to send through it.
`akoe send --as <project>` is refused when the project is not listed. `--as` is
attribution, not authentication: anything that can run Akoe can name an
allowed project. What stops an agent from sending is confinement
([§FS-privacy.4](FS-privacy.md#4-confinement-is-a-precondition-not-a-feature)), not this list.

**Example.** `~/.config/akoe/mail-sourcing.toml` says `senders =
["crop-warden"]`. An agent in another project that runs `akoe send --as rhei`
by mistake is refused. An agent that passes `--as crop-warden` on purpose is
stopped only by confinement.

Source: V-A13, V-A17, V-A22.

- **Open:** whether and how the allowlist applies to `akoe-forge reply`, whose
  caller is ephor carrying the person's post.

## 9. What Akoe never does when sending

- It never makes a second attempt for an operation.
- It never attempts on a thread that has an `unknown` operation.
- It never reports success before the network's known acceptance.
- It never admits a never-sent draft older than its thread.
- It never changes an admitted operation's payload.
- It never sends again to repair something that failed after acceptance: a
  failed label is applied again, and a failed mirror into another system is
  retried there.
- It never sends because something was received: no automatic reply, reaction
  or receipt ([§FS-record.5](FS-record.md#5-receiving-has-no-side-effects)).
- It never resubmits on its own: Akoe has no retry loop.

Each of these has a test.

**Example.** crop-warden's lead 12 mail is accepted, but mirroring it into the
lead's issue fails. Lead 12 is sent, the mirror is retried, and Akoe sends no
second mail to give the mirror something to record.

Source: V-A01, V-A08, V-A16, V-A17, V-A19.

## 10. Receipts and resets

### 10.1 A reconciled receipt says the send was accepted earlier

An answer that returns an earlier recorded acceptance says that the earlier send
was accepted. It never says the message was posted or sent now, and implies no
new transmission or delivery. The forge's reply answer is an empty object in
both cases; ephor knows from its own pending intent which one it is, and words
its receipt accordingly.

**Example.** A retry after a timeout prints "reply confirmed accepted" in ephor,
while a new send prints "↩ reply posted". `akoe send` rerun with an accepted
key reports the acceptance it recorded at 14:02, not a send at 14:10.

Source: V-A27, V-A15.

- **Open:** the exact words and fields of Akoe's own structured answer for a
  reconciled acceptance.

### 10.2 An identity reset keeps the ledger

A deliberate reset of an account's conversation identity ([§FS-names.4.3](FS-names.md#43-an-identity-reset)) keeps
the operation ledger. Every operation and its recorded outcome survive, and are
still found by the descriptors and keys issued before the reset.

**Example.** After a reset gives the account's conversations new row names,
operation 41 and its recorded acceptance are still in the ledger. A second press
on a draft made before the reset is answered with that acceptance and sends
nothing.

Source: V-A27.
