# Security Policy

## Reporting a vulnerability

**Please do not open a public issue for a security vulnerability.**

Report it privately through either channel:

- [GitHub private vulnerability reporting](https://github.com/wingfoil-io/wingfoil/security/advisories/new)
  — preferred, and keeps the report attached to the repository.
- Email `hello@wingfoil.io` with `SECURITY` in the subject line.

Please include enough to reproduce: the version (or commit), the feature flags
enabled, the adapter involved if any, and a minimal graph or test that shows
the problem.

We aim to acknowledge a report within three working days, and to keep you
updated as we work through it. When a fix ships we will credit you in the
advisory unless you would rather we didn't.

## Supported versions

Wingfoil has not yet reached a long-term-support release. Fixes land on the
latest published version; there is no backporting to earlier majors. See the
[releases](https://github.com/wingfoil-io/wingfoil/releases) for what is
current.

## Scope

In scope:

- The engine and runtime (`crates/wingfoil`), including anything reachable
  from a graph built with untrusted input.
- The I/O adapters — in particular the ones parsing bytes off a network
  (`fix`, `web`, `aeron`, `zmq`, `kafka`, `redis`, `etcd`) or reading files
  from disk (`csv`, `lines`, `kdb`).
- The Python bindings (`crates/wingfoil-python`) and the WASM/TypeScript
  client, where a memory-safety or sandbox-escape issue would cross a
  language boundary.

Out of scope:

- Vulnerabilities in third-party services the adapters talk to — report those
  upstream.
- Denial of service achieved only by configuring a graph to consume unbounded
  resources. Wingfoil runs the graph you wire; it is not a sandbox for
  untrusted graph definitions.
- Findings from automated scanners with no demonstrated impact.

## Dependency vulnerabilities

Dependency advisories are caught two ways, and the difference matters:

- [`security-audit.yml`](.github/workflows/security-audit.yml) **fails CI** on a
  dependency with a known advisory — `cargo audit` against RustSec for both
  Cargo workspaces, `pnpm audit` for `js/`, and `dependency-review` to block a
  pull request that *introduces* a vulnerable dep. It also runs weekly, so an
  advisory disclosed against an already-pinned dependency surfaces without a
  code change.
- **Dependabot security updates** open the upgrade PRs, automatically, against
  the GitHub Advisory Database. These are a repository setting rather than a
  `dependabot.yml` entry, so they cover every ecosystem Dependabot can parse.

  Dependabot **version updates** — routine bumps of dependencies with no
  advisory against them — are deliberately **not** enabled. Routine bumps are
  [Renovate](.github/renovate.json)'s job instead: weekly, grouped, and
  `rangeStrategy: "bump"` so it raises the *manifest floor* rather than only
  the lock. Running one updater rather than two is the point; two would race
  each other on the same manifests.

## Release cooldown

Routine updating is a different trade to the advisory gate above. Staying at
the tip of every dependency shortens the distance to a future security fix,
but it also puts this repository in the first wave to install any newly
published release — exactly the population a compromised-maintainer attack
targets. Neither `cargo audit` nor `pnpm audit` defends against that: they
match against advisory databases, and a freshly malicious release has no
advisory yet by construction.

Age is the defence that does work there, because such releases are typically
yanked within hours. So every routine update waits **seven days** after
publication:

- `minimumReleaseAge` in [`.github/renovate.json`](.github/renovate.json)
  covers everything Renovate opens a PR for, Cargo and npm alike.
- `minimum-release-age` in [`js/.npmrc`](js/.npmrc) covers the manual path —
  `pnpm update`, `pnpm add` — which Renovate never sees. It does not affect
  `pnpm install --frozen-lockfile`, so CI is unchanged. Needs pnpm >= 10.16.

The cooldown is scoped to releases that have **no** advisory. A fix for one
that is already public must never wait: `vulnerabilityAlerts` in the Renovate
config sets `minimumReleaseAge: null` and `schedule: at any time`, and
Dependabot security updates are not rate-limited at all.

Manual upgrades are still expected to be deliberate — `cargo update` /
`pnpm update` when there is a reason to, and read what moved.

## Reviewed-dependency audits (cargo-vet)

`cargo audit` asks "is there an advisory against this?". [`cargo
vet`](https://mozilla.github.io/cargo-vet/) asks the complementary question:
"has a human read this crate version's code?" — which is the only one of the
two that can catch a malicious release on the day it ships.

It is answered mostly with other people's review work. `supply-chain/config.toml`
imports the audit sets published by Mozilla, Google, the Bytecode Alliance,
ZCash, Embark, ISRG and Fermyon; `supply-chain/imports.lock` pins what those
sets said, so a CI run cannot be changed by someone else editing their audits
file. Refresh it deliberately by running `cargo vet` and committing the result.

At adoption this tree was 140 crates fully audited against 603 **exemptions**.
The exemptions are the pre-existing graph, grandfathered in by `cargo vet init`
so the gate starts green — they are not a claim that anything was reviewed.
The value is the ratchet on what arrives *next*: a new dependency, or a bump to
a version nobody has audited, surfaces as a diff to `supply-chain/` instead of
sliding in unremarked. Since `cargo vet` certifies *deltas*, a later bump of an
already-certified crate only costs a review of the diff.

The [`cargo vet` job](.github/workflows/security-audit.yml) is **non-blocking**
(`continue-on-error`) while we learn how well the import coverage holds across
a few Renovate cycles. Drop that line to make it a gate.

## Pinning GitHub Actions

Every `uses:` in [`.github/workflows`](.github/workflows) is pinned to a full
**commit SHA**, with the human-readable tag kept as a trailing comment:

```yaml
uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7
```

A tag is mutable and a branch more so, so `@v4` is a promise the upstream owner
can rewrite after the fact — the vector `tj-actions/changed-files` was used for.
A SHA is not rewritable. This applies to *all* workflows, not only the ones
holding credentials: a low-privilege workflow that can be made to lie is its own
problem, and `security-audit.yml` reporting a false green is the clearest case.

Two consequences worth knowing before you edit a workflow:

- **Actions that infer behaviour from the ref name need an explicit input once
  pinned.** `taiki-e/install-action@nextest` picks its tool from the tag, so
  pinned call sites must pass `tool: nextest`. (`dtolnay/rust-toolchain` is
  safe — its `toolchain` input defaults to `stable` independently of the ref.)
- **Renovate maintains the pins** via `pinDigests` in
  [`.github/renovate.json`](.github/renovate.json), updating the SHA and the
  `# vX` comment together. Do not replace a SHA with a floating tag to "make
  updates easier"; that is the thing being prevented.

You are welcome to open a normal public issue for a dependency advisory — they
are already public by definition.
