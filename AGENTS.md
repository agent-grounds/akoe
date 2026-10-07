# AGENTS.md

Akoe is a message gateway between mail and chat accounts and ephor. The
repository holds draft specifications only; start at README.md and
docs/grund.md, and write or change the spec point before any behavior.

Examples and fixtures are synthetic: `example.org` addresses and +1 555 01xx
numbers, never a real name, address, number, message or session
([§FS-privacy.3](docs/functional-spec/FS-privacy.md#3-the-repository-holds-nothing-personal)).

Validation: `grund check`.

<!-- BEGIN GRUND MANAGED BLOCK -->
## Grounding with grund (v12)

This project uses [`grund`](https://github.com/agent-grounds/grund): every spec, goal, decision, and end-to-end test has a stable ID `<KIND>-<slug>[.<section>]` (`KIND ∈ {GRUND, FS, AR, DF, DA}`), cited with the marker `§` — e.g. `<§>FS-user-login.3.1`, where the ID is a shape illustration rather than a real one in this repo, which is what the `<§>` escape says. Type `$$` in a grund-aware editor and it becomes `§`. Bare ID-shaped tokens are ignored — `[reference] strict = true` is set in `grund.toml`, so only `§`-prefixed citations are checked.

### Grounding from a citation

A `§<ID>` is a pointer to a fact, not a file path. Resolve it with `grund` and climb only as far as needed:

- `grund <ID>` — the lead (heading-less, cut at the first child section). The cheap first read for a bare `§<ID>` citation.
- `grund <ID> --toc` — the lead plus the nested section map. Use to choose which subsection to fetch next.
- `grund <ID> --full` — the entire body. Escalate to this when narrower reads aren't enough.
- `grund <ID> --brief` — heading + first paragraph only.
- `grund refs <ID>` — every site that cites the ID; add `--summary` for one line per file. Run before renaming or moving a declaration.
- `grund list` / `grund list --kind FS,AR` — discover IDs if you get lost
- `grund list --size=words --top 10` — find heavy leads and move detail into citable child points before reading them in full

### Project map

- [GRUND](docs/grund.md): Why: project motivation
- [FS](docs/functional-spec): What: behavior, requirements, and constraints
- [AR](docs/architecture): How: high-level implementation, structure, and design
- [DF](docs/decisions/functional): Product behavior decisions and tradeoffs
- [DA](docs/decisions/architectural): Architecture decisions and tradeoffs
- [tests/e2e/](tests/e2e): User scenarios: black-box proof of the spec
- [tests/integration/](tests/integration): Integration tests: proof that the parts fit as designed

### Project namespaces

A namespace is a project boundary, not a docs folder. The current project is the local namespace: cite its IDs as `§<ID>`.

Create or use a separate namespace when work introduces an independently checked app, package, service, or subproject. Give that project its own `grund.toml`, add it to the workspace root's `[workspace] members`, run `grund init` there, and set a stable `project_name`.

Do not create a namespace for a regular module or component that still belongs to this project. Cite across namespaces as `§alias/<ID>` and run `grund check` from the workspace root.

### Declarations and citations

Declarations are heading lines `# <ID>: …` in markdown. In a code doc-comment (Rustdoc, Javadoc, JSDoc, Python docstring, Go `//`, …) drop the `#` — write `/// <ID>: …` directly. Numbered headings inside a declaration are citable sections: use depth-matching headings (`## 1. …`, `### 1.1 …`, etc.) so `§<ID>.1` / `§<ID>.1.1` resolve; mismatched heading depth is a `grund check` error. Every non-declaration heading inside a Markdown declaration body must carry a numbered or enabled named section path; file titles, headings that close the body, fenced examples, non-ATX text, and source doc-comments stay exempt, while bold labels remain the non-citable alternative. One doc-comment may declare multiple IDs (e.g. an `AR-` and an `FS-` on the same class) — each gets its own body. An inline source declaration is reachable from the configured kind home via a one-line stub: `# <ID>: [<path>](<path>)`.

### Rules

- **Spec first.** For behavior or design changes, write or update the most-specific spec point before code.
- **Cite as you write.** Place `§<ID>` at the point a claim or behavior is made — on the doc-comment for a whole behavior, inline beside the clause it enforces.
- **Marker = live citation.** A `§`-prefixed token resolves and is checked wherever it appears — including inside Markdown backticks. To mention an ID without citing it, write `<§><ID>`, omit the marker, or use a fenced code block.
- **Inline citation style.** Inline notes: ≤ 1 line preferred, hard cap 3 lines; ≤ 100 columns. A note is one comment block: a blank line splits it, an empty comment line does not. Doc-comments (`///`, `//!`, `/** */`, a docstring, a Go, Ruby, shell or SQL comment right above a definition) are documentation, not notes: they are never measured, so cite in-sentence there.
- **Always cite the most-specific point.**

### Citation directions

`must`/`never` are `grund check` errors; `should`/`avoid` are suggestions (`grund check --suggestions`). Every source file must cite a declared ID or declare one inline; every file under tests/e2e/ and tests/integration/ must cite one.

- Each **FS** declaration should cite GRUND or FS; never cite AR.
- Each **AR** declaration should cite FS.
- Each **DF** declaration should cite FS or GRUND.
- Each **DA** declaration should cite AR, FS, or GRUND.
- Each file in **tests/e2e/** must cite FS; avoid citing AR.
- Each file in **tests/integration/** should cite AR.
- Each source file outside the Project map (**code**) that cites anything should cite FS or AR.
Anything not listed above is allowed.

### Clickable citations

On repository web surfaces, link `§<ID>` to the PR branch in PR bodies, the reviewed commit in reviews, an exact commit for permalinks, and the default branch otherwise; fall back to plain when unsure.
<!-- END GRUND MANAGED BLOCK -->
