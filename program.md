# kgateway tooling program

**Last updated:** 2026-11-22

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
| krp | `pkg/envoy/model`, `parser`, `filter`, `source`, `source/kube`, `origin`, `chain`, `render`; `krp dump -o json` (`v1alpha1`) | delivered, krp Phases 4–6 | `v0.4.3` (module `github.com/DuncanDoyle/xdsview`); one Go break since v0.3: `model.WeightedCluster` is not comparable (D-009). v0.4.1 (2026-11-21): attribution fixes #28, RouteExtra D-010 |
| Snellius | `pkg/cache`, `engine`, `objects`, `kinds`, `computed`, `setup`, `ctrlmetrics` | delivered, UI Phases 1–5; versioned API (UI D-003). From `v0.6.0`, controller metrics reach the engine only as a collected value: `ctrlmetrics.CollectOnce` + `Publish` into `setup.Config.Collected` (UI Phase 5 design §7); `computed.Rules` loses `Delegation` (optional `computed.Delegator`); `extension.NewCore` and packs are additive. `v0.7.0` adds `cache.ListOnce` and `cache.Config.Namespaces` (UI Phase 6, D-076). `v0.8.0` adds `explore.Index.RouteSignals` (configuration and traffic path per route; UI Phase 7, D-087) | `v0.8.0`; `v0.8.1` and `v0.8.2` change only the web UI, no contract |
| Snellius | analyzer: `pkg/health` (codes 001–026), `pkg/diagnose` (groups, `Fix`) | delivered, UI sub-phases 3.3–3.5 (023, 024 in 3.4) | `v0.8.0`; unchanged in `v0.8.1` and `v0.8.2` |

`model`, `parser` and `filter` must not import client-go, lipgloss or bubbletea, so the UI can import them under its D-016 import rules. Fetching lives in `source`; port-forwarding, the only client-go user, is in `source/kube`.

## Dependency edges

