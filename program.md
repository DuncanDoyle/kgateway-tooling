# kgateway tooling program

**Last updated:** 2026-11-05

Three projects build kgateway operator tooling. Each has its own roadmap and agent sessions. This file is the source of truth for what crosses repo boundaries: exported contracts, dependency edges, order of work and program-level decisions. Per-repo detail stays in each repo.

## Components

| Component | Repo | Role | Workflow |
|---|---|---|---|
| xdsview (was krp) | [DuncanDoyle/xdsview](https://github.com/DuncanDoyle/xdsview) | Envoy data-plane library `xdsview` + reference CLI/TUI `krp` | agentic-engineering-sdlc (`docs/phase-status.md`, `docs/phases/`) |
| kgateway-ui | [DuncanDoyle/kgateway-ui](https://github.com/DuncanDoyle/kgateway-ui) | Web UI; Go backend whose `pkg/` is a versioned library | agentic-engineering-sdlc (`docs/phase-status.md`, `docs/phases/`) |
| CLI (working name `kgwctl`) | [DuncanDoyle/kgwctl](https://github.com/DuncanDoyle/kgwctl) | kgateway CLI; integrates krp and kgateway-ui libraries | agentic-engineering-sdlc |

**Final home:** the CLI moves into [kgateway-dev/kgateway](https://github.com/kgateway-dev/kgateway). The UI's home is open. Libraries may live outside kgateway until a consumer moves.

## Contracts

| Provider | Packages | Status | Version |
|---|---|---|---|
| krp | `pkg/envoy/model`, `parser`, `filter`, `source`, `source/kube`, `origin`; `krp dump -o json` (`v1alpha1`) | delivered, krp Phases 4–5 | `v0.3.0` (module `github.com/DuncanDoyle/xdsview`) |
| kgateway-ui | `pkg/cache`, `engine`, `objects`, `kinds`, `computed` | exists; versioned API (UI D-003) | untagged |
| kgateway-ui | analyzer package | planned, UI sub-phase 3.3 | tbd |

`model`, `parser` and `filter` must not import client-go, lipgloss or bubbletea, so the UI can import them under its D-016 import rules. Fetching lives in `source`; port-forwarding, the only client-go user, is in `source/kube`.

## Dependency edges

| ID | Consumer | Needs | Provider deliverable | Status |
|---|---|---|---|---|
| E1 | CLI Phase 3 | `proxy routes` (krp view in the CLI) | krp Phase 4 library, `v0.1.0` | available — krp `v0.1.0` (2026-11-04); notified kgwctl#1 |
| E2 | CLI Phase 1 | `get` / `describe` on live cluster | UI `pkg/cache` + `engine` used one-shot | available, with caveats below (confirmed by kgateway-ui 2026-11-04) |
| E3 | CLI Phase 5 | full `check` analysis, `analyze -f` | UI 3.3 analyzer exported under `pkg/` | open — accepted for 3.3 planning |
| E4 | UI Phase 9 | runtime filter chain tab | krp Phase 4 (+5 mapping, +6 intended vs accepted) | mostly available — krp `v0.2.0` (model, mapping, `source/kube` cancellation); intended vs accepted is krp Phase 6; notified kgateway-ui#21 |
| E5 | UI, CLI | Envoy name → Gateway API ref mapping | krp Phase 5 `origin` (see P-001) | available — krp `v0.2.0` (2026-11-05); notified kgwctl#2, kgateway-ui#21 |
| E6 | CLI move into kgateway | license-clean, vendor-neutral libraries | krp Apache-2.0 license (Phase 4); vendor policy plugins outside core (P-002) | open — license in krp `v0.1.0`; P-002 seam (`origin.Chain`) in `v0.2.0`; krp's `testdata/enterprise/` still to remove |
| E7 | CLI Phase 6 | attached and effective policy in `describe` | UI Phase 5 policy model in `pkg/` | open |
| E8 | CLI Phase 2 | `check` MVP | UI health checks (KGWUI-001 onward) run one-shot via `engine.Compute`, with controller metrics from one synchronous scrape | available; convenience helpers requested (see below) — confirmed by kgateway-ui 2026-11-04 |

**xdsview `v0.3.0` use (E1, E4, E5):** import `github.com/DuncanDoyle/xdsview/pkg/envoy/...` (renamed from `DuncanDoyle/krp` in `v0.3.0`, D-007; API unchanged).
1. `source.File`, `source.Admin` or `kube.Gateway`/`kube.Deployment` → `ConfigDump(ctx)` → `parser.Parse` → `filter.Filter`. Close the reader to stop a port-forward.
2. `kube.Options.RESTConfig` takes the caller's `*rest.Config`. client-go v0.36.2.
3. The JSON contract is `krp dump -o json`; golden files in krp `testdata/golden/`.
4. `source/kube` cancellation is fixed ([krp#25](https://github.com/DuncanDoyle/xdsview/issues/25)), except through an HTTP/SOCKS proxy in front of the API server ([krp#26](https://github.com/DuncanDoyle/xdsview/issues/26), not a requirement).
5. Mapping (E5): `origin.Annotate(snap, origin.Chain(plugin, origin.Kgateway(opts)))`. Every answer carries a certainty: exact (metadata), inferred, ambiguous, unknown. Route names usually read several ways; pass the HTTPRoutes you know as `KgatewayOptions.Known` (`kube.HTTPRoutes` lists them, since `v0.2.1`), or run kgateway with `enableRouteSourceMetadata`.
6. UI `EnvoyNamer.Object(name)`: `origin.ObjectForName(mapper, name)` maps cluster and route-config names from proxy stats. It needs `KgatewayOptions.Gateway` for route configs.

**E2 one-shot use:**
1. `cache.New(cfg)`, then `Start(ctx)`, which does not block. Wait for `Settled()` or the first `Config.Changes` notify. Then `Snapshot()`, `engine.New(cfg).Compute()`.
2. "Settled" also means a List/Watch failed. Report the snapshot's per-kind access errors, or a forbidden kind looks empty.
3. Proxy stats, xDS rejects and process health are scraped on a timer. A load-once caller sees them as unknown unless it waits for a scrape. Gateway API state is complete.
4. The UI builds `cache.Config` / `engine.Config` in its `cmd/`. No exported helper yet; the CLI builds its own, or asks for one via a `cross-repo` issue.

**E8 `check` MVP:**
1. Run checks through `engine.New(cfg).Compute()` over a settled cache; it calls `health.Evaluate` per install. Calling `Evaluate` directly needs many derived inputs the engine builds.
2. Controller metrics: `ctrlmetrics.NewCollector(PodProxyScraper{...}).Collect(ctx, install)` scrapes once, synchronously. `cfg.Metrics` takes the exported `ctrlmetrics.Source` interface (one method), so the CLI can wrap that Result itself. Not blocking; a shared static source and config builder are requested in a `cross-repo` issue.
3. Time-based checks (restart hold, grace periods, metric staleness) are pending on a single pass. Report them as not yet known, or wait until `Report.Next`.
4. Key output on finding code and `Finding.id`, both stable (UI D-028, D-040). Severity belongs to each finding, not to a code.

## Order of work

| Step | krp | CLI | kgateway-ui |
|---|---|---|---|
| 1 | Phase 4 library extraction → `v0.1.0` | roadmap, Phase 1 design | 3.2 |
| 2 | Phase 5 data-plane accuracy | Phase 1: scaffold, `get`/`describe`; Phase 2: `check` MVP | 3.3 analyzer |
| 3 | Phase 6 intended vs accepted | Phase 3: `proxy routes`; Phase 4: support bundle; Phase 5: full analysis, `analyze -f` | 3.4, then Phase 4+ |
| later | — | Phase 6 policies (after UI Phase 5); Phase 7 move into kgateway | Phase 9 consumes krp |

krp delivered `v0.1.0` (Phase 4, 2026-11-04) and `v0.2.0` (Phase 5, 2026-11-05). No hard blockers today.

## Program decisions

- **P-001** krp owns the mapping from Envoy names to Gateway API refs. The UI's `computed.EnvoyNamer` and the CLI call it. Keep it swappable, so a CLI inside kgateway can use kgateway's own naming code.
- **P-002** Vendor-specific policy support (beyond kgateway OSS) is a plugin outside library cores. No vendor or edition names in any library.
- **P-003** The CLI is built from scratch. [kgateway#13654](https://github.com/kgateway-dev/kgateway/pull/13654) and earlier prototypes are idea sources only: the relation graph, label-based policy discovery, `check`, `analyze -f`. No gwctl fork, no go-graphviz.
- **P-004** The CLI imports the libraries and talks to the cluster directly. It does not need the UI backend. A thin-client mode against the UI API is optional, later.
- **P-005** ~~Get kgateway maintainer buy-in on the CLI early with a short design proposal, before a large PR.~~ Deferred 2026-11-05 (kgwctl D-011): the owner leads both the product and most maintainers; revisit at the CLI's upstream move.

## Coordination protocol

1. **Phase start (every repo):** read this file. Check open `cross-repo` issues in your repo and in the repos you wait on.
2. **Requests between repos:** a GitHub issue in the provider repo, label `cross-repo`, title prefix `[from <repo>]`, body names the edge ID.
3. **Contracts are tags.** Consumers require a tagged version. For co-development, use an uncommitted `go.work` in the parent directory (see `go.work.example`).
4. **Phase close:** update the edge status and contract table here. Propose a release tag if a contract changed. Notify each consumer with a `cross-repo` issue in its repo naming the edge and version.
5. **Live pings:** local sessions can message each other (`SendMessage`; find names with `ListAgents`, named after the repo directory). Use it for "tag published" or "issue filed" only. The record lives in this file and in issues.
6. **Changes to this file** are made by the session whose phase changed, in a small commit of its own.
