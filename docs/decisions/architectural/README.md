# Architectural decisions

Decisions about how Akoe is built, and the tradeoffs behind them. One file per
decision; each H1 declares a `DA-<slug>` ID and the body is the record.

| ID | Subject |
|---|---|
| [§DA-network-libraries](DA-network-libraries.md#da-network-libraries-akoe-builds-on-the-libraries-zeroclaw-wraps-not-on-zeroclaw) | build on the libraries ZeroClaw wraps, not on ZeroClaw; what would reverse it |
| [§DA-local-daemon](DA-local-daemon.md#da-local-daemon-the-owners-processes-reach-akoe-through-a-daemon-and-unix-sockets-not-through-sudo) | one daemon as Akoe's user, reached through Unix sockets rather than `sudo` per call |

This index is navigational — citations should target the decision ID directly,
never this file.