| ID | Consumer | Needs | Provider deliverable | Status |
|---|---|---|---|---|
| E1 | CLI Phase 3 | `proxy routes` (krp view in the CLI) | krp Phase 4 library, `v0.1.0` | done — krp `v0.1.0` (2026-11-04); notified kgwctl#1; consumed by CLI Phase 3 on xdsview `v0.4.3`, complete 2026-11-21 |
| E2 | CLI Phase 1 | `get` / `describe` on live cluster | UI `pkg/cache` + `engine` used one-shot | done — kgateway-ui `v0.3.0` (2026-11-05, kgateway-ui#16); consumed by CLI Phase 1, complete 2026-11-06. `v0.4.0` (2026-11-07): list-only kinds (kgwctl#17) and `cache.Wait` (kgwctl#20). `v0.5.0` (2026-11-08): module path `github.com/DuncanDoyle/snellius`. `v0.6.0` (2026-11-09): packs and `extension.NewCore`, additive; `extension.New` unchanged. `v0.7.0` (2026-11-12): `cache.ListOnce`, a list-only snapshot (R2, snellius#18), and `cache.Config.Namespaces` for namespace-scoped RBAC (R3, snellius#19), additive; notified kgwctl#8. `v0.8.0` (2026-11-19): two signals per route through `explore.Index.RouteSignals`, additive; a caller without computed views gets `path: unknown, no-path` for an accepted route without a fault; notified kgwctl#9 |
| E3 | CLI Phase 5 | full `check` analysis, `analyze -f` | UI 3.3 analyzer exported under `pkg/` | available — kgateway-ui `v0.4.0` (2026-11-07): the analyzer over a cache snapshot, with 023 and 024; `analyze -f` (snapshot from files) is not in UI Phase 3 (UI R79). `v0.5.0` (2026-11-08): module path `github.com/DuncanDoyle/snellius`; 3.5 fixes. `v0.6.0` (2026-11-09): pack checks run only on their installs; kgateway findings unchanged |
| E4 | UI Phase 12 | runtime filter chain tab | krp Phase 4 (+5 mapping, +6 runtime filter chain) ~~+7 intended vs accepted~~ | available — krp `v0.2.0` (model, mapping, `source/kube` cancellation); xdsview `v0.4.0` (2026-11-21) adds the runtime filter chain per route and listener state; notified kgateway-ui#21, snellius#63. Intended vs accepted is withdrawn (P-009, 2026-11-22; xdsview D-011): no provider. Policy labels on the chain need E13 first |
| E5 | UI, CLI | Envoy name → Gateway API ref mapping | krp Phase 5 `origin` (see P-001) | available — krp `v0.2.0` (2026-11-05); notified kgwctl#2, kgateway-ui#21 |
| E6 | CLI move into kgateway | license-clean, vendor-neutral libraries | krp Apache-2.0 license (Phase 4); vendor policy plugins outside core (P-002) | open — license in krp `v0.1.0`; P-002 seam (`origin.Chain`) in `v0.2.0`; krp's `testdata/enterprise/` and the enterprise transformation type in the core mapper still to remove: xdsview Phase 8, xdsview#32, target `v0.5.0` (xdsview D-013) |
| E7 | CLI Phase 6 | policies in `describe`, each labelled with its source (P-008): declared (targetRefs as written), controller-reported attached (ancestor conditions as the controller reports them), and, for a route, the policy IDs in that route's `merge.*` metadata | Snellius Phase 8 declared-policy model in `pkg/`; xdsview `origin.Policies` for the route-level xDS part (E5, E13). Listener, RouteConfiguration and virtual-host metadata have no provider yet | open. Redefined 2026-11-22 (P-008): was "attached and effective policy". `merge.*` is level-local: it does not say which level governs a request. Anything beyond controller-reported evidence stays out of scope, also after U1/U4 land |
| E8 | CLI Phase 2 | `check` MVP | UI health checks (KGWUI-001 onward) run one-shot via `engine.Compute`, with controller metrics from one synchronous scrape | done — consumed by CLI Phase 2 on Snellius `v0.8.2`, complete 2026-11-20 (kgwctl#5–#7 closed). kgateway-ui `v0.3.0` (2026-11-05): `pkg/setup` and `ctrlmetrics.Static` (UI D-053, kgateway-ui#13); `v0.4.0` adds `setup.Options.NoEndpoints`; `v0.5.0` (2026-11-08) renames the module to `github.com/DuncanDoyle/snellius` and drops the status-sync count from `ctrlmetrics`. Breaks in Snellius `v0.6.0` (UI Phase 5, on `main` from 2026-11-09): `ctrlmetrics.Static`, `ctrlmetrics.Source` and `cfg.Engine.Metrics` are gone; use `ctrlmetrics.CollectOnce` + `Publish` into `setup.Config.Collected` (kgwctl#6) |
| E9 | CLI Phase 2 `check --live` | what a check is waiting on before it decides (holds) | Snellius 5.7: optional `hold` {`until` RFC 3339, `count` in a per-check unit, `reason` for display} on each check result; `ok` and `pending` unchanged | done — consumed by CLI Phase 2 `check --live`, complete 2026-11-20. Delivered Snellius `v0.7.0` (2026-11-12); first in `v0.7.0-alpha.1` (2026-11-10, pre-release; [snellius#33](https://github.com/DuncanDoyle/snellius/issues/33), Snellius D-068) |
| E10 | a downstream CLI build on kgwctl `pkg/cli` (local MCP, stdio; kgwctl D-023) | one read-only MCP tool set shared with Snellius's remote MCP, fed from a one-shot load with the user's kubeconfig | Snellius MCP tool set under `pkg/` (Snellius D-089); Snellius UI Phase 7 sets its visibility and audit contract | planned — UI Phase 7 shipped the visibility and audit contract in `v0.8.0` (2026-11-19, design §9–§10); the tool set package is still to come |
| E11 | CLI Phase 3.2 | `proxy routes` tree and filter order per route | xdsview: exported stdlib text renderer `pkg/envoy/render`, `EffectiveChain` per route, listener state (active, warming, draining, error); UI-neutral types in `model` for Snellius Phase 12 (E4) too | done — consumed by CLI Phase 3 on xdsview `v0.4.3`, complete 2026-11-21 (kgwctl#10 closed). xdsview `v0.4.0` (2026-11-21, Phase 6); xdsview#27 closed; notified kgwctl#10. v0.4.1 (2026-11-21): attribution fixes #28, RouteExtra D-010. v0.4.2 (2026-11-21): narrowing edge cases #29. v0.4.3 (2026-11-21): warning for a skipped HCM, narrowing keeps listeners mid-update #31 |
| E12 | a downstream consumer (separate service and web UI) | reuse of Snellius login, authorization and UI building blocks | Snellius: `pkg/auth/oidc` and `pkg/auth/session`; the SubjectAccessReview checker, cache and limiter in `pkg/auth/sar`, split from Snellius's visibility layer; `@snellius/shell` theme tokens and components, published or as a git dependency | deferred (2026-11-21) — nothing is extracted until the consumer's design starts, after Snellius Phase 14 and the remote MCP. Decide then which pieces to extract and version as contracts. Until then, keep `pkg/auth/oidc` and `pkg/auth/session` free of Snellius-specific imports. A generic external-link action on Snellius pages is deferred with it. Reused policy rendering is declared and controller-reported only (P-008) |
| E13 | CLI `proxy routes` (shipped, Phase 3); UI Phase 12; E7 | every attribution states its certainty: a policy contribution is at most `inferred`; a policy field links to a filter only when exactly one filter entry matches; unmapped fields stay listed, not dropped (xdsview#30) | xdsview Phase 8.1 (xdsview D-012, D-014), released with Phase 8 as `v0.5.0`: JSON adds `certainty` (additive), text adds `(inferred)`, consumer goldens change | open — owner moved it first in Phase 8 (2026-11-22). Correctness fix: kgwctl `proxy routes` shows inferred attribution as fact today |
| E14 | to be named by the convergence investigation (P-009) | per-proxy, per-type xDS convergence evidence | to be decided; xdsview supplies Envoy-side primitives only when the investigation names an operator scenario (xdsview D-011) | proposed — not a contract. Nothing is promised until the investigation reports |

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

Done: xdsview Phases 4–6 (`v0.4.3`); kgwctl Phases 1–3; Snellius Phases 1–7 (`v0.8.2`).

| Priority | xdsview | kgwctl | Snellius |
|---|---|---|---|
| now | Phase 8 vendor-neutral core → `v0.5.0`: 8.1 attribution certainty (E13), then the enterprise corpus and the seam (E6) (xdsview D-015) | Phase 4 support bundle: proxy-side convergence inputs and controller metrics per replica, timestamped; E13 golden update when `v0.5.0` is tagged | Phase 8 Policies & Filters: declared policy first, controller-reported provenance second, no resolver (P-008) |
| next | xDS convergence investigation, design only (P-009) | Phase 5 full analysis, `analyze -f` | M-1: record its rule-1 entry and matcher conformance; design the optional xDS cross-check for Phase 12 |
| later | a convergence phase only if the investigation names one (E14) | Phase 6 after E7; Phase 7 move into kgateway after E6; local MCP in a downstream build (E10, kgwctl D-023) | Phases 9, 14, then 10–13; Phase 12 consumes E4, E5, E13 |

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
| M-1 | Snellius D-020 (`pkg/computed`, rule sets `kgw23`–`kgw25`) | view A route→listener attachment; view B delegation child selection, including rejected and dropped candidates; view C the delegated child's merged matcher (kgateway `pkg/kgateway/utils/delegation` `MergeParentChildRouteMatch`); kgateway 2.3–2.5 | KGWUI-012 on a status disagreement. Status checks do not confirm the matcher; matcher conformance is separate. Optional later cross-check: the Envoy route's match, when xDS is available (P-008 point 6) | per view: view C's matcher with U10; views A and B only when upstream covers them | registered; Snellius to record matcher conformance (rule 2) |
| M-2 | Snellius KGWUI-016 | only the zero-candidate diagnostic: a targetSelector matches no readable object (empty selects nothing; global policy namespace). Never attachment | none from status today | kgateway#14998 (U6) | registered, scoped |
| M-3 | convergence investigation | the controller's client key for a proxy pod: normalised role, hash of the pod's labels augmented with node hostname and locality, namespace | a match against a key in `/snapshots/xds` shows only that some client has that entry; it does not bind the sampled pod to it | U9 | candidate, not approved |

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

## Ownership

| Concern | Owner | Consumers |
|---|---|---|
| Policy resolution and translation | kgateway controller (upstream) | everyone, read-only |
| Declared state: resources, targetRefs, status | Snellius `pkg/cache`, `objects`, `explore` | kgwctl (E2, E7) |
| Reading ancestor conditions into findings | Snellius `pkg/health` | kgwctl `check` (E3, E8) |
| Envoy name → Gateway API ref; decoding `merge.*` and `route_source` metadata (no other decoder) | xdsview `origin` (P-001) | Snellius (E5, and M-1's optional cross-check), kgwctl (E5, E7) |
| Runtime filter chain; field→filter attribution, labelled | xdsview `chain` (E11, E13) | kgwctl, Snellius (E4) |
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
| U7 | per-stream ACK and NACK record (version, nonce, last error) with a connected-nodes endpoint or CSDS; pod label on the rejects metrics | none yet; related kgateway#14264 | P-009 questions 1, 3, 4 |
| U8 | `Programmed` reflects NACKs | kgateway#14453 | P-009 |
| U9 | expose each pod's snapshot key | none yet | retires M-3 |
| U10 | delegation chain provenance in xDS metadata | none yet | retires M-1 view C; views A and B need their own evidence |

## Open questions

1. Do M-1's goldens test the matcher against real controller output per minor (rule 2), or only status? Snellius answers.
2. How does M-1's optional xDS cross-check join an Envoy route to a delegation path when one child is emitted in several contexts? It returns `ambiguous` rather than pick one. Snellius, when Phase 12 is designed.
3. Snellius's core (`pkg/engine`, `pkg/api`) imports the kgateway rule sets through `pkg/computed/rulesets`. Should that selection move behind the kgateway pack (Snellius D-003)? Snellius answers.
4. Is `/snapshots/xds` safe to collect in a support bundle? Phase 4 leaves it out for now. kgwctl, later.
5. Which upstream asks to file, and in what order? Owner.
6. Does the convergence investigation produce a new edge, a Snellius finding, or nothing? The investigation decides (E14).

## Coordination protocol

1. **Phase start (every repo):** read this file. Check open `cross-repo` issues in your repo and in the repos you wait on.
2. **Requests between repos:** a GitHub issue in the provider repo, label `cross-repo`, title prefix `[from <repo>]`, body names the edge ID.
3. **Contracts are tags.** Consumers require a tagged version. For co-development, use an uncommitted `go.work` in the parent directory (see `go.work.example`).
4. **Phase close:** update the edge status and contract table here. Propose a release tag if a contract changed. Notify each consumer with a `cross-repo` issue in its repo naming the edge and version. Snellius also files a `cross-repo` issue in snellius.dev for the site session (P-006).
5. **Live pings:** local sessions can message each other (`SendMessage`; find names with `ListAgents`, named after the repo directory). Use it for "tag published" or "issue filed" only. The record lives in this file and in issues.
6. **Changes to this file** are made by the session whose phase changed, in a small commit of its own.
