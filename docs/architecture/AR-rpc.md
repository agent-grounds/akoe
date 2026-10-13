# AR-rpc: clients speak one JSON request per connection over three sockets whose paths are the permissions

The local protocol between the thin clients and `akoed`
([§AR-overview.2](AR-overview.md#2-users-groups-and-sockets)). The crate `akoe-rpc` defines every request and answer as
Rust types with serde, the framing, and a blocking client; `akoed`, `akoe` and
`akoe-forge` all depend on it, and nothing else defines a message.

## 1. Framing

A client connects, writes one JSON object and a newline, and reads one JSON
object and a newline; then both sides close. A request carries `"v": 1` and a
`method`. An answer is `{"ok": …}` or `{"error": {"code": …, "message": …,
"operation": …}}`. A method sent on the wrong socket is answered with the error
`wrong-socket`, so the socket, not the method name, is what grants.

One method streams: `pair` writes progress objects (a QR payload, a pairing
code, a prompt for a password) before its final answer, each on its own line
with `"progress": …`.

**Example.** `akoe operation --account mail-me 41` writes
`{"v":1,"method":"operation","account":"mail-me","number":41}` to `read.sock`
and reads `{"ok":{"number":41,"state":"accepted",…}}`.

## 2. Methods

| Method | Socket | Request | Answer |
|---|---|---|---|
| `messages` | read | account, scope | conversations, or an error naming why the source cannot vouch |
| `thread` | read | account, conversation | one conversation with its descriptor ([§FS-interfaces.4.2](../functional-spec/FS-interfaces.md#42-akoe-thread)) |
| `operation` | read | account, number or caller key | the operation and its evidence ([§FS-interfaces.4.3](../functional-spec/FS-interfaces.md#43-akoe-operation)) |
| `wait` | read | account, cursor | the new cursor and the conversations that moved ([§FS-interfaces.4.4](../functional-spec/FS-interfaces.md#44-akoe-wait)) |
| `status` | read | none | per account: whether it vouches and why not, gaps, quota, `unknown` operations |
| `config-show` | read | none | each account's declared scope, senders, and what its network cannot report ([§FS-record.7](../functional-spec/FS-record.md#7-declared-scope)) |
| `config-sources` | read | none | the status.json blocks ([§AR-forge.5](AR-forge.md#5-source-blocks)) |
| `reply` | send | account, descriptor, text | the outcome ([§AR-forge.3](AR-forge.md#3-reply)) |
| `send` | send | account, as, caller key, recipients and subject or a descriptor, body, label | the outcome and the conversation ([§FS-interfaces.4.1](../functional-spec/FS-interfaces.md#41-akoe-send)) |
| `pair` | admin | account | progress (a QR or a code), then the verified identity |
| `retire`, `purge` | admin | account, conversation | done ([§FS-record.4](../functional-spec/FS-record.md#4-retirement-and-quotas)) |
| `resolve` | admin | account, operation, `sent` or `not-sent` | done ([§AR-ledger.6](AR-ledger.md#6-resolving-an-unknown)) |
| `ack-gap` | admin | account, gap | done ([§FS-record.3.4](../functional-spec/FS-record.md#34-gaps)) |
| `reset-identity` | admin | account | the new epoch ([§FS-names.4.3](../functional-spec/FS-names.md#43-an-identity-reset)) |
| `backup`, `restore` | admin | directory | the files written or restored ([§FS-record.2.3](../functional-spec/FS-record.md#23-backup-is-a-persons-act)) |
| `release-restore` | admin | account | done |
| `reload` | admin | none | the accounts started, stopped and changed |

## 3. Error codes

| Code | Meaning | `akoe` exits |
|---|---|---|
| `refused` | admission refused it; nothing was admitted | 2 |
| `rejected` | admitted, and the network definitively refused it | 3 |
| `cannot-vouch` | the source cannot vouch now | 4 |
| `not-found` | no such source, account, conversation or operation | 5 |
| `wrong-socket` | this socket does not grant this method | 77 |
| `internal` | a defect in `akoed` | 70 |

A client that cannot connect exits 69, and one refused the admin socket by its
permissions says to run the command as `sudo -u akoe`. An `unknown` outcome is
an answer, not an error, and `akoe` prints it on stdout and exits 6, so a
program that reads only the exit code never takes it for success
([§FS-interfaces.4](../functional-spec/FS-interfaces.md#4-the-command-line-for-programs)). `wait` with nothing new is an answer too, and exits 75.

**Example.** crop-warden's `akoe send` is refused because its key is already in
the ledger with other words. It exits 2 and prints `mail-sourcing: refused: key
lead-12-outreach-1 is operation 41 with a different body`.

## 4. Versions

`v` is the protocol version. `akoed` answers a request with an unknown `v` with
an error naming the versions it speaks, and the clients and `akoed` ship from
one release, so a mismatch means a half-done upgrade. A field is only ever
added, with a default; renaming or removing one is a new version.
