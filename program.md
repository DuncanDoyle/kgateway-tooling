# kgateway tooling program

**Last updated:** 2026-11-27

Three projects build kgateway operator tooling. Each has its own roadmap and agent sessions. This file is the source of truth for what crosses repo boundaries: exported contracts, dependency edges, order of work and program-level decisions. Per-repo detail stays in each repo.

## Components

| Component | Repo | Role | Workflow |
|---|---|---|---|
| xdsview (was krp) | [DuncanDoyle/xdsview](https://github.com/DuncanDoyle/xdsview) | Envoy data-plane library `xdsview` + reference CLI/TUI `krp` | agentic-engineering-sdlc (`docs/phase-status.md`, `docs/phases/`) |
| Snellius (formerly kgateway-ui) | [DuncanDoyle/snellius](https://github.com/DuncanDoyle/snellius) | Web UI; Go backend whose `pkg/` is a versioned library (module `github.com/DuncanDoyle/snellius` since `v0.5.0`) | agentic-engineering-sdlc (`docs/phase-status.md`, `docs/phases/`) |
| CLI (working name `kgwctl`) | [DuncanDoyle/kgwctl](https://github.com/DuncanDoyle/kgwctl) | kgateway CLI; integrates krp and Snellius libraries | agentic-engineering-sdlc |
| snellius.dev | [DuncanDoyle/snellius.dev](https://github.com/DuncanDoyle/snellius.dev) | Snellius website and user docs (Astro + Starlight, deployed by Cloudflare from `main`) | its own session owns the whole site, SEO included (2026-11-20); follows Snellius releases (P-006) |
| snellius-demo | [DuncanDoyle/snellius-demo](https://github.com/DuncanDoyle/snellius-demo) | Installs kgateway, Snellius and sample use cases on any cluster, optionally with Keycloak and secure mode; walkthrough scenarios, each with its verified expected findings (P-007) | installs the latest Snellius chart by default; check its README at each Snellius release |

**Final home:** the CLI moves into [kgateway-dev/kgateway](https://github.com/kgateway-dev/kgateway). The UI's home is open. Libraries may live outside kgateway until a consumer moves.

## Contracts

| Provider | Packages | Status | Version |
|---|---|---|---|
| krp | `pkg/envoy/model`, `parser`, `filter`, `source`, `source/kube`, `origin`, `chain`, `render`; `krp dump -o json` (`v1alpha1`) | delivered, krp Phases 4–6, 8, 9 | `v0.7.0` (module `github.com/DuncanDoyle/xdsview`); `v0.7.0` (2026-11-26): the E15 contract break, see E15; `v0.6.0` (2026-11-23): vendor-neutral core (E6), Go API unchanged from `v0.5.0`, OSS output byte-identical; `v0.5.0` (2026-11-22) adds certainty to every attribution, additive in Go and JSON (E13); one Go break since v0.3: `model.WeightedCluster` is not comparable (D-009). v0.4.1 (2026-11-21): attribution fixes #28, RouteExtra D-010 |
| Snellius | `pkg/cache`, `engine`, `objects`, `kinds`, `computed`, `setup`, `ctrlmetrics` | delivered, UI Phases 1–5; versioned API (UI D-003). From `v0.6.0`, controller metrics reach the engine only as a collected value: `ctrlmetrics.CollectOnce` + `Publish` into `setup.Config.Collected` (UI Phase 5 design §7); `computed.Rules` loses `Delegation` (optional `computed.Delegator`); `extension.NewCore` and packs are additive. `v0.7.0` adds `cache.ListOnce` and `cache.Config.Namespaces` (UI Phase 6, D-076). `v0.8.0` adds `explore.Index.RouteSignals` (configuration and traffic path per route; UI Phase 7, D-087). `v0.9.0` adds `pkg/policy` (E7, UI Phase 8) and `install.Scope.Excluded` (E2), additive | `v0.9.0` (2026-11-26); `v0.8.1` and `v0.8.2` change only the web UI, no contract |
| Snellius | analyzer: `pkg/health` (codes 001–026), `pkg/diagnose` (groups, `Fix`) | delivered, UI sub-phases 3.3–3.5 (023, 024 in 3.4) | `v0.9.0`: a rejected policy raises KGWUI-002 on every supported minor (UI D-135), DirectResponse ancestors are judged (UI D-129), policy readings come from the pack's `PolicyInterpreter`; codes unchanged |

`model`, `parser` and `filter` must not import client-go, lipgloss or bubbletea, so the UI can import them under its D-016 import rules. Fetching lives in `source`; port-forwarding, the only client-go user, is in `source/kube`.

## Dependency edges

| ID | Consumer | Needs | Provider deliverable | Status |
|---|---|---|---|---|
| E1 | CLI Phase 3 | `proxy routes` (krp view in the CLI) | krp Phase 4 library, `v0.1.0` | done — krp `v0.1.0` (2026-11-04); notified kgwctl#1; consumed by CLI Phase 3 on xdsview `v0.4.3`, complete 2026-11-21 |
| E2 | CLI Phase 1 | `get` / `describe` on live cluster | UI `pkg/cache` + `engine` used one-shot | done — kgateway-ui `v0.3.0` (2026-11-05, kgateway-ui#16); consumed by CLI Phase 1, complete 2026-11-06. `v0.4.0` (2026-11-07): list-only kinds (kgwctl#17) and `cache.Wait` (kgwctl#20). `v0.5.0` (2026-11-08): module path `github.com/DuncanDoyle/snellius`. `v0.6.0` (2026-11-09): packs and `extension.NewCore`, additive; `extension.New` unchanged. `v0.7.0` (2026-11-12): `cache.ListOnce`, a list-only snapshot (R2, snellius#18), and `cache.Config.Namespaces` for namespace-scoped RBAC (R3, snellius#19), additive; notified kgwctl#8. `v0.8.0` (2026-11-19): two signals per route through `explore.Index.RouteSignals`, additive; a caller without computed views gets `path: unknown, no-path` for an accepted route without a fault; notified kgwctl#9. `v0.9.0` (2026-11-26): `install.Scope.Excluded`, the namespace names outside a scope, `known=false` when the scope is unknown (mirror M-4, snellius#66), additive; notified kgwctl#16 |
| E3 | CLI Phase 5 | full `check` analysis, `analyze -f` | UI 3.3 analyzer exported under `pkg/` | available — kgateway-ui `v0.4.0` (2026-11-07): the analyzer over a cache snapshot, with 023 and 024; `analyze -f` (snapshot from files) is not in UI Phase 3 (UI R79). `v0.5.0` (2026-11-08): module path `github.com/DuncanDoyle/snellius`; 3.5 fixes. `v0.6.0` (2026-11-09): pack checks run only on their installs; kgateway findings unchanged |
| E4 | UI Phase 12 | runtime filter chain tab | krp Phase 4 (+5 mapping, +6 runtime filter chain) ~~+7 intended vs accepted~~ | available — krp `v0.2.0` (model, mapping, `source/kube` cancellation); xdsview `v0.4.0` (2026-11-21) adds the runtime filter chain per route and listener state; notified kgateway-ui#21, snellius#63. Intended vs accepted is withdrawn (P-009, 2026-11-22; xdsview D-011): no provider. Policy labels on the chain need E13 first |
| E5 | UI, CLI | Envoy name → Gateway API ref mapping | krp Phase 5 `origin` (see P-001) | available — krp `v0.2.0` (2026-11-05); notified kgwctl#2, kgateway-ui#21 |
| E6 | CLI move into kgateway | license-clean, vendor-neutral libraries | krp Apache-2.0 license (Phase 4); vendor policy plugins outside core (P-002) | available — xdsview `v0.6.0` (2026-11-23, Phase 8; xdsview#32 closed; notified kgwctl#12): license since `v0.1.0`; enterprise corpus out (D-013, D-018); policy links only for kgateway TrafficPolicy, transformation only for the OSS module; Go API unchanged, OSS output byte-identical. kgwctl's move no longer waits on xdsview. Three Phase 8 review findings (P1, none in the OSS corpus) are deferred by the owner to the Phase 9 design (E15; xdsview `phase-8-complete.md` Waivers) |
| E7 | CLI Phase 6 | policies in `describe`, each labelled with its source (P-008): declared (targetRefs as written), controller-reported attached (ancestor conditions as the controller reports them), and, for a route, the policy IDs in that route's `merge.*` metadata | Snellius Phase 8 declared-policy model in `pkg/`; xdsview `origin.Policies` for the route-level xDS part (E5, E13). Listener, RouteConfiguration and virtual-host records come with E15 (xdsview Phase 9) | Snellius part delivered: `pkg/policy` in Snellius `v0.9.0` (2026-11-26; Snellius D-123, D-130, D-131, D-133, D-138; notified kgwctl#16). Route-level and scoped `merge.*` records come from xdsview (E5, E13, E15). Redefined 2026-11-22 (P-008): was "attached and effective policy". `merge.*` is level-local: it does not say which level governs a request. Anything beyond controller-reported evidence stays out of scope, also after U1/U4 land |
| E8 | CLI Phase 2 | `check` MVP | UI health checks (KGWUI-001 onward) run one-shot via `engine.Compute`, with controller metrics from one synchronous scrape | done — consumed by CLI Phase 2 on Snellius `v0.8.2`, complete 2026-11-20 (kgwctl#5–#7 closed). kgateway-ui `v0.3.0` (2026-11-05): `pkg/setup` and `ctrlmetrics.Static` (UI D-053, kgateway-ui#13); `v0.4.0` adds `setup.Options.NoEndpoints`; `v0.5.0` (2026-11-08) renames the module to `github.com/DuncanDoyle/snellius` and drops the status-sync count from `ctrlmetrics`. Breaks in Snellius `v0.6.0` (UI Phase 5, on `main` from 2026-11-09): `ctrlmetrics.Static`, `ctrlmetrics.Source` and `cfg.Engine.Metrics` are gone; use `ctrlmetrics.CollectOnce` + `Publish` into `setup.Config.Collected` (kgwctl#6) |
| E9 | CLI Phase 2 `check --live` | what a check is waiting on before it decides (holds) | Snellius 5.7: optional `hold` {`until` RFC 3339, `count` in a per-check unit, `reason` for display} on each check result; `ok` and `pending` unchanged | done — consumed by CLI Phase 2 `check --live`, complete 2026-11-20. Delivered Snellius `v0.7.0` (2026-11-12); first in `v0.7.0-alpha.1` (2026-11-10, pre-release; [snellius#33](https://github.com/DuncanDoyle/snellius/issues/33), Snellius D-068) |
| E10 | a downstream CLI build on kgwctl `pkg/cli` (local MCP, stdio; kgwctl D-023) | one read-only MCP tool set shared with Snellius's remote MCP, fed from a one-shot load with the user's kubeconfig | Snellius MCP tool set under `pkg/` (Snellius D-089); Snellius UI Phase 7 sets its visibility and audit contract | planned — UI Phase 7 shipped the visibility and audit contract in `v0.8.0` (2026-11-19, design §9–§10); the tool set package is still to come |
| E11 | CLI Phase 3.2 | `proxy routes` tree and filter order per route | xdsview: exported stdlib text renderer `pkg/envoy/render`, `EffectiveChain` per route, listener state (active, warming, draining, error); UI-neutral types in `model` for Snellius Phase 12 (E4) too | done — consumed by CLI Phase 3 on xdsview `v0.4.3`, complete 2026-11-21 (kgwctl#10 closed). xdsview `v0.4.0` (2026-11-21, Phase 6); xdsview#27 closed; notified kgwctl#10. v0.4.1 (2026-11-21): attribution fixes #28, RouteExtra D-010. v0.4.2 (2026-11-21): narrowing edge cases #29. v0.4.3 (2026-11-21): warning for a skipped HCM, narrowing keeps listeners mid-update #31 |
| E12 | a downstream consumer (separate service and web UI) | reuse of Snellius login, authorization and UI building blocks | Snellius: `pkg/auth/oidc` and `pkg/auth/session`; the SubjectAccessReview checker, cache and limiter in `pkg/auth/sar`, split from Snellius's visibility layer; `@snellius/shell` theme tokens and components, published or as a git dependency | deferred (2026-11-21) — nothing is extracted until the consumer's design starts, after Snellius Phase 14 and the remote MCP. Decide then which pieces to extract and version as contracts. Until then, keep `pkg/auth/oidc` and `pkg/auth/session` free of Snellius-specific imports. A generic external-link action on Snellius pages is deferred with it. Reused policy rendering is declared and controller-reported only (P-008) |
| E13 | CLI `proxy routes` (shipped, Phase 3); UI Phase 12; E7 | every attribution states its certainty: a policy contribution is at most `inferred`; a policy field links to a filter only when exactly one filter entry matches; unmapped fields stay listed, not dropped (xdsview#30) | xdsview Phase 8.1 (xdsview D-012, D-014, D-016), released as `v0.5.0`: JSON adds `certainty` (additive), text adds `(inferred)`, consumer goldens change. The 6.5 level tie-break is dropped (xdsview D-016, D-017); no recorded golden changes its contributions, so kgwctl sees labels only: `certainty` and `indexCertainty` in JSON, ` (inferred)` in text | available — xdsview `v0.5.0` (2026-11-22, sub-phase 8.1); notified kgwctl#11. Consumed by kgwctl Phase 4b through xdsview `v0.7.1`, released as kgwctl `v0.1.0` (2026-11-28; kgwctl#11 closed). An exact link when several filters match needs U11 |
| E14 | kgwctl (live diagnostic); Snellius (optional) | xDS convergence per proxy pod and per subscription: `yes`, `no` or `unknown`, each naming the side that proves it | split by the P-009 investigation ([xdsview `263055d`](https://github.com/DuncanDoyle/xdsview/blob/263055d/docs/investigations/xds-convergence.md) §4.1):<ul><li>xdsview core, Phase 9: flag an active HCM whose RDS route configuration is absent from a complete dump (`unknown` when the dump is filtered, `RoutesConfigDump` is missing, or any route config entry was skipped); warming and draining listeners and warming clusters by name (largely exists)</li><li>kgwctl: the live join of controller `/snapshots/xds` and proxy evidence, and the verdict ([kgwctl#13](https://github.com/DuncanDoyle/kgwctl/issues/13)). It extracts only key, type, version and resource names, timestamped per read, and never stores the snapshot</li><li>kgwctl bundles: an offline ceiling only; they cannot answer question 1</li><li>Snellius, optional: the "Config accepted" wording, delivered in `v0.9.1` ([snellius#68](https://github.com/DuncanDoyle/snellius/issues/68)); proxy-only version differences between replicas, reported as a difference, not drift, wait for UI Phase 12</li><li>kgateway upstream: U7, U9, U8</li></ul> | decided 2026-11-24 — the investigation reported, owner-approved. No generic `/snapshots/xds` vs `/config_dump` diff. M-3 not approved; kgwctl answers `unknown` when a Gateway's keys disagree |
| E15 | kgwctl (`proxy routes`, Phase 6 via E7); UI Phase 12 | the Phase 9 contract (P-010): execution and provenance as separate results, claims with their own evidence, results keyed by occurrence, scoped `merge.*` records, explicit module composition, a `NameMapper` for stats names; krp JSON `v1alpha2` | xdsview Phase 9, one tagged release; supersedes the Go usage of E5, E11 and E13 | available — xdsview `v0.7.0` (2026-11-26, Phase 9, `phase-9-complete.md`; notified kgwctl#15, snellius#71): core packages read any Envoy dump; `provenance.Compose(snap, modules...)` returns an `Index` of claims; kgateway knowledge only in `kgateway.New()`; `origin` and `filter` deleted; krp JSON `v1alpha2`. `Index.Dependencies` and `Index.Inputs` are outside P-010 rule 7 until Snellius Phase 12 is designed: a minor release may change them, and its notice says so. Open with the owner: A4 (cross-module filter links) and E4 (sealed assertion kinds). kgwctl migrates `proxy routes` in step, independent of its Phase 6: it builds its own `ProxyRoutes` payload instead of embedding `model`, and bumps the shared envelope to `kgwctl.kgateway.dev/v1alpha2`, which touches every command's JSON and YAML; text, JSON and YAML goldens are reviewed. kgwctl Phase 6 (E7) joins the scoped `merge.*` records with Snellius `pkg/policy` by ref. UI Phase 12 targets this tag, not an earlier one. Consumed by kgwctl Phase 4b on xdsview `v0.7.1` (notified kgwctl#17), released as kgwctl `v0.1.0` (2026-11-28); kgwctl#14, #15 and #17 closed. xdsview `v0.7.2` keeps an explicit index certainty on a tie (notified kgwctl#18); kgwctl `v0.1.1` (2026-11-28) consumes it and dropped its own normalisation (kgwctl D-036); kgwctl#18 closed |

UI phase numbers follow Snellius D-084 (2026-11-14): Phase 7 is Identity & Access, and the old UI Phases 7–13 are now 8–14. References above use the new numbers.

**xdsview `v0.3.0` use (E1, E4, E5):** import `github.com/DuncanDoyle/xdsview/pkg/envoy/...` (renamed from `DuncanDoyle/krp` in `v0.3.0`, D-007; API unchanged).
1. `source.File`, `source.Admin` or `kube.Gateway`/`kube.Deployment` → `ConfigDump(ctx)` → `parser.Parse` → `filter.Filter`. Close the reader to stop a port-forward.
2. `kube.Options.RESTConfig` takes the caller's `*rest.Config`. client-go v0.36.2.
3. The JSON contract is `krp dump -o json`; golden files in krp `testdata/golden/`.
4. `source/kube` cancellation is fixed ([krp#25](https://github.com/DuncanDoyle/xdsview/issues/25)), except through an HTTP/SOCKS proxy in front of the API server ([krp#26](https://github.com/DuncanDoyle/xdsview/issues/26), not a requirement).
5. Mapping (E5): `origin.Annotate(snap, origin.Chain(plugin, origin.Kgateway(opts)))`. Every answer carries a certainty: exact (metadata), inferred, ambiguous, unknown. Route names usually read several ways; pass the HTTPRoutes you know as `KgatewayOptions.Known` (`kube.HTTPRoutes` lists them, since `v0.2.1`), or run kgateway with `enableRouteSourceMetadata`.
6. UI `EnvoyNamer.Object(name)`: `origin.ObjectForName(mapper, name)` maps cluster and route-config names from proxy stats. It needs `KgatewayOptions.Gateway` for route configs.

**xdsview `v0.4.0` use (E11, E4):**
1. `chain.Annotate(snap, mapper)` sets `Route.Chain`: each HCM filter's `Runs`, `State`, deciding `Level`, request/response side, and extension and policy contributions (`origin.PolicyFilters`, optional). `RequestOrder()` / `ResponseOrder()`. A traffic split with per-backend config sets `Conditional`.
2. `render.Text(w, snap, render.Options)` is the plain-text route view (stdlib + `pkg/envoy` core only); `Style` and `Extra` hooks for styled or interactive callers. Text goldens: `testdata/golden/scenarios/*/*.txt`.
3. `Listener.State`: active, warming, draining, rejected, `Updating`, `Draining`, `Error`.
4. Annotate before `filter.Filter` to narrow on resolved origins; `filter.Ambiguous` lists routes left out. `RouteAnnotation.NarrowedByKnown` marks an origin `Known` settled.
5. Go break: `model.WeightedCluster` holds a map and is not comparable (xdsview D-009).

**E2 one-shot use:**
1. `cache.New(cfg)`, then `Start(ctx)`, which does not block. Wait for `Settled()` or the first `Config.Changes` notify. Then `Snapshot()`, `engine.New(cfg).Compute()`.
2. "Settled" also means a List/Watch failed. Report the snapshot's per-kind access errors, or a forbidden kind looks empty.
3. Proxy stats, xDS rejects and process health are scraped on a timer. A load-once caller sees them as unknown unless it waits for a scrape. Gateway API state is complete.
4. `setup.Build` builds `cache.Config` / `engine.Config` the way the UI does (since `v0.3.0`); its Go doc has the one-shot wait.
5. Since `v0.4.0`: DirectResponse, GatewayExtension and ReferenceGrant are list-only (`kinds.Descriptor.ListOnly`), outside cluster access and `Settled`; `cache.Wait()` returns once every goroutine `Start` launched has exited.
6. Since `v0.5.0`: import `github.com/DuncanDoyle/snellius/...`; earlier tags stay under `github.com/DuncanDoyle/kgateway-ui`.
7. Since `v0.7.0`: `cache.ListOnce(ctx, cfg)` replaces steps 1–2 for a caller without `watch`: one LIST per input, no goroutines; a kind it cannot list shows in `Access`. `cfg.Namespaces` reads namespaced inputs per namespace (GatewayClass and Namespace stay cluster-wide); `Access` holds the worst state, `NamespaceAccess` the state per namespace. Checks never judge a namespace that was not read.

**E8 `check` MVP:**
1. Run checks through `engine.New(cfg).Compute()` over a settled cache; it calls `health.Evaluate` per install. Calling `Evaluate` directly needs many derived inputs the engine builds.
2. Controller metrics (since `v0.6.0`): after the wait, take one `Snapshot()`, keep the `install.Discover` installs whose `Impl` is `"kgateway"`, then `ctrlmetrics.Publish(cfg.Collected, ctrlmetrics.CollectOnce(ctx, ctrlmetrics.PodProxyScraper{Kube: kube}, insts, setup.MetricsMaxAge(interval)))`. Set `cfg.Engine.Snapshots` to return that snapshot and call `engine.New(cfg.Engine).Compute()`. Staleness is per entry, from `Results.MaxAge`. Without `Publish`, KGWUI-001 reads "controller metrics not collected yet". The `ctrlmetrics` Go doc has the sequence.
3. Time-based checks (restart hold, grace periods, metric staleness) are pending on a single pass. Report them as not yet known, or wait until `Report.Next`.
4. Key output on finding code and `Finding.id`, both stable (UI D-028, D-040). Severity belongs to each finding, not to a code.

## Order of work

Revised 2026-11-22. The first order (2026-11-04) is in git history.

Done: xdsview Phases 4–6, 8 and 9 (`v0.7.0`, E15); kgwctl Phases 1–4 and 4b (`v0.1.0`, consumes E15 and the Snellius part of E7's release); Snellius Phases 1–8 (`v0.9.0`, E7); Snellius `v0.9.1` (E14 wording).

| Priority | xdsview | kgwctl | Snellius |
|---|---|---|---|
| now | Phase 9 done: `v0.7.0` (E15, 2026-11-26). Open with the owner: A4, E4 | Phase 4b done (2026-11-28): `proxy routes` on E15 (xdsview `v0.7.1`), Snellius `v0.9.0`, envelope `v1alpha2`; first release `v0.1.0`; `v0.1.1` on xdsview `v0.7.2`. Next: Phase 5 when the owner starts it | Phase 8 done: `v0.9.0` (E7, 2026-11-26). Next: snellius#68, then Phase 9 Search & Route Tester when the owner starts it |
| next | additive E15 extensions on a named consumer case (P-010 point 7) | Phase 5 full analysis, `analyze -f`; the E14 live convergence diagnostic (kgwctl#13), slot to be decided | M-1 matcher conformance; design the optional xDS cross-check for Phase 12 |
| later | a convergence phase only if the investigation names one (E14) | Phase 6 after E7 and E15 (scoped `merge.*` records); Phase 7 move into kgateway after E6; local MCP in a downstream build (E10, kgwctl D-023) | Phases 9, 14, then 10–13; Phase 12 consumes E4, E5, E13 through the E15 tag |

No hard blockers. kgwctl Phase 4, xdsview Phase 8 and Snellius Phase 8 run in parallel.

## Program decisions

- **P-001** krp owns the mapping from Envoy names to Gateway API refs. The UI's `computed.EnvoyNamer` and the CLI call it. Keep it swappable, so a CLI inside kgateway can use kgateway's own naming code.
- **P-002** Vendor-specific policy support (beyond kgateway OSS) is a plugin outside library cores. No vendor or edition names in any library.
- **P-003** The CLI is built from scratch. [kgateway#13654](https://github.com/kgateway-dev/kgateway/pull/13654) and earlier prototypes are idea sources only: the relation graph, label-based policy discovery, `check`, `analyze -f`. No gwctl fork, no go-graphviz.
- **P-004** The CLI imports the libraries and talks to the cluster directly. It does not need the UI backend. A thin-client mode against the UI API is optional, later.
- **P-005** ~~Get kgateway maintainer buy-in on the CLI early with a short design proposal, before a large PR.~~ Deferred 2026-11-05 (kgwctl D-011): the owner leads both the product and most maintainers; revisit at the CLI's upstream move.

- **P-006** snellius.dev is Snellius's user documentation. A Snellius change that users can see (pages, findings, flags, chart values, install steps, supported versions) updates the site in the same release: the Snellius phase or sub-phase does not close until the site matches. Finding codes come from Snellius `docs/findings.md` via `npm run sync:findings` in the site repo. The site presents Snellius as a kgateway UI only. Since 2026-11-20 the site has its own session, which owns all site changes, release updates and SEO included. At each Snellius release, Snellius files a `cross-repo` issue in snellius.dev that lists what users can see change; the site session makes the update, and the Snellius release is not done until it is live.
- **P-007** The walkthrough scenarios live in snellius-demo (2026-11-20; moved from a Snellius draft, snellius#41). Each is a small kgateway setup with the exact findings it raises, verified on a real cluster. They are shared test input: Snellius uses them to walk through and review the UI, and kgwctl uses them to check `check` against the same analyzer. A finding change in Snellius (new code, severity or timing) updates the affected scenarios' expected findings in the same release, like P-006.
- **P-008** (2026-11-22) The kgateway controller owns policy and translation semantics. Tools read declared and runtime configuration on their own. Resolved policy comes only from the controller. Details in "Policy provenance" below.
  1. Three layers. Declared: Gateway API and kgateway resources, targetRefs, status conditions. Resolved: attachment, precedence, inheritance, merge, override and translation, all computed by the controller. Runtime: the xDS Envoy serves.
  2. No tool reimplements policy resolution: precedence, inheritance, merge, override, or which policy is effective. The controller already reports per-field winners in xDS. Its semantics are configurable and have open bugs, so a copy would show what kgateway should do, not what it does.
  3. Deterministic structural controller logic may be mirrored, only as a registered mirror (rules below).
  4. Every answer carries its source: declared, controller-reported, or inferred, ambiguous or unknown. An attached policy is not an effective policy. Policy models keep the source on every value, so a later source (such as a future registered policy mirror) needs no model change.
  5. P-001 stands: xdsview's name mapping is runtime-to-declared correlation, with certainty on every answer.
  6. Snellius works without xDS. Its views come from declared state, controller status and metrics, and proxy stats. Runtime config from xdsview adds evidence and cross-checks; it never becomes a prerequisite.
  7. Allowing policy mirrors later takes a new program decision that supersedes point 2, under the same mirror rules.
- **P-009** (2026-11-22) xDS convergence replaces intended vs accepted. A generic diff of the controller snapshot against `/config_dump` is withdrawn: E4's intended-vs-accepted part and xdsview Phase 7 (xdsview D-011). The question is whether each proxy runs what the controller intends, per type. The xdsview session investigates it, design only, after xdsview Phase 8. Details in "xDS convergence" below.

- **P-010** (2026-11-23) xdsview is control-plane agnostic. Envoy semantics live in the core; product modules add provenance only. Phase 9 makes one coordinated contract break, after the P-009 investigation. Details in "Control-plane-agnostic xdsview" below.

- **P-011** (2026-11-25) Logical IDs that must survive replacement come from scoped names, never from UIDs or ranks. UIDs may key state of one incarnation. Details in "Architecture guidelines" below.
- **P-012** (2026-11-25) Moving to a later kgateway, downstream product, Gateway API or Envoy version reviews every versioned interface the tool consumes: CRDs, kind lists, version-keyed mappings and mirrors. Details in "Architecture guidelines" below.

## Policy provenance (P-008)

**Glossary.**
- *Declared target*: a policy's targetRef or targetSelector names the object.
- *Controller-reported attached*: the policy's ancestor condition `Attached` is True.
- *Controller-reported contribution*: the policy IDs in `merge.*` xDS metadata for a field at one config level. It does not say which level governs a request.
- *Runtime filter chain*: the HCM filters Envoy runs on a route. xdsview calls it `EffectiveChain`, which is not policy-effective. Docs and UIs say "runtime filter chain".
- *Effective policy* with no qualifier: not used.

**What the controller reports today** (kgateway main `8a71677fb8`, 2026-11-19):

| Source | Reports | Granularity | Status |
|---|---|---|---|
| Policy `status.ancestors`. `Accepted`: Valid, PartiallyValid, Invalid, Pending, TargetNotFound. `Attached`: Attached, Merged, Overridden, Pending, TargetNotFound | attached, plus a lossy hint of effect. Messages are fixed strings, except BackendConfigPolicy's TLS override, which names the winning BackendTLSPolicy | route and Gateway policies: per Gateway or ListenerSet, no sectionName, route or field. Backend policies: per backend, sectionName kept. TargetNotFound for explicit refs: a synthetic summary ancestor | stable API with known bugs (U3, U4). Not exhaustive: readers keep ancestors and reasons they do not recognise |
| xDS `filter_metadata["merge.<Kind>.<group>"]` | per field, the policy IDs that contributed at that level | listener, RouteConfiguration, virtual host, route, when merge origins exist; not the HCM, backendRef policies, clusters or DirectResponse | always on; internal and undocumented (U1) |
| xDS `dev.kgateway.route_source` | the HTTPRoute rule and match behind an Envoy route; not the delegation path | per route; synthetic fallback routes have none | off by default, experimental (U2) |
| admin `/snapshots/krt`, `/snapshots/xds` (9095) | KRT debug output, including attached policies per IR object; intended xDS per cache key, with SDS Secret payloads stripped; a key can return an error | per object; per key | debug only, no auth; binds localhost by default, configurable with `AdminBindAddress` |

**Mirror rules.** A mirror of controller logic is allowed only when all hold:
1. Registered below, naming the kgateway code it mirrors and the minors it is pinned to.
2. Conformance tests against real controller output, recorded per supported minor.
3. Checked against controller output at runtime where any exists. A disagreement is a finding or `unknown`, never a silent answer.
4. Labelled as computed in UI and CLI output.
5. Has an upstream ask and retires when it lands.

**Mirror register.**

| ID | Owner | Mirrors | Cross-check | Retires with | Status |
|---|---|---|---|---|---|
| M-1 | Snellius D-020 (`pkg/computed`, rule sets `kgw23`–`kgw25`) | view A route→listener attachment; view B delegation child selection, including rejected and dropped candidates; view C the delegated child's merged matcher (kgateway `pkg/kgateway/utils/delegation` `MergeParentChildRouteMatch`); kgateway 2.3–2.5 | KGWUI-012 on a status disagreement. Status checks do not confirm the matcher; matcher conformance is separate. Optional later cross-check: the Envoy route's match, when xDS is available (P-008 point 6) | per view: view C's matcher with U10; views A and B only when upstream covers them | registered (rule 1: Snellius D-128 item 1). Rule 2 not met for the matcher: goldens are hand-transcribed and cite kgateway's translator goldens per tag, but no test compares them; e2e checks delegation against status only. Conformance test tracked in Snellius (D-128 item 2) |
| M-2 | Snellius KGWUI-016 | only the zero-candidate diagnostic: a targetSelector matches no readable object (empty selects nothing; global policy namespace). Never attachment | none from status today | kgateway#14998 (U6) | registered, scoped |
| M-3 | convergence investigation | the controller's client key for a proxy pod: normalised role, hash of the pod's labels augmented with node hostname and locality, namespace | a match against a key in `/snapshots/xds` shows only that some client has that entry; it does not bind the sampled pod to it | U9 | candidate, not approved (owner, 2026-11-24): a key match doesn't bind the pod, and the mirror rules' cost buys little. Without it, kgwctl#13 answers `yes` only when all keys of a Gateway agree, otherwise `unknown`. Revisit if kgwctl reports too many `unknown`s. Interim: `DISABLE_POD_LOCALITY_XDS` where one key per Gateway is acceptable |
| M-4 | Snellius `pkg/install` scope (Snellius D-009, D-024) | the controller's discovery scope: `KGW_DISCOVERY_NAMESPACE_SELECTORS` parsed with kgateway's semantics (empty means all namespaces; OR of label selectors); kgateway `api/settings/settings.go` DiscoveryNamespaceSelectors, `pkg/pluginsdk/collections/discovery.go` NewDiscoveryNamespacesFilter; kgateway 2.3–2.5. Consumers: Snellius install assignment, KGWUI-004 and 007, `Scope.Excluded` (next Snellius release); kgwctl Phase 4 bundle scope | KGWUI-007 against `kgateway_resources_managed` per namespace | U12 | registered; rule 2 is a unit test only, not checked against real controller output (same gap as M-1) |

## xDS convergence (P-009)

Starting evidence (kgateway main `8a71677fb8`, Envoy 1.39.x; xdsview `docs/phases/phase-7/assessment.md`):
1. The controller exposes no per-proxy ACK record. It holds the sent nonce and version per stream and type in memory, for retry correlation only. NACKs show as `kgateway_envoy_xds_rejects_{total,active}` (labels: Gateway namespace, Gateway name, type; no pod) and `kgateway_envoy_xds_nack_resends_suppressed_total`, plus a log line. The active gauge clears on stream close or an error-free request for the type.
2. Versions are per type: opaque content hashes, sometimes composed (`-carry-N`), not monotonic, published per type and not atomically. The client key hashes the pod's labels augmented with node hostname and locality, so replicas on different nodes can have different keys.
3. The controller can hold back a newer translation for a client. The published snapshot is the intent a proxy should reach, not the latest translation.
4. `Programmed=True` is a translation result, not delivery evidence (kgateway#14453).
5. Envoy's `<type>.version_text` is the last update a subscription accepted; `update_success` counts them. Neither shows that the controller received the ACK, nor that the proxy serves it: a listener can still be warming. `client_status` is never set. Only LDS records `error_state`. A resource's `version_info` in `/config_dump` is the type version at its last change; EDS entries leave it unset.

Rules for the investigation:
- Zero NACKs does not prove convergence.
- A config dump does not prove ACK provenance.
- Accepted is not serving.
- Every claim names which side proves it: the controller, the proxy, or both.
- Every comparison can return `unknown`: when the pod's key is unproven, when captures are too far apart to tell drift from a publish in flight, or when a translation is held.
- A single capture gives capture time, not onset. "Since when" needs repeated observations or an event.

Questions to answer, each with an operator scenario:
1. Per replica, has each type reached the controller's version for its key?
2. Do replicas of one Gateway differ (`version_text` drift)?
3. Is a non-LDS rejection still in force after its counter stops moving?
4. Is a proxy disconnected or stale, and since when?
5. Are listeners or clusters stuck warming or draining?
6. Which of these Snellius KGWUI-001, 006, 008, 025, 026 and the withheld-config signal already answer.

Inputs: kgwctl Phase 4 bundles give proxy captures and controller metrics, timestamped. They leave out the controller's admin snapshots (kgwctl Phase 4 design). The investigation captures the controller side itself on kind clusters, and records which questions a bundle alone cannot answer. Output: a recommendation naming the operator scenarios, the provider of each piece (E14), and the upstream asks it depends on.

**Outcome (2026-11-24).** The investigation reported ([xdsview `263055d`](https://github.com/DuncanDoyle/xdsview/blob/263055d/docs/investigations/xds-convergence.md), two Codex reviews). Convergence is answered per proxy pod and per subscription, as `yes`, `no` or `unknown`, naming the proving side. No generic xDS diff. Providers are in E14; the upstream asks it refines are U7, U8 and U9.

**Bundle evidence as delivered (kgwctl Phase 4, 2026-11-27).** `manifest.json` lists each controller and proxy replica with namespace, pod, UID, owner, readiness, phase, start time and restart count per container. Each capture is an item with its own time and status. Per proxy replica: `config_dump.json` (with endpoints unless `--no-endpoints`), which carries `version_text`, listener states and the rejection details that Envoy reports; `stats.txt`, `metrics.txt`, `server_info.json`, `certs.json` and `clusters.txt`. Per controller replica: `metrics.txt`. It leaves out the controller's admin snapshots, so a bundle stays an offline ceiling (E14): it shows what each replica held at capture time, not whether that is what the controller intends.

## Control-plane-agnostic xdsview (P-010)

Layers, with dependencies pointing down only:
1. Consumers: krp, kgwctl, Snellius Phase 12, downstream product modules' callers. They compose execution first, then provenance.
2. Product modules: `adapter/kgateway` in xdsview; others downstream (P-002), against xdsview's public API only.
3. Provenance contracts: claims, evidence, composition.
4. Envoy core: `model`, `parser`, the effective chain, `render`, `source`; standard library only (xdsview D-004), enforced by the existing dependency test. `source/kube` stays the only client-go user.

Rules:
1. Execution analysis (parse, effective chain, order, sides, run and disable state, warnings) depends only on Envoy data. It never calls a module. The core never imports a module.
2. Modules get immutable or isolated input and return claims; the composer applies them. A test compares independent parses, with zero modules and with every supported composition: raw input and execution output are identical. Provenance is checked separately.
3. The core never invents execution. Where it cannot establish something (ECDS without config, sides of an unknown implementation, conditional wrappers, unparsed predicates or actions), it keeps the raw slot in its position with explicit uncertainty: an ECDS filter without config is not shown as running. Known Envoy wrapper behaviour (Composite, ExtensionWithMatcher) stays. Modules never supply sides.
4. Certainty is per claim. A `merge.*` policy record is an exact, level-local observation of validated controller metadata; it says nothing about which level governs a request (P-008). A field→filter link names both its record and one installed filter slot by occurrence. It is at most inferred when derived from a table, and only made when exactly one slot matches; otherwise the record stays unplaced (E13). An exact link needs evidence that names the filter at the same level (U11) and a decision update.
5. A module needs evidence per object. Caller-supplied context (a Gateway name, a product guess) is not evidence.
6. Composition is explicit and deterministic. Conflicting claims of equal authority become ambiguous; no first-wins.
7. Phase 9 is the one break. After it, contract changes are additive and driven by a named consumer case; another break needs its own program decision.
8. P-001, P-002, P-008, P-009, E6 and xdsview D-011 to D-018 stand. `origin`'s ownership of `merge.*` and `route_source` decoding moves with the kgateway module.

**Phase 9 scope** (xdsview designs the API; this list is what the break must contain):
1. Pure execution: the effective chain no longer imports provenance.
2. Execution and provenance as separate results over read-only input.
3. One claim per assertion with its own certainty, evidence kind and producing module. A policy record and its filter link are separate claims; the link references both occurrences (rule 4).
4. Results keyed by snapshot occurrence (listener, HCM, route config, virtual host, route, filter slot). Names are evidence, not keys: duplicate filter names and shared route configs stay distinct.
5. Listener, RouteConfiguration and virtual-host `merge.*` records decoded beside route records, level-local (P-008 glossary). The evidence scope of each runtime block is enumerable, so Snellius can withhold a whole block the viewer may not fully read.
6. Candidate route readings come from the kgateway module; selection and pruning are explicit calls by the consumer. Every candidate is kept and narrowing is recorded; krp's pre-filter and kgwctl's unresolved-route listing keep working.
7. A separate `NameMapper` with certainty, resource kind and ambiguity for names without a dump (E5, Snellius `EnvoyNamer`).
8. krp JSON `v1alpha2`. kgwctl builds its own payload instead of embedding xdsview's `model`.

Core correctness in the same phase, after the break: an active HCM whose RDS route configuration is absent from a complete dump (E14), static listeners and route configs (the kgateway corpus has a static readiness listener), non-HCM chains kept, ECDS and wrapper state kept, top-level header and query matchers, invert, `case_sensitive`, default filter chains, cluster selectors, direct-response and redirect actions, and a coverage policy for sections outside the analysis. OSS output changes; goldens are reviewed.

**Additive later**, each on a named consumer case: delegation path when a producer exists (U10); exact filter links (U11); wider policy scopes (U1); cluster origins beyond bare names; product detection from capture context.

**Other control planes.** Their dumps live in a private downstream repo, sanitized: no customer hosts, no inline secrets. That repo runs them through xdsview's core without any module at each xdsview release, and reports regressions to xdsview without naming the product. A downstream product module comes later, only when a consumer needs its attribution and an owner accepts responsibility for its compatibility; it depends only on xdsview's public API. Public records say "downstream product module". Public xdsview tests the core with neutral fixtures: a static HTTP bootstrap, a synthetic custom filter, and optionally an Envoy Gateway dump.

## Architecture guidelines

Rules every repo in the program follows in design and review. A design that breaks one says so and records why in its own `decisions.md`. Review: [`reviews/2026-11-25-p011-p012-codex.md`](reviews/2026-11-25-p011-p012-codex.md).

### Identity (P-011)

1. **Logical IDs come from scoped names.** An ID that must survive a recompute, restart, upgrade, reinstall or delete-and-recreate derives from group, kind, namespace and name, plus a discriminator the design defines (a section name, a finding code). An entity with no Kubernetes name may use a discriminator its producer guarantees stable; otherwise rule 6 applies. The design declares the scope (cluster or install) and includes it where keys cross that boundary. This covers finding IDs, URLs and joins across tools.
2. **No ranks or ordinals in logical IDs.** A position in a sorted list shifts when a sibling is added, removed or hidden, and rekeys survivors. Snellius D-132 fixed this for D-040's rank-based IDs.
3. **UIDs key one incarnation.** A UID may key state that belongs to the current object or pod, and may verify that two observations saw the same one: grace periods, restart holds, per-pod counter deltas, kgwctl's pod re-identify. It never keys a logical ID. generation, resourceVersion and content hashes track revisions within one incarnation; they are not instance markers either.
4. **Recreate keeps the ID, not every clock.** A same-name recreate keeps its logical ID, so bookmarks and URLs still work. Each time value says what it measures: a grace period or `since` that measures the current incarnation may restart.
5. **Names that are not stable are not keys.** Generated pod names and control-plane-generated Envoy names churn or collide. A rename changes a name-based ID unless the design adds an alias or migration. Runtime-to-declared joins go through xdsview's provenance mapping (P-001, P-010).
6. **Positional and unnamed IDs stay in their snapshot.** xdsview's Phase 9 occurrence IDs are valid only in the dump that produced them. An entity with no stable discriminator stays snapshot-local; a join across snapshots reports it as ambiguous or unknown.

Each repo checks its existing logical IDs against these rules at its next phase start; Snellius's remaining D-040 ranks are the known case. Snellius D-102 and D-132 already follow rule 1.

### Version bumps (P-012)

A repo that moves to a later kgateway, downstream product, Gateway API or Envoy version reviews, in the same change, every versioned interface it consumes.

1. **Diff the CRDs, where the tool reads them.** Old against new, for each product.
2. **New kinds:** decide RBAC and support for each. A kind list the tool holds (RBAC roles, watched kinds, kind tables) gains the kind or records why not.
3. **Changed and removed kinds:** check served and storage versions, removed kinds and versions (stale watches and version mappings), new Secret references, and new credential fields that redaction must cover.
4. **Mappings outside CRDs.** Tables keyed by product or Envoy version go stale with an empty CRD diff: xdsview's policy-field-to-filter and filter-type tables, for example. Check them against controller output or Envoy fixtures for the new version.
5. **Mirrors.** Before claiming support for a new minor, run the mirror rules: update the pinned minors and check conformance against real controller output.
6. **Each list has a check for its intended scope.** A guard test, an existing behavioural test, or a recorded exclusion counts. With no authoritative input to test against, the repo documents the exception and its fallback check. kgwctl D-028 is the reference: `TestReaderRoleCoversServedKinds` plus a bump checklist next to its test CRDs.

Applies to xdsview, kgwctl, Snellius and every downstream build with its own lists or mappings. Found when Gateway API v1.7.0 served UDPRoute and three experimental kinds the kgwctl reader role lacked.

## Ownership

| Concern | Owner | Consumers |
|---|---|---|
| Policy resolution and translation | kgateway controller (upstream) | everyone, read-only |
| Declared state: resources, targetRefs, status | Snellius `pkg/cache`, `objects`, `explore` | kgwctl (E2, E7) |
| Reading ancestor conditions into findings | Snellius `pkg/health` | kgwctl `check` (E3, E8) |
| Envoy name → Gateway API ref; decoding `merge.*` and `route_source` metadata (no other decoder) | xdsview `origin` (P-001); the kgateway module from Phase 9 (P-010) | Snellius (E5, and M-1's optional cross-check), kgwctl (E5, E7) |
| Runtime filter chain (execution) | xdsview `chain` (E11); module-free from Phase 9 (P-010) | kgwctl, Snellius (E4) |
| Field→filter attribution, labelled | xdsview `chain` (E13); the kgateway module and composition from Phase 9 (P-010) | kgwctl, Snellius (E4) |
| Product modules for other control planes | their downstream owners (P-002, P-010) | their own callers |
| Registered mirrors | the project named in the register | — |
| Findings, including future convergence findings | Snellius `pkg/health` | kgwctl `check` |
| Convergence data capture | kgwctl Phase 4 bundle (proxy side, controller metrics); the investigation (controller snapshots on kind) | the investigation |
| Convergence investigation | xdsview session | program (E14) |
| Upstream asks | the owner decides what is filed | — |

## Upstream kgateway dependencies

None is filed by a tooling session without the owner's approval.

| ID | Ask | Upstream | Unblocks |
|---|---|---|---|
| U1 | document `merge.*` metadata as a contract; stamp it on the HCM, backendRef and cluster policies and DirectResponse | none yet | E7 completeness; wider attribution after E13 (not a gate for E13) |
| U2 | `route_source` metadata on by default | kgateway#15067, #15068 | exact route origins (E5); M-1's optional xDS cross-check |
| U3 | `Overridden` only when overridden on every listener | kgateway#14861, fix #14870 | E7 status reading |
| U4 | status names the winning policy and keeps sectionName | #14919 (sectionName); none yet for the winner | E7 |
| U5 | GatewayExtension status | kgateway#14792, #14815 | declared view |
| U6 | status says when a targetSelector matches nothing | kgateway#14998 | retires M-2 |
| U7 | per stream and type, tied to pod identity and controller replica: each ACK or NACK correlated to its response by nonce, with version, time, the response's resource names and, for a NACK, the error; connect time per stream; a `pod` label on `rejects_*`. An error-free request from a new subscription stays distinguishable from an ACK. Define what a newer publication and a stream close do to "rejection in force". It proves a response was NACKed; which resource failed still needs proxy evidence, because one response feeds several watches (P-009 investigation §4.2) | none yet; related kgateway#14264 | P-009 questions 1, 3, 4; E14 verdicts without `unknown` |
| U8 | `Programmed` reflects NACKs. Investigation case: an RDS NACK left a new listener `Programmed=True` that served 404 | kgateway#14453 | P-009, E14 |
| U9 | return each pod's snapshot key for the connection that pod holds; a proxy cannot learn it, since Envoy shows only the bootstrap role. Alternative today: `DISABLE_POD_LOCALITY_XDS` where one key per Gateway is acceptable | none yet | retires M-3; E14 per-pod verdicts |
| U10 | delegation chain provenance in xDS metadata | none yet | retires M-1 view C; views A and B need their own evidence |
| U11 | when kgateway writes `typed_per_filter_config[<filter>]` from a policy, record filter name → policy refs and field in the metadata of the same RouteConfiguration, virtual host or route (Envoy's `FilterConfig` has no metadata). Check the field numbers against the Envoy version kgateway ships before filing | none yet (proposed by xdsview, D-016) | exact filter links in E13 when several filters match a field |
| U12 | a read-only controller endpoint with identity, version, controllerNames, parsed discovery selectors, selected namespaces and enabled kinds | [kgateway#15070](https://github.com/kgateway-dev/kgateway/issues/15070) (filed 2026-11-22) | retires M-4; KGWUI-007 for experimental kinds (Snellius D-017 §4) |

## Open questions

1. ~~Do M-1's goldens test the matcher against real controller output per minor (rule 2), or only status? Snellius answers.~~ Answered 2026-11-22 by Snellius D-128: rule 2 not met for the matcher.
2. How does M-1's optional xDS cross-check join an Envoy route to a delegation path when one child is emitted in several contexts? It returns `ambiguous` rather than pick one. Snellius, when Phase 12 is designed.
3. ~~Snellius's core (`pkg/engine`, `pkg/api`) imports the kgateway rule sets through `pkg/computed/rulesets`. Should that selection move behind the kgateway pack (Snellius D-003)? Snellius answers.~~ Answered 2026-11-22 by Snellius D-128: selection goes through the pack, fallbacks move later.
4. Is `/snapshots/xds` safe to collect in a support bundle? Phase 4 leaves it out for now. kgwctl, later.
5. Which upstream asks to file, and in what order? Owner.
6. ~~Does the convergence investigation produce a new edge, a Snellius finding, or nothing?~~ Answered 2026-11-24: both, see E14.
7. Exact Go names and shapes of the Phase 9 contract (E15). xdsview's Phase 9 design decides. kgwctl and Snellius review the design before it is approved.
8. ~~M-3 (the client-key mirror) before U9?~~ Answered 2026-11-24: not now; see the mirror register.
9. Can the core take Envoy-build knowledge from the caller (descriptions of filters a custom Envoy build adds: sides, config decoding) without breaking P-010 rule 1? Today modules only add claims after the parse. Raised by the owner 2026-11-27. xdsview, when a consumer names a case.

## Coordination protocol

1. **Phase start (every repo):** read this file. Check open `cross-repo` issues in your repo and in the repos you wait on.
2. **Requests between repos:** a GitHub issue in the provider repo, label `cross-repo`, title prefix `[from <repo>]`, body names the edge ID.
3. **Contracts are tags.** Consumers require a tagged version. For co-development, use an uncommitted `go.work` in the parent directory (see `go.work.example`).
4. **Phase close:** update the edge status and contract table here. Propose a release tag if a contract changed. Notify each consumer with a `cross-repo` issue in its repo naming the edge and version. Snellius also files a `cross-repo` issue in snellius.dev for the site session (P-006).
5. **Live pings:** local sessions can message each other (`SendMessage`; find names with `ListAgents`, named after the repo directory). Use it for "tag published" or "issue filed" only. The record lives in this file and in issues.
6. **Changes to this file** are made by the session whose phase changed, in a small commit of its own.
7. **Reviews:** a review brief or reviewer prompt in any repo links "Architecture guidelines" in this file, so reviewers and running sessions check against the current rules.
