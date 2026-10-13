# Architecture

How Akoe is built, one part per file. Each H1 declares an `AR-<slug>` ID, and
its numbered sections are citable. A point here realizes the functional specs
it cites; it never adds behavior they do not state.

| ID | Subject |
|---|---|
| [§AR-overview](AR-overview.md#ar-overview-one-daemon-running-as-akoes-own-user-owns-every-account-thin-clients-carry-requests-to-it) | the daemon, its user and sockets, account actors, what one request does, recovery, logs |
| [§AR-ledger](AR-ledger.md#ar-ledger-admission-is-one-transaction-an-attempt-is-bracketed-by-two-and-recovery-never-resubmits) | an operation's states, admission, the attempt, recovery, the generation, resolving |
| [§AR-store](AR-store.md#ar-store-one-sqlite-file-per-account-its-tables-split-into-record-and-cache) | the per-account SQLite file, record and cache tables, quota, backup, purge |
| [§AR-networks](AR-networks.md#ar-networks-an-adapter-either-reads-the-networks-history-or-keeps-a-live-session-and-akoes-own-trait-is-all-the-actor-sees) | readers, keepers and sessions; the adapter traits; the lease; each network |
| [§AR-forge](AR-forge.md#ar-forge-akoe-forge-maps-ephors-forge-protocol-onto-akoes-sockets-and-adds-nothing-of-its-own) | `akoe-forge` against ephor's forge contract; the reply descriptor; source blocks |
| [§AR-rpc](AR-rpc.md#ar-rpc-clients-speak-one-json-request-per-connection-over-three-sockets-whose-paths-are-the-permissions) | the socket protocol, its methods and error codes |
| [§AR-crates](AR-crates.md#ar-crates-one-workspace-one-crate-per-seam-so-each-part-can-be-built-and-tested-on-its-own) | the workspace, which crate depends on which, what lands first |
| [§AR-testing](AR-testing.md#ar-testing-every-spec-example-is-a-test-the-networks-are-faked-where-they-can-be-and-proven-by-hand-where-they-cannot) | test layers, adapter conformance, kill points, the ephor contract, the keeper proof |

This index is navigational — citations should target the spec ID directly,
never this file.
