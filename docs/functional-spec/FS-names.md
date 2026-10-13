# FS-names: a conversation's id never changes, a room names a venue, and the owner is one person across accounts

How Akoe names conversations, rooms, accounts and the owner. ephor keys a row
as `<provider>:<conversation id>` and hangs its drafts and work on that key, so a
conversation reported under a new id loses its draft from view. Stable
conversations are what [§GRUND-akoe.1](../grund.md#1-what-it-removes) promises.

## 1. Conversation ids

### 1.1 Network-native where that is enough

A conversation's id is built from the network's own identifiers where those are
enough:

| Network | One conversation per |
|---|---|
| WhatsApp | chat JID, with its LID and phone forms as aliases |
| Signal | service identity or group id, never a phone number |
| Slack | direct conversation, and channel thread |
| Discord | channel or thread |
| Matrix | direct room, a room's main timeline (`!room:server`), and each thread in a room (`!room:server/<thread root event id>`) |
| Mail | thread: an issued id ([§FS-names.1.2](FS-names.md#12-issued-for-generic-mail)) |

The account is not part of the id: the source's provider name already names it.
A Matrix room's alias, such as `#suppliers:example.org`, can be moved to another
room, so it is never part of an id.

**Example.** A direct WhatsApp chat with +1 555 0142 is row
`wa-me:15550142@s.whatsapp.net`. When the network later reveals the contact's
LID, it joins that id as an alias and the row keeps its key.

Source: V-A06, V-C10; settled 2026-10-13.

### 1.2 Issued for generic mail

For mail without a usable server thread id, Akoe issues an opaque id over the
messages' `Message-ID`, `References` and `In-Reply-To` headers, and keeps it in
the record ([§FS-record.2.2](FS-record.md#22-record-and-cache)).

**Example.** A supplier's first mail to `owner@example.org` arrives with
`Message-ID: <a1@supplier.example.org>` and no references. Akoe issues `k1`,
and ephor shows row `mail-me:k1`.

Source: V-A04, V-A06.

### 1.3 An issued id never changes

An id, once reported, is never taken back or changed, except by a deliberate
identity reset ([§FS-names.4.3](FS-names.md#43-an-identity-reset)). When later evidence reveals another form of the
same conversation, such as an older parent message or another form of a
contact's address, that form joins the id as an alias.

**Example.** Mail C arrives citing only B, and Akoe issues `k1`. Later B
arrives citing an older A. A and B join `k1`; the row stays `mail-me:k1`, with
its draft.

Source: V-A04, V-A06.

### 1.4 Merged threads keep both ids

When late evidence shows that two conversations with issued ids are one thread,
both ids stay and both rows keep being reported. Akoe retires neither, because
it cannot see which row holds a draft. From the merge on, both rows report the
whole thread, every message of both, with the same reply descriptor and the same
generation, and a new message appears in both. A draft on either row is judged
against that one generation, so the two rows cannot send the same thing twice.

**Example.** Rows `mail-me:k3` and `mail-me:k4` were issued for two mails that a
later reply shows to be one thread. Both rows stay in ephor, each with all three
mails, and a draft on either is not stranded. "Thanks, confirmed." posted from
`k3` is accepted; the same words posted from `k4` find that operation and send
nothing.

Source: V-A27; settled 2026-10-13.

## 2. Threading generic mail

Generic mail is threaded by `References` and `In-Reply-To`, never by subject.
async-imap 0.11.3 parses Gmail's `X-GM-THRID` but exposes no accessor for it, so
Gmail mail is threaded by its headers too. An account never changes how it
names its threads: once Gmail's thread id can be read, it joins the issued ids
as an alias and as evidence that two messages are one thread, and where it joins
two issued ids, [§FS-names.1.4](FS-names.md#14-merged-threads-keep-both-ids) applies.

**Example.** A supplier's reply carries only `In-Reply-To: <b2@example.org>`.
Gmail files it in its own thread, but Akoe sees only the header and threads it
with the message `<b2@example.org>`. Two unrelated mails both titled "Order"
stay two conversations.

Source: V-A06, V-C09; settled 2026-10-13.

## 3. Rooms

### 3.1 A room is the network's venue, whichever account sees it

A room, such as a group or a channel, is named from the network's own id for the
venue. The name is the same whichever of the owner's accounts observes it, so
one registry claim places the venue for every account that sees it. How a room
is spelled is Akoe's configuration; ephor matches it exactly.

**Example.** WhatsApp group G is room `whatsapp:G@g.us` in the answers of both
`wa-a` and `wa-b`. One claim on `whatsapp:G@g.us` in a registry row places both
rows.

| Network | Room |
|---|---|
| WhatsApp | `whatsapp:<group id>@g.us` |
| Signal | `signal:group:<group id>` |
| Slack | `slack:<team id>/<channel id>`, such as `slack:T024BE7LD/C0123ABCD` |
| Discord | `discord:<guild id>/<channel id>`; a thread is in its parent channel's room |
| Matrix | `matrix:<room id>`; a thread is in its room's room |
| mail | `mail:<address>/label/<label>` ([§FS-names.3.3](FS-names.md#33-a-label-the-owner-applies-can-be-a-room)) |

Every part is the network's own id, never a display name, which a member can
change.

Source: V-A06, V-A13; settled 2026-10-13.

### 3.2 A direct conversation has no room and no reason

A direct conversation is reported with no room and with an empty reasons list;
ephor's ordinary rule makes it wait when the last message is not the owner's.
A conversation in a room carries facts about the owner's part in it, from
ephor's closed list, never a judgement of whether it matters:

| Reason | When |
|---|---|
| `mentioned` | a message carries the network's own mention of one of the owner's identities; words that merely contain the owner's name are not a mention |
| `in-thread` | one of the owner's identities wrote a message in the conversation |
| `authored` | one of the owner's identities wrote its first message |

Akoe never reports `review-requested` or `assigned`.
Akoe never reports a `direct` reason, since an unknown reason word fails
ephor's whole answer. The conversation carries no inbox field either: where a
personal inbox belongs is a registry row's fallback claim on the source, which
is ephor's.

**Example.** A WhatsApp DM whose last message is the contact's "Can you call
me?" is reported with `"reasons": []` and no room. ephor shows it waiting.

**Example.** In room `slack:T024BE7LD/C0123ABCD`, a colleague writes "@owner can
you look at rhei#12?", tagging the owner. The thread is reported with
`"reasons": ["mentioned"]`. Once the owner answers in it, `["mentioned",
"in-thread"]`.

Source: V-A12, V-C15; settled 2026-10-13.

### 3.3 A label the owner applies can be a room

A mail label can be a room: a thread is in room `mail:<address>/label/<label>`
when any of its messages carries the label, computed from Akoe's conversation
map. Akoe never applies a label on reception; a label comes only from the
sender's explicit act ([§FS-interfaces.5](FS-interfaces.md#5-labels-are-the-senders-act)).

**Example.** crop-warden sends lead 12's first mail with `--label suppliers`.
The supplier's reply carries no label, but its thread does, so the thread is in
room `mail:sourcing@example.org/label/suppliers`, which crop-warden's registry
row claims, with no registry edit per lead.

Source: V-A06, V-A17.

## 4. Accounts and identity

### 4.1 The account namespace

Each account has an internal UUID that stays inside Akoe. Pairing again to the
same verified remote identity keeps it. Pairing a different identity creates a
new namespace, so the first identity's history is never shown as the second's.
Akoe verifies and records the remote identity before it reuses an account's id.

**Example.** `wa-me` is paired to +1 555 0100. Later the owner points `wa-me` at
+1 555 0177. Akoe sees a different identity and starts a new namespace; none of
the first number's chats appear under the second.

Source: V-A06, V-A21.

### 4.2 The owner's identities and `mine`

Akoe's configuration lists the owner's identities across all accounts:
addresses, numbers and user ids. A message is the owner's (`mine`) when any of
them wrote it, in any account's conversation.

**Example.** The owner is in WhatsApp group G from accounts A (+1 555 0100) and
B (+1 555 0101). A message the owner sends from B is reported as `mine` in A's
row for G too.

Source: V-A06.

### 4.3 An identity reset

A deliberate identity reset gives an account's conversations new ids. It keeps
the operation ledger ([§FS-sending.10.2](FS-sending.md#102-an-identity-reset-keeps-the-ledger)). Rows and drafts that ephor holds under
the old ids are stranded; that is the cost the person accepts by resetting.

**Example.** After a reset, `mail-me:k1` comes back as `mail-me:k1b`. The
draft on `mail-me:k1` stays filed under the old row, and operation 41 on that
thread is still in the ledger.

A person resets an account with `sudo -u akoe akoe reset-identity <account>`,
which asks for the account's name to be typed again before it acts.

Source: V-A27, V-C10; settled 2026-10-13.

### 4.4 An author is reported by address

Each message's author is reported as an address, in one form per network, so
that ephor can place a conversation by its authors:

| Network | Author |
|---|---|
| mail | the address, in lower case: `orders@supplier.example.org` |
| WhatsApp | the number, `+15550142`, or the hidden form `<id>@lid` where the network shows no number |
| Signal | the number where Signal shares it, otherwise the service id |
| Slack | `<team id>/<user id>`, such as `T024BE7LD/U012AB3CD` |
| Discord | the user id |
| Matrix | the user id, `@name:example.org` |

The form first reported for an author is kept. When the network later shows
another form of the same author, such as a WhatsApp contact's number behind a
hidden id, it joins as an alias, and the messages keep their first form, because
ephor recognises a message partly by its author. The author's display name is
reported beside it as Akoe's own key `author_name`.

**Example.** A WhatsApp contact first writes from a hidden id and is reported as
`8411337102341@lid`. A week later the network reveals the number +1 555 0142.
The contact's messages, old and new, are still reported as `8411337102341@lid`,
and `akoe thread` lists `+15550142` as its alias.

Source: V-A06, V-A12; ephor's FS-001-forge-interface (attribution from authors, 2026-10-08); settled 2026-10-13.

## 5. A group seen from two accounts

A room the owner belongs to from two accounts is reported by both by default.
The configuration may name one account with `report_from` for that room; the
other account then stops reporting it, without deleting its record. A selected
account that cannot vouch fails, rather than borrowing the other account's
observation. Switching `report_from` later moves the room to other row keys,
with the cost of [§FS-names.4.3](FS-names.md#43-an-identity-reset) for drafts on the old rows.

**Example.** The owner is in group G from accounts A and B. ephor shows
`wa-a:G` and `wa-b:G`. Setting `report_from = "A"` for G removes B's row; B's
record of G stays. When A's keeper loses its lease, `wa-a` fails and B does not
stand in.

Source: V-A06.

## 6. A thread that changes scope changes its key

Where an account is split into sources by scope ([§FS-interfaces.2.2](FS-interfaces.md#22-a-split-for-ownership-or-for-a-selector-never-for-isolation)), a thread
that moves from one scope to another is reported by the other source, under
that source's provider name. Its row key changes. This is a known limit.

**Example.** Account `mail-biz` is split into `mail-biz-direct` and
`mail-biz-rooms` by label. When the owner labels a direct thread `suppliers`, it
moves to `mail-biz-rooms`, and row `mail-biz-direct:k7` becomes
`mail-biz-rooms:k7`; a draft on the old row stays there.

Source: V-A06, V-A12.
