# GRUND-akoe: no person carries a conversation into the work loop, or an answer back

A matter received on a communication account still waits for a person to carry
it into an intermittent work loop, and an approved answer waits to be carried
back with a trustworthy account of what happened. Akoe removes that repeated
handoff: where the network cannot supply the conversation later it keeps the
session and a durable receiving record; where history can be read it reads that
history; in both cases it supplies stable conversations, and it admits each
authorized outbound operation once, never repeats an attempt that may have
reached the network, and records whether the network accepted it, rejected it,
or left it unknown. It proves the completeness it actually has and exposes a gap
or an uncertain send. It serves the intake, <§>GRUND-automatic.4, and claims no
reason of its own: placement, response policy and the answer stay with ephor,
the consuming project and the person.

This is the paragraph both participants of the founding agora signed (V-A01).
The agora wrote it under the placeholder name keryx; only the name has changed.

## 1. What it removes

Two carries, one in each direction. Inbound, a conversation on a mail or chat
account reaches ephor as a forge source, whole, under an id that does not
change. Outbound, an authorized message goes back through the account it
belongs to, and the record says what became of it.

The promise is about Akoe's own conduct and its record. An attempt is one
submission of a frozen message. A definite failure before submission may be
tried again; an attempt that may have reached the network is never repeated.
`accepted` means the network took the message. It does not mean the recipient
received it, or received exactly one copy. Completeness is promised the same
way: an answer is complete only while Akoe can show it is observing the
account now, and otherwise the source fails and names the gap.

**Example.** crop-warden's first mail to a new supplier is admitted as
operation 41, and the SMTP session times out after the server may have accepted
it. Akoe records `unknown`. Neither a retry by crop-warden nor anything inside
Akoe submits operation 41 again, and Akoe does not claim the supplier got one
copy.

Source: V-A01, V-C01.

## 2. What it deliberately does not do

Akoe decides nothing about work. Which project a conversation is about is
ephor's attribution. Whether a conversation waits on the owner is ephor's rule,
read from which messages Akoe marks as the owner's. Whether a source is
private, and where its work goes, is ephor's site configuration. What to answer
is the person's, with a proposal a run drafted. Akoe's own configuration holds
facts about networks: the accounts, their credentials, how rooms are spelled,
which projects may send through which account, and each account's declared
scope. Every network name stays in that site data; ephor's code never names
one.

Akoe is not a fifth foundation tool and adds no fifth reason to the
workspace's four. It serves the existing intake.

**Example.** "Source `wa-me` is private and its work goes to the overlay" is a
line in ephor's site configuration. "`wa-me` is the number +1 555 0100, its
session is under `~/.secrets/akoe/wa-me/`, and crop-warden may send through
`mail-sourcing`" are lines in `~/.config/akoe`.

Source: V-A01, V-A02, V-A13, V-C01.

## 3. When it shrinks

The part of Akoe that carries risk is the keeper: a process holding a live
session for a network that keeps no history Akoe can read later. WhatsApp is
where the keeper is proved, by four checks on a test pair:

1. The network is cut while the process lives: the source fails as stale within
   its maximum age.
2. The owner replies from the phone: the reply appears as the owner's.
3. A message is sent to the owner while Akoe is down: it appears after the
   restart.
4. Akoe is killed during a send: the record shows one send or one `unknown`.

If the keeper fails these checks, Akoe shrinks to adapters that read history at
refresh plus the send record. If the WhatsApp library cannot surface replies
the owner types on the phone, WhatsApp leaves scope: a feed in which every
answered chat looks unanswered is worse than none.

**Example.** On the test pair, the contact at +1 555 0142 writes "Are you free
Sunday?" and the owner answers "Yes" on the phone. If the next refresh shows the
chat still waiting on the owner, check 2 has failed and WhatsApp leaves scope.

Source: V-A02, V-A25.

## 4. Why a project of its own

Every other home breaks a given or a settled boundary:

- **Inside ephor**, as a companion crate, the networks' churn would ride on
  ephor's release train, which ephor's decision DF-003-chat-is-a-forge-capability
  refused: a network changes on its own schedule, not ephor's.
- **Inside `me`**, the owner's private repository, the organization's shared
  channels would depend on a private repository.
- **Upstream in ZeroClaw**, as a relay mode, ephor's contract would rest on a
  pre-1.0 agent gateway that drops what the contract needs.
- **As scripts per consuming project**, each would need its own session, and a
  network that allows one linked device per account cannot be shared that way.

So Akoe is a public project with its own grund, as skopos is.

**Example.** Two projects both answer on the organization's WhatsApp account,
+1 555 0199. As scripts per project, each would pair a linked device of its own
or copy the other's session files. Through Akoe, one paired session serves
both, and each sends only where the account's configuration lists it.

Source: V-A02.
