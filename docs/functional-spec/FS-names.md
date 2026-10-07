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
| Mail | thread: Gmail's thread id where Akoe can read it, otherwise an issued id ([§FS-names.1.2](FS-names.md#12-issued-for-generic-mail)) |

The account is not part of the id: the source's provider name already names it.

**Example.** A direct WhatsApp chat with +1 555 0142 is row
`wa-me:15550142@s.whatsapp.net`. When the network later reveals the contact's
LID, it joins that id as an alias and the row keeps its key.

Source: V-A06, V-C10.

- **Open:** the id for Matrix, which the verdict's grouping does not list.

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
it cannot see which row holds a draft.

**Example.** Rows `mail-me:k3` and `mail-me:k4` were issued for two mails that a
later reply shows to be one thread. Both rows stay in ephor, and a draft on
either is not stranded.

Source: V-A27.

- **Open:** which messages each of the two rows carries after the merge, and
  which row a new message joins.

## 2. Threading generic mail

Generic mail is threaded by `References` and `In-Reply-To`, never by subject.
Gmail's own thread id is used where Akoe can read it. async-imap 0.11.3 parses
Gmail's `X-GM-THRID` but exposes no accessor for it, so until one is added Gmail
mail is threaded by its headers too.

**Example.** A supplier's reply carries only `In-Reply-To: <b2@example.org>`.
Gmail files it in its own thread, but Akoe sees only the header and threads it
with the message `<b2@example.org>`. Two unrelated mails both titled "Order"
stay two conversations.

Source: V-A06, V-C09.

- **Open:** how Gmail's thread id, once readable, joins ids already issued for
  the same threads, given [§FS-names.1.3](FS-names.md#13-an-issued-id-never-changes).

## 3. Rooms

### 3.1 A room is the network's venue, whichever account sees it

A room, such as a group or a channel, is named from the network's own id for the
venue. The name is the same whichever of the owner's accounts observes it, so
one registry claim places the venue for every account that sees it. How a room
is spelled is Akoe's configuration; ephor matches it exactly.

**Example.** WhatsApp group G is room `whatsapp:G@g.us` in the answers of both
`wa-a` and `wa-b`. One claim on `whatsapp:G@g.us` in a registry row places both
rows.

Source: V-A06, V-A13.

- **Open:** the room grammar for each network. The verdict shows only
  `whatsapp:<jid>` and `mail:<address>/label/<label>`.

### 3.2 A direct conversation has no room and no reason

A direct conversation is reported with no room and with an empty reasons list;
ephor's ordinary rule makes it wait when the last message is not the owner's.
Akoe never reports a `direct` reason, since an unknown reason word fails
ephor's whole answer. The conversation carries no inbox field either: where a
personal inbox belongs is a registry row's fallback claim on the source, which
is ephor's.

**Example.** A WhatsApp DM whose last message is the contact's "Can you call
me?" is reported with `"reasons": []` and no room. ephor shows it waiting.

Source: V-A12, V-C15.

- **Open:** which reason words Akoe reports for a conversation in a room.

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

Source: V-A27, V-C10.

- **Open:** the reset command.

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
