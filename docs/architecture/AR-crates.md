# AR-crates: one workspace, one crate per seam, so each part can be built and tested on its own

How the code is laid out, and which crate may depend on which. The seams are
drawn so that parts can be built in parallel against types that land first.
No crate is published ([§FS-interfaces.6](../functional-spec/FS-interfaces.md#6-no-library-crate-and-no-mcp-server)); every one sets `publish = false`.

## 1. The crates

| Crate | Holds | Depends on |
|---|---|---|
| `akoe-model` | ids, generations, operations and outcomes, descriptors, scopes, diagnoses; no I/O | nothing of Akoe's |
| `akoe-config` | the configuration files, their validation, and the source blocks | model |
| `akoe-store` | the schema, migrations and every query ([§AR-store](AR-store.md#ar-store-one-sqlite-file-per-account-its-tables-split-into-record-and-cache)) | model |
| `akoe-ledger` | admission, the attempt protocol, recovery, the generation ([§AR-ledger](AR-ledger.md#ar-ledger-admission-is-one-transaction-an-attempt-is-bracketed-by-two-and-recovery-never-resubmits)) | model, net (traits only) |
| `akoe-net` | the adapter traits, the event sink, the lease, the conformance kit ([§AR-networks.2](AR-networks.md#2-the-adapter-traits)) | model |
| `akoe-net-fake` | a scripted network with faults, for every test layer | net |
| `akoe-net-mail`, `-whatsapp`, `-signal`, `-slack`, `-discord`, `-matrix` | one adapter each, on its library | net, model |
| `akoe-rpc` | the socket protocol and its client ([§AR-rpc](AR-rpc.md#ar-rpc-clients-speak-one-json-request-per-connection-over-three-sockets-whose-paths-are-the-permissions)) | model |
| `akoe-daemon` | `akoed`: accounts, actors, the lease, the sockets ([§AR-overview](AR-overview.md#ar-overview-one-daemon-running-as-akoes-own-user-owns-every-account-thin-clients-carry-requests-to-it)) | everything above |
| `akoe-forge` | the `akoe-forge` binary ([§AR-forge](AR-forge.md#ar-forge-akoe-forge-maps-ephors-forge-protocol-onto-akoes-sockets-and-adds-nothing-of-its-own)) | rpc, model |
| `akoe-cli` | the `akoe` binary | rpc, model |
| `akoe-testkit` | scripted IMAP and SMTP peers, temporary daemons, ephor request builders | dev-dependency only |

The clients depend on no store, adapter or library of a network, and a test in
`akoe-cli` asserts it from `cargo metadata`, so a client cannot grow a way
around `akoed`.

**Example.** A change to how Signal reports sync-sent events touches
`akoe-net-signal` alone; `akoe-forge` is not rebuilt, and the conformance kit
and the end-to-end tests are what tell whether it still fits.

## 2. What lands first

The order that lets the most work run at once:

1. **The frame:** the workspace, `akoe-model`, the traits of `akoe-net`, the
   types of `akoe-rpc`, `akoe-net-fake`'s skeleton, and CI. Every other part
   codes against these.
2. **In parallel:** `akoe-store`; `akoe-ledger` on in-memory fakes;
   `akoe-config` with the source blocks; `akoe-testkit`'s mail peers; the mail
   adapter; `akoe-forge` and `akoe-cli` against a stub daemon.
3. **Then:** `akoe-daemon`, which joins them; the end-to-end suite; the ephor
   loop.
4. **Alongside, from step 1:** each keeper and each history reader (Slack,
   Discord, Matrix) with its conformance run, and the keeper proof's runbook.

A part changes a type of step 1 only through its own small pull request, which
lands before the change that uses it.

## 3. The toolchain

The workspace pins one Rust toolchain in `rust-toolchain.toml`, high enough for
every adapter (Rust 1.96, for `matrix-sdk` 0.19.1), and commits `Cargo.lock`;
every build and install is `--locked` ([§DA-network-libraries.2](../decisions/architectural/DA-network-libraries.md#2-decision)). Each network
library is pinned to an exact version, and moving one reruns that adapter's
conformance kit and, for a keeper, the keeper proof.
