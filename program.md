# kgateway tooling program

**Last updated:** 2026-11-04

Three projects build kgateway operator tooling. Each has its own roadmap and agent sessions. This file is the source of truth for what crosses repo boundaries: exported contracts, dependency edges, order of work and program-level decisions. Per-repo detail stays in each repo.

## Components

| Component | Repo | Role | Workflow |
|---|---|---|---|
| krp | [DuncanDoyle/krp](https://github.com/DuncanDoyle/krp) | Envoy data-plane library + reference CLI/TUI | krp phase commands (`docs/phase-status.md`, `docs/plans/`) |
| kgateway-ui | [DuncanDoyle/kgateway-ui](https://github.com/DuncanDoyle/kgateway-ui) | Web UI; Go backend whose `pkg/` is a versioned library | agentic-engineering-sdlc (`docs/phase-status.md`, `docs/phases/`) |
| CLI (working name `kgwctl`) | no GitHub repo yet (roadmap drafted locally) | kgateway CLI; integrates krp and kgateway-ui libraries | agentic-engineering-sdlc |

**Final home:** the CLI moves into [kgateway-dev/kgateway](https://github.com/kgateway-dev/kgateway). The UI's home is open. Libraries may live outside kgateway until a consumer moves.

## Contracts

| Provider | Packages | Status | Version |
|---|---|---|---|
| krp | `pkg/envoy/model`, `parser`, `filter`, `source` | planned, krp Phase 4 | `v0.1.0` at Phase 4 close |
| kgateway-ui | `pkg/cache`, `engine`, `objects`, `kinds`, `computed` | exists; versioned API (UI D-003) | untagged |
| kgateway-ui | analyzer package | planned, UI sub-phase 3.3 | tbd |

`model`, `parser` and `filter` must not import client-go, lipgloss or bubbletea, so the UI can import them under its D-016 import rules. Fetching lives in `source`.

## Dependency edges

| ID | Consumer | Needs | Provider deliverable | Status |
|---|---|---|---|---|
| E1 | CLI Phase 2 | `proxy routes` (krp view in the CLI) | krp Phase 4 library, `v0.1.0` | open |
| E2 | CLI Phase 1 | `get` / `describe` on live cluster | UI `pkg/cache` + `engine` used one-shot | available, with caveats below (confirmed by kgateway-ui 2026-11-04) |
| E3 | CLI Phase 3 | `check` / `analyze -f` | UI 3.3 analyzer exported under `pkg/` | open — accepted for 3.3 planning |
| E4 | UI Phase 9 | runtime filter chain tab | krp Phase 4 (+5 mapping, +6 intended vs accepted) | open |
| E5 | UI, CLI | Envoy name → Gateway API ref mapping | krp Phase 5 `filter` (see P-001) | open |
| E6 | CLI move into kgateway | license-clean, vendor-neutral libraries | krp Apache-2.0 license (Phase 4); vendor policy plugins outside core (P-002) | open |
| E7 | CLI Phase 4 | attached and effective policy in `describe` | UI Phase 5 policy model in `pkg/` | open |

**E2 one-shot use:**
1. `cache.New(cfg)`, then `Start(ctx)`, which does not block. Wait for `Settled()` or the first `Config.Changes` notify. Then `Snapshot()`, `engine.New(cfg).Compute()`.
2. "Settled" also means a List/Watch failed. Report the snapshot's per-kind access errors, or a forbidden kind looks empty.
3. Proxy stats, xDS rejects and process health are scraped on a timer. A load-once caller sees them as unknown unless it waits for a scrape. Gateway API state is complete.
4. The UI builds `cache.Config` / `engine.Config` in its `cmd/`. No exported helper yet; the CLI builds its own, or asks for one via a `cross-repo` issue.

## Order of work

| Step | krp | CLI | kgateway-ui |
|---|---|---|---|
| 1 | Phase 4 library extraction → `v0.1.0` | roadmap, Phase 1 design | 3.2 |
| 2 | Phase 5 data-plane accuracy | Phase 1: scaffold, `get`/`describe` | 3.3 analyzer |
| 3 | Phase 6 intended vs accepted | Phase 2: `proxy routes`; Phase 3: `check`, `analyze -f` | 3.4, then Phase 4+ |
| later | — | Phase 4 policies (after UI Phase 5); Phase 5 move into kgateway | Phase 9 consumes krp |

krp Phase 4 is the only hard blocker today.

## Program decisions

- **P-001** krp owns the mapping from Envoy names to Gateway API refs. The UI's `computed.EnvoyNamer` and the CLI call it. Keep it swappable, so a CLI inside kgateway can use kgateway's own naming code.
- **P-002** Vendor-specific policy support (beyond kgateway OSS) is a plugin outside library cores. No vendor or edition names in any library.
- **P-003** The CLI is built from scratch. [kgateway#13654](https://github.com/kgateway-dev/kgateway/pull/13654) and earlier prototypes are idea sources only: the relation graph, label-based policy discovery, `check`, `analyze -f`. No gwctl fork, no go-graphviz.
- **P-004** The CLI imports the libraries and talks to the cluster directly. It does not need the UI backend. A thin-client mode against the UI API is optional, later.
- **P-005** Get kgateway maintainer buy-in on the CLI early with a short design proposal, before a large PR.

## Coordination protocol

1. **Phase start (every repo):** read this file. Check open `cross-repo` issues in your repo and in the repos you wait on.
2. **Requests between repos:** a GitHub issue in the provider repo, label `cross-repo`, title prefix `[from <repo>]`, body names the edge ID.
3. **Contracts are tags.** Consumers require a tagged version. For co-development, use an uncommitted `go.work` in the parent directory (see `go.work.example`).
4. **Phase close:** update the edge status and contract table here. Propose a release tag if a contract changed. Notify each consumer with a `cross-repo` issue in its repo naming the edge and version.
5. **Live pings:** local sessions can message each other (`SendMessage`; find names with `ListAgents`, named after the repo directory). Use it for "tag published" or "issue filed" only. The record lives in this file and in issues.
6. **Changes to this file** are made by the session whose phase changed, in a small commit of its own.
