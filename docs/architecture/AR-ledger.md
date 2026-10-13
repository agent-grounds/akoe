# AR-ledger: admission is one transaction, an attempt is bracketed by two, and recovery never resubmits

How the account actor carries out [§FS-sending](../functional-spec/FS-sending.md#fs-sending-each-authorized-operation-is-admitted-once-attempted-at-most-once-and-recorded-as-accepted-rejected-or-unknown). The tables are [§AR-store.3](AR-store.md#3-the-ledger-tables); the
crate is `akoe-ledger`, which holds the rules and calls the store and the
adapter through traits, so it is tested without a network or a disk
([§AR-testing.2](AR-testing.md#2-unit-tests-hold-the-rules)).

## 1. An operation's states

```text
            admit                 mark                 adapter returns
 request ─────────▶ admitted ─────────────▶ attempting ─────────────▶ accepted
                       │                        │        ├──────────▶ rejected
                       │ crash before mark      │        └──────────▶ unknown
                       ▼                        │ crash                  │ a person's akoe resolve
                    rejected                    ▼                        ▼
             ("not attempted")               unknown          unknown, with a finding:
                                                               sent or not-sent
```

`accepted`, `rejected` and `unknown` are the outcomes, and each is final: Akoe
never changes an outcome. A person's `akoe resolve` adds a finding to an
`unknown` operation, `sent` or `not-sent`, with the evidence it was made on; the
outcome stays `unknown`, because Akoe still does not know
([§FS-sending.3](../functional-spec/FS-sending.md#3-unknown-blocks-the-thread)). An `unknown` operation with a finding no longer blocks its
conversation.

An operation holds: its number (per account, increasing), its kind (`reply` or
`send`), its destination (conversation key and generation, or caller key), the
request's identity (the trimmed text for a reply, the whole command for a send),
the frozen bytes the adapter will submit, the native id Akoe assigned before
submission where the network lets it (a mail `Message-ID`), the caller's uid and
`--as` project, and a timestamp for every state.

**Example.** Operation 7 on `mail-me`: kind `reply`, conversation 17 at
generation 3, identity "Thanks, confirmed.", frozen bytes the full RFC 5322
message with `Message-ID: <akoe.7.3f9c@example.org>`, uid 1000, as `ephor`.

## 2. Admission is one transaction in the account's actor

The actor admits a request in one store transaction, testing in the order of
[§FS-sending.5](../functional-spec/FS-sending.md#5-admission):

1. **Lookup.** Find an operation with the same destination and identity (for a
   reply, conversation key, generation and text; for a send, the caller key).
   - Found `accepted`: answer it, reconciled; send nothing.
   - Found `unknown` with the finding `sent`: answer it as accepted, saying a
     person found it sent; send nothing.
   - Found `unknown` with no finding: answer `unknown`, naming it.
   - Found `admitted` or `attempting`: wait for that attempt's outcome and
     answer it; the request joins the attempt in flight and starts none.
   - Found `rejected`, or `unknown` with the finding `not-sent`: go on, as a new
     operation.
   - A caller key found with a different identity: refuse, naming it
     ([§FS-sending.7](../functional-spec/FS-sending.md#7-new-outbound-mail-takes-a-caller-key)).
2. **Blocked.** Any `unknown` operation without a finding on the conversation:
   refuse `blocked`, naming it.
3. **Stale.** The request's generation is older than the conversation's:
   refuse `stale`, naming the newest message.
4. **Used generation.** An operation at this generation with another identity,
   unless it is `rejected` or found `not-sent`: refuse, naming it
   ([§FS-sending.5.4](../functional-spec/FS-sending.md#54-a-used-generation-refuses-a-different-payload)).
5. **Allowed.** For `send`, the `--as` project is in the account's `senders`;
   for a reply from ephor, the account has `replies` on ([§FS-sending.8](../functional-spec/FS-sending.md#8-the-sender-allowlist-guards-against-mistakes)).
6. **Admit.** Render the frozen bytes, assign the native id, insert the
   operation as `admitted`.

A refusal writes nothing. Because the actor handles one request at a time, two
identical requests arriving together are admitted once and answered twice.

**Example.** Case (d) of [§FS-sending.6](../functional-spec/FS-sending.md#6-the-five-cases): the saved generation-3 request arrives
after the conversation reached generation 5. Step 1 finds operation 7 accepted
and answers it; step 3, which would have refused it as stale, never runs.

## 3. The attempt protocol

An attempt is bracketed by two transactions:

1. **Mark.** Set the operation `attempting` and commit. Nothing has been sent.
2. **Submit.** Hand the frozen bytes to the adapter's `attempt`
   ([§AR-networks.2](AR-networks.md#2-the-adapter-traits)), once. The adapter performs no retry of its own and
   classifies the result as `accepted`, `rejected` or `unknown`.
3. **Record.** In one transaction, write the outcome, the network's own id if it
   returned one, and on `accepted` advance the conversation's generation
   ([§AR-ledger.5](AR-ledger.md#5-the-generation)).

The actor serves other requests on the same account while an attempt is
in flight, but admits none on the attempting operation's conversation: step 2 of
admission treats `attempting` as blocking, like `unknown`.

Work that follows acceptance and touches the network again, such as applying a
label ([§FS-interfaces.5](../functional-spec/FS-interfaces.md#5-labels-are-the-senders-act)) or filing the sent copy ([§FS-sending.11](../functional-spec/FS-sending.md#11-a-mail-reply-is-addressed-and-filed-as-a-mail-client-would)), is a
follow-up of the operation, recorded beside it and repeated until it succeeds.
A follow-up never submits the message again.

**Example.** `akoe send --label suppliers` is accepted as operation 41; the
`STORE +X-GM-LABELS` that labels it fails. Operation 41 stays `accepted`, its
label follow-up stays `pending`, and the actor tries the label again on its next
pass. No second SMTP session opens.

## 4. Recovery after a crash

Before `akoed` opens its sockets, each account's recovery runs in one
transaction:

- `attempting` becomes `unknown`, with "Akoe stopped during the attempt": the
  bytes may have reached the network ([§FS-sending.2](../functional-spec/FS-sending.md#2-one-attempt-three-outcomes)).
- `admitted` becomes `rejected`, with "not attempted: Akoe stopped before the
  attempt": the mark was never committed, so nothing was submitted.
- A pending follow-up of an accepted operation stays pending.

**Example.** `akoed` is killed between the mark and the SMTP `DATA`. Recovery
cannot tell that nothing was sent, because the mark is the last durable fact;
operation 12 becomes `unknown`, and its thread is blocked until a person
resolves it.

## 5. The generation

Each conversation has a generation in the store. It advances, in the same
transaction as the fact behind it:

- once per message the first time Akoe observes it, by its native id; Akoe's own
  accepted message, read back later with the native id Akoe assigned, does not
  advance it again;
- once per operation recorded `accepted`, at that moment;
- once per finding of `sent`, unless a message Akoe correlated with the
  operation already advanced it;
- once per revision and once per tombstone ([§FS-record.6.2](../functional-spec/FS-record.md#62-edits-are-revisions-deletions-are-tombstones)).

These are [§FS-sending.4](../functional-spec/FS-sending.md#4-the-thread-generation)'s events, and they are exactly the moves ephor can
also see, so the reply descriptor ([§AR-forge.4](AR-forge.md#4-the-reply-descriptor)) changes only when the
thread has moved.

**Example.** Operation 7 is accepted at generation 3 and the conversation moves
to 4. The next refresh reads the sent copy from Sent with `Message-ID:
<akoe.7.3f9c@example.org>`, finds it is operation 7's, and leaves the
generation at 4.

## 6. Resolving an `unknown`

`sudo -u akoe akoe resolve <account> <operation> sent|not-sent` records a
person's finding on an `unknown` operation, with the evidence Akoe showed beside
it ([§FS-sending.3](../functional-spec/FS-sending.md#3-unknown-blocks-the-thread)). `sent` advances the generation ([§AR-ledger.5](AR-ledger.md#5-the-generation)); `not-sent`
does not. Either unblocks the conversation. The words match ephor's own
`ephor reply ID --resolve sent|not-sent`, which settles ephor's saved send and
not Akoe's ledger, so a person who resolves in one resolves in the other too
([§FS-loop.4](../functional-spec/FS-loop.md#4-when-a-send-is-unknown)). Akoe never resolves an operation from evidence by itself,
however strong the evidence.

**Example.** Operation 10 is `unknown`; a message with its `Message-ID` turns up
in Sent and `akoe operation` shows it as evidence. The owner runs `sudo -u akoe
akoe resolve mail-me 10 sent`; the conversation advances to the next generation
and accepts replies again.
