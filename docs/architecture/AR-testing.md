# AR-testing: every spec example is a test, the networks are faked where they can be and proven by hand where they cannot

What proves Akoe, and the shared harness every implementer builds on, so that
no two of them invent their own. A change is done when the functional-spec
points it touches are each cited by a passing test, and `grund check` and the
gates below pass.

## 1. The layers

| Layer | Where | Cites | Runs |
|---|---|---|---|
| unit | each crate's `src` and `tests` | the AR point it realizes | every push |
| integration | `tests/integration/` | AR | every push |
| end-to-end | `tests/e2e/` | FS, never AR | every push, with the fakes |
| network | `tests/network/` | FS | nightly, against test servers and test accounts |
| keeper proof | `tests/keeper-proof/` | [§GRUND-akoe.3](../grund.md#3-when-it-shrinks) | by hand, before a release and after a library bump |

The workspace's root package `akoe-acceptance` owns the `tests/` targets.
Nothing in `tests/` holds a real name, address, number, message or session
([§FS-privacy.3](../functional-spec/FS-privacy.md#3-the-repository-holds-nothing-personal)).

## 2. Unit tests hold the rules

`akoe-ledger` takes the store and the adapter through traits, so admission,
the attempt protocol, recovery and the generation are tested with in-memory
fakes. Every row of [§FS-sending.6](../functional-spec/FS-sending.md#6-the-five-cases) and every bullet of [§FS-sending.9](../functional-spec/FS-sending.md#9-what-akoe-never-does-when-sending) has a
unit test here, and again end to end.

**Example.** A unit test admits "Thanks, confirmed." at generation 3, has the
fake adapter answer `accepted`, then submits the same request at generation 3
after the conversation reached 5, and asserts the fake adapter saw one
`attempt`.

## 3. Adapter conformance

`akoe-net` ships a conformance kit: one suite of tests, parameterized by an
adapter and a scripted peer, that every adapter crate runs. It asserts:

- an `attempt` is one submission, under every scripted fault;
- the classification of [§AR-networks.2](AR-networks.md#2-the-adapter-traits) at each point of the exchange;
- nothing a contact or another client could see is sent on reception
  ([§FS-record.5](../functional-spec/FS-record.md#5-receiving-has-no-side-effects));
- `commit` precedes the network's acknowledgement;
- a lease is never granted from anything but the receiving session.

The scripted peers are:

| Network | Peer for unit and conformance tests | Peer for the network layer |
|---|---|---|
| Mail | `akoe-testkit`'s scripted IMAP and SMTP servers, which log every command | a GreenMail container |
| Matrix | recorded `/sync` and `/messages` exchanges | a local homeserver container |
| Slack, Discord | a scripted WebSocket and HTTP peer from the platforms' published schemas | a test workspace and a test guild |
| WhatsApp, Signal | a fake of the library's client trait | the keeper proof's test pairs |

**Example.** The scripted IMAP server logs `EXAMINE INBOX`, `UID SEARCH SINCE
1-Oct-2026` and `UID FETCH … BODY.PEEK[]`. The conformance test fails if the log
holds any `SELECT`, `STORE` or a fetch without `PEEK`.

## 4. End-to-end tests prove the functional spec

An end-to-end test starts the built `akoed` with a temporary home and socket
directory as the test's own user, configured with fake accounts
(`akoe-net-fake`, scripted per test), and drives it only through `akoe-forge`
with ephor's requests and through `akoe`, as ephor and programs do. Each test
cites the FS point whose example it plays. The user split itself is tested by
an install test in a container that creates the users, groups and unit and
checks each socket's permissions.

**Example.** `tests/e2e/sending_five_cases.rs` cites [§FS-sending.6](../functional-spec/FS-sending.md#6-the-five-cases) and plays
cases (a) to (e) with the exact JSON ephor sends, counting the fake network's
submissions.

## 5. Kill points

Built with the `fault-injection` feature, `akoed` reads `AKOE_KILL_AT=<point>`
and aborts itself, as a `SIGKILL` would, when it reaches that point:
`after-admit`, `after-mark`, `after-submit`, `before-outcome-commit`,
`after-receive-before-commit`, `mid-observe`. A test restarts it and asserts the
recovered state ([§AR-ledger.4](AR-ledger.md#4-recovery-after-a-crash), [§FS-record.3.5](../functional-spec/FS-record.md#35-a-kill-loses-no-received-message)). Release builds do not
contain the feature.

**Example.** With `AKOE_KILL_AT=after-submit`, the fake SMTP peer receives the
message and `akoed` dies before recording the outcome. After the restart,
`akoe operation` shows the operation `unknown`, and a second identical request
makes no submission.

## 6. The ephor contract

Akoe pins one ephor commit, in `tests/ephor.pin`. Two suites hold Akoe to it:

- **Schema.** Every answer `akoe-forge` prints in the end-to-end tests is
  validated against ephor's `assets/ephor-forge.schema.json` and
  `assets/ephor-answer.schema.json`, vendored from the pinned commit.
- **The loop.** A test builds ephor at the pinned commit, writes a
  `status.json` from `akoe config sources`, runs `ephor refresh`, reads `ephor feed
  --json`, and posts with `ephor reply`, against `akoed` with fake accounts. It
  plays [§FS-loop](../functional-spec/FS-loop.md#fs-loop-a-conversation-goes-round-ephors-loop-and-its-answer-comes-back-through-akoe) from the first refresh to the reply read back as the
  owner's.

The loop test also plays each turn of [§FS-loop.5](../functional-spec/FS-loop.md#5-other-turns-the-loop-can-take) whose ephor side has
shipped, and asserts today's behavior for the one that has not: a definite
refusal leaves ephor's saved send pending ([§FS-privacy.5.5](../functional-spec/FS-privacy.md#55-a-definite-refusal-releases-the-saved-send)). When ephor ships
that change, the assertion is the first thing to fail, which is the signal to
move the pin and the forge's refusal answer together.

Moving the pin is a change of its own, made when ephor's contract changes.

## 7. Measurements

Some promises are numbers nobody has measured yet. Each is measured by the
network layer or the keeper proof and written into the FS point that names it:

- the cost of a mail refresh against conversation count, which bounds the
  timeout `akoe config sources` writes ([§FS-record.1.1](../functional-spec/FS-record.md#11-no-listener-and-no-body-store));
- each keeper's offline window ([§FS-record.3.6](../functional-spec/FS-record.md#36-downtime));
- ten sources refreshed by one ephor refresh ([§FS-interfaces.2.3](../functional-spec/FS-interfaces.md#23-a-budget-of-about-ten-sources)), on the
  generated ten-source fixture.

## 8. The keeper proof

`tests/keeper-proof/` is a runbook and a script for [§GRUND-akoe.3](../grund.md#3-when-it-shrinks)'s four
checks on a test pair: two test numbers per network, neither the owner's, held
by a person. The script drives the Akoe side and asks the person to act on the
test phone where a check needs a human. Its report is committed under
`docs/proofs/` with the library versions it ran against.

**Example.** Check 3: the script stops `akoed`, asks the person to send "Are you
free Sunday?" from the test phone, restarts `akoed`, and passes when the next
`messages` answer holds the message and the source vouches.

## 9. Continuous integration

Every pull request runs `cargo fmt --check`, `cargo clippy`, the unit,
integration and end-to-end tests, the ephor schema suite, `grund check` and
`fissile check`, on Linux. The network layer and the ephor loop run nightly and
before a release; the keeper proof runs by hand. Akoe runs on Linux only: it
depends on Unix users, peer credentials and systemd.
