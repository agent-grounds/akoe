# FS-loop: a conversation goes round ephor's loop and its answer comes back through akoe

One conversation, followed from the network into ephor's feed and its answer
back out, naming which side does each step. This is the whole of
[§GRUND-akoe.1](../grund.md#1-what-it-removes) told as one story, so that the parts built separately are judged
against it, and an end-to-end test plays it against the ephor Akoe pins. Every
ephor behavior named here has shipped, except the one [§FS-privacy.5.5](FS-privacy.md#55-a-definite-refusal-releases-the-saved-send) asks
for.

## 1. Setting up

### 1.1 Installing Akoe

Installing Akoe creates its user `akoe`, the groups `akoe-read` and
`akoe-send`, and the service that holds every account, and puts the owner in
both groups ([§FS-privacy.2.2](FS-privacy.md#22-akoes-own-unix-user)). Nothing is paired yet, and no real account is
paired before agents are confined ([§FS-privacy.4.3](FS-privacy.md#43-no-real-account-before-confinement)).

**Example.** After the install, `akoe status` run as the owner prints `no
accounts`, and `id` lists `akoe-read` and `akoe-send` among the owner's groups.

Source: V-A21, V-A22; settled 2026-10-13.

### 1.2 Pairing an account and choosing its scope

The person pairs each account at a terminal ([§FS-privacy.1](FS-privacy.md#1-pairing-is-a-persons-act)), choosing its scope
from choices that say what they leave out ([§FS-record.7](FS-record.md#7-declared-scope)), and lists the owner's
identities ([§FS-names.4.2](FS-names.md#42-the-owners-identities-and-mine)). `akoe config show` then says what the account
reads and what its network cannot report ([§FS-record.6.3](FS-record.md#63-what-a-network-cannot-report-is-stated)).

**Example.** The owner runs `sudo -u akoe akoe pair mail-me` for the test mailbox
`owner@example.org`, picks "all delivered mail since 2026-10-01", and `akoe
config show` reports `mail-me: owner@example.org, all delivered mail since
2026-10-01, no edits, deletions not reported`.

Source: V-A16, V-A21; settled 2026-10-13.

### 1.3 Giving ephor the sources

`akoe config sources` prints one block per source ([§FS-interfaces.2.3](FS-interfaces.md#23-a-budget-of-about-ten-sources)). The owner
adds them, unchanged, to the site `sources` of ephor's `status.json`, so ephor
asks each one once per refresh and its own attribution places the rows. The
rest is ephor's configuration, never Akoe's:

- sources whose work stays the owner's are listed in ephor's
  `work.private.sources`, with the root that work goes to ([§FS-privacy.5.1](FS-privacy.md#51-ownership-follows-the-source));
- a registry row claims a venue by its exact room ([§FS-names.3.1](FS-names.md#31-a-room-is-the-networks-venue-whichever-account-sees-it)), a
  correspondent by the author's address ([§FS-names.4.4](FS-names.md#44-an-author-is-reported-by-address)), or a whole source as
  its fallback home;
- a personal source is the fallback of the owner's own `me` row and of nothing
  else until the owner decides otherwise ([§FS-privacy.5.4](FS-privacy.md#54-personal-sources-reach-site-wide-placement-last)).

No row lists the owner's own address: every conversation the owner spoke in
would match it.

**Example.** The owner pastes `{"provider": "mail-me", "command":
"/usr/bin/akoe-forge", "account": "mail-me", "timeout_seconds": 60}` into
`sources`, adds `"mail-me"` to `work.private.sources` with the root
`~/me/private/{org}/{project}`, and gives the `me` row `"fallback_sources":
["mail-me"]`. A supplier row claims `orders@supplier.example.org`.

Source: V-A10, V-A11, V-A12, V-A13; ephor's FS-008, FS-018 and DF-005 (2026-10-08); settled 2026-10-13.

## 2. A conversation reaches the feed

`ephor refresh` probes `akoe-forge capabilities`, then runs `akoe-forge
messages` for each source. Akoe vouches for the account, by a read that
completes now or by a keeper's lease ([§FS-record.1.2](FS-record.md#12-the-read-is-the-observation), [§FS-record.3.2](FS-record.md#32-the-vouching-lease)), and
answers every conversation in scope, whole ([§FS-interfaces.3.1](FS-interfaces.md#31-messages)). ephor keys each
row `<provider>:<id>`, places it by room, author or fallback, and shows it
waiting when its last message is not the owner's.

**Example.** The supplier writes "Can you confirm order 12?". The next refresh
reads it from the server, Akoe answers it as conversation `k1` with a reply
descriptor at generation 1, and ephor shows `mail-me:k1`, "Order 12", under the
supplier's project, waiting on the owner.

Source: V-A04, V-A05, V-A06, V-A10; ephor's FS-001-forge-interface (2026-10-08).

### 2.1 When a source cannot vouch

A source that cannot vouch fails as a whole, with one line naming the account and
the cause. ephor keeps its last rows, marked stale, so a failure never reads as
"nothing new". When Akoe's service is not running at all, the line says so in
the words ephor reads as unreachable.

**Example.** The mail server refuses logins for an hour. Every refresh in that
hour shows `mail-me` failed with `mail-me: read failed: authentication refused`,
and `mail-me:k1` stays in the feed marked stale.

Source: V-A05; ephor's FS-001-forge-interface (2026-10-08).

## 3. An answer goes back

### 3.1 Drafting is ephor's

ephor drafts an answer as work on the row, from what the row showed, and binds
the draft to the thread and its reply descriptor. Akoe takes no part: it sees
neither the draft nor the work, and a private source's answer runs only when a
person starts it ([§FS-privacy.5.2](FS-privacy.md#52-an-answer-on-a-private-source-is-a-persons-act)).

**Example.** The owner starts an answer on `mail-me:k1`. Its plan runs under
`~/me/private/…`, and the draft "Thanks, confirmed." is bound to generation 1.

Source: V-A11, V-A23.

### 3.2 Posting

The owner presses `p` on the draft. ephor checks the draft against what its last
refresh showed and refuses a stale one itself, saves the send it is about to
make, and runs `akoe-forge reply` with the draft's descriptor and words. Akoe
admits it ([§FS-sending.5](FS-sending.md#5-admission)), attempts it once ([§FS-sending.2](FS-sending.md#2-one-attempt-three-outcomes)), and answers the
outcome ([§FS-interfaces.3.2](FS-interfaces.md#32-reply)). On `accepted`, ephor confirms the saved send and
retires the draft.

**Example.** `p` on "Thanks, confirmed." becomes operation 7 on `mail-me`,
addressed as Reply All ([§FS-sending.11](FS-sending.md#11-a-mail-reply-is-addressed-and-filed-as-a-mail-client-would)). The SMTP server accepts it, the thread
moves to generation 2, `akoe-forge` prints `{"status": "accepted"}`, and ephor
prints that the reply was posted.

Source: V-A08, V-A15, ruling V-D02; ephor's FS-005-dispatch (2026-10-08).

### 3.3 The answer comes back as the owner's

The next refresh reads the sent copy and reports it as the owner's, so the row
stops waiting. Akoe knows the copy as operation 7's message, by the id it set,
and does not move the generation again ([§FS-sending.4](FS-sending.md#4-the-thread-generation)).

**Example.** The refresh after the post reports "Thanks, confirmed." with `mine:
true` as the last message of `mail-me:k1`, and ephor no longer shows the row
waiting. The descriptor still says generation 2.

Source: V-A02, V-A08.

## 4. When a send is unknown

When an attempt may have reached the network with no acknowledgement, Akoe
answers `{"status": "unknown"}` with a note naming the operation. ephor holds its
saved send and Akoe blocks the thread ([§FS-sending.3](FS-sending.md#3-unknown-blocks-the-thread)); neither sends anything on
that thread until a person looks at the network and records what they found, in
both:

- `ephor reply <row> --resolve sent|not-sent` settles ephor's saved send;
- `sudo -u akoe akoe resolve <account> <operation> sent|not-sent` settles
  Akoe's operation.

The words are the same in both, and the order does not matter. Resolving in
ephor alone leaves Akoe refusing the thread as blocked, naming the operation.

**Example.** `p` on "See you Monday" times out after the final `.` of the SMTP
session. Akoe records operation 10 `unknown`, and ephor shows `mail-me:
operation 10 may have reached the network; resolve it with akoe resolve`. The
owner finds the mail in the supplier's reply quote, runs `ephor reply mail-me:k1
--resolve sent` and `sudo -u akoe akoe resolve mail-me 10 sent`, and the next
draft on the thread posts normally.

Source: V-A01, V-A08; ephor's FS-005-dispatch (`--resolve`, 2026-10-08); settled 2026-10-13.

## 5. Other turns the loop can take

| What happens | ephor | Akoe | The person |
|---|---|---|---|
| ephor's timeout ends `akoe-forge` during the attempt | keeps the saved send, and offers it again | finishes the attempt and records its outcome | presses `p` again, and gets that outcome; nothing is sent twice ([§FS-sending.5.1](FS-sending.md#51-an-identical-request-returns-the-recorded-outcome)) |
| a message arrived that ephor has fetched | refuses the draft as stale, shows what moved, offers a new draft | is not called | drafts again or types other words |
| a message arrived that only Akoe has seen | calls Akoe | refuses as stale, naming the message ([§FS-sending.5.3](FS-sending.md#53-a-stale-never-sent-draft-is-refused)) | sees the refusal; the saved send stays pending until ephor ships [§FS-privacy.5.5](FS-privacy.md#55-a-definite-refusal-releases-the-saved-send) |
| the network refuses the message | keeps the saved send pending | records `rejected` | as above |
| Akoe's service is down | shows the source unreachable and keeps the saved send | admits nothing | presses `p` once it is back, and the send goes out then |
| the source's block was changed | refuses to recover the saved send | is not called | puts the block back as `akoe config sources` prints it ([§FS-interfaces.2.3](FS-interfaces.md#23-a-budget-of-about-ten-sources)) |
| Akoe was restored from a copy | calls Akoe | refuses every send on the account ([§FS-record.2.3](FS-record.md#23-backup-is-a-persons-act)) | checks what was sent since the copy, then releases the account |

**Example.** The owner presses `p`, and the SMTP server takes 40 seconds to
accept; ephor's timeout is 30. ephor reports the call failed and keeps the saved
send. Akoe records operation 7 `accepted` at second 40. The owner presses `p`
again, and Akoe answers operation 7's acceptance without opening a session.

Source: V-A01, V-A08, V-A15, ruling V-D02; ephor's FS-001-forge-interface (reply reconciliation, 2026-10-08); settled 2026-10-13.

## 6. Programs go round without ephor

A program does not use ephor's feed. It sends with `akoe send` under its own
key, waits with `akoe wait`, reads with `akoe thread`, answers inside the
conversation with `akoe send --reply-to`, and checks an outcome with `akoe
operation` ([§FS-interfaces.4](FS-interfaces.md#4-the-command-line-for-programs)). It keeps its own records and decides by exit code,
so an `unknown` never reads as sent. If the program's account is also an ephor
source, its conversations appear in the feed like any other, and its `replies`
setting decides whether ephor may answer them ([§FS-sending.8](FS-sending.md#8-the-sender-allowlist-guards-against-mistakes)).

**Example.** crop-warden's program state sends lead 12's first mail with key
`lead-12-outreach-1` and stores the conversation `k9`. Each morning it runs
`akoe wait --account mail-sourcing --after <cursor>`. When it exits 0 naming
`k9`, the program reads the supplier's reply with `akoe thread`, and a person
approves the follow-up, which goes out with `--reply-to` and key
`lead-12-outreach-2`.

Source: V-A17, V-A18, V-A19; settled 2026-10-13.
