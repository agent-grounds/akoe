# DA-local-daemon: the owner's processes reach Akoe through a daemon and Unix sockets, not through sudo

**Status:** Accepted by the owner on 2026-10-13
**Date:** 2026-10-13

## 1. Context

Akoe runs as its own Unix user so that the owner's other processes, agents
included, cannot read its sessions or write its ledger ([§FS-privacy.2.2](../../functional-spec/FS-privacy.md#22-akoes-own-unix-user)).
ephor and programs run as the owner, and the founding verdict named no way for
them to reach Akoe's user. Keepers need a long-running process in any case,
because WhatsApp and Signal must hold a live session ([§FS-record.3.1](../../functional-spec/FS-record.md#31-what-is-kept)).

## 2. Decision

One daemon, `akoed`, runs as user `akoe` under systemd and owns every account.
The owner's processes reach it through Unix sockets in `/run/akoe/`: `read.sock`
and `send.sock` for groups the owner belongs to, and `admin.sock` for Akoe's
user alone, used by a person through `sudo -u akoe`. `akoe-forge` and `akoe` are
clients that hold nothing ([§AR-overview](../../architecture/AR-overview.md#ar-overview-one-daemon-running-as-akoes-own-user-owns-every-account-thin-clients-carry-requests-to-it)).

## 3. Why

- **One path for every network.** Mail read at refresh, keepers and sessions all
  live in the process that holds their credentials. With `sudo` per call, mail
  would run in a fresh process per refresh and keepers in a daemon anyway: two
  paths to the same store, and two writers to coordinate.
- **The socket path is the permission.** A sandbox denies `/run/akoe/send.sock`
  as it denies any file, and a deployment can grant `read.sock` without
  `send.sock`. A sudoers rule is a second policy language in a file a worker must
  also be kept from.
- **An attempt outlives its caller.** ephor kills a forge at its timeout. In the
  daemon, the attempt finishes and is recorded; in a per-call process it would
  die with the process and every timeout would become an `unknown`.
- **A person's act stays a person's.** The admin socket needs Akoe's user, which
  needs the owner's password at a terminal, so retiring, resolving and pairing
  cannot be done by anything that runs as the owner without one.

Source: the owner's answer of 2026-10-13, choosing "Daemon + sockets".

## 4. Rejected

- **`sudo -u akoe` for every call.** It is rejected for the reasons of
  [§DA-local-daemon.3](DA-local-daemon.md#3-why).
- **The owner's own user until confinement lands.** Simpler, but it gives up
  [§FS-privacy.2.2](../../functional-spec/FS-privacy.md#22-akoes-own-unix-user), which the founding verdict made a requirement and not a
  hardening step.

## 5. Consequences

Akoe runs on Linux only, with systemd and peer credentials. Installing it creates
a user, two groups and a unit, so it ships an install script and a test of
that install ([§AR-testing.4](../../architecture/AR-testing.md#4-end-to-end-tests-prove-the-functional-spec)). Upgrading the daemon restarts every session,
and each keeper earns its lease again.
