# c7-apps Baseline — What Detour Takes

Inventory of the infrastructure, shared code, conventions and tooling to lift
from `c7-apps` for the .NET rewrite of the Detour backend. Source of truth for
the c7 side: `/home/jonas/git/c7/c7-apps`.

The goal is *"looks like c7-apps, isn't c7-apps"* — same shape, same
conventions, same operational ergonomics, none of the c7 product surface
(multi-tenancy, C2, MCP toolsets, AI chat, reporting, SIEM, MITRE).

Functional target: [BACKEND_FUNCTIONAL_SPEC.md](BACKEND_FUNCTIONAL_SPEC.md).

---

## 1. Sizing the target

Detour is a **single-tenant consumer app**: individual accounts, no
organisations, no tenant isolation, no admin control plane over customer
deployments. c7-apps is a **multi-tenant B2B platform**. That difference decides
most of the include/exclude calls below.

Closest existing model in c7-apps: **`backend/Steward`** — four projects
(`.Api`, `.Domain`, `.Database`, `.Tests`), a plain `Program.cs`, no tenancy,
minimal shared surface. Take Steward's *skeleton* and layer on the auth,
persistence and observability pieces that SevenHunter/Hackerflow use.

## 2. Target project layout

```
backend/
  Directory.Build.props            ← from c7-apps/backend
  Directory.Packages.props         ← from c7-apps/backend (trimmed)
  xunit.runner.json                ← from c7-apps/backend
  Detour.slnx
  Detour/
    Detour.Api                     ← controllers, DI, middleware pipeline, background jobs
    Detour.Domain                  ← entities, repository interfaces, domain services
    Detour.Database                ← DbContext, entity configurations, repositories, migrations
    Detour.Domain.Tests
    Detour.Database.Tests          ← Testcontainers-backed
    Detour.InfraTests              ← WebApplicationFactory integration tests
  Shared/                          ← copied subset, see §4
    ...
    SharedTests.slnx
```

Notes on the shape:

- **`.slnx`**, not `.sln`. c7-apps moved to the XML solution format throughout.
- Three-layer split with **no reference from Domain to Database**. `Domain`
  declares `IRepository<T>` interfaces; `Database` implements them.
- `Api` is the only project that knows about ASP.NET Core.
- Test projects sit beside the projects they test, named `<Project>.Tests`, plus
  a single `InfraTests` for end-to-end pipeline tests.
- The live relay (§11 of the functional spec) needs a decision: a hosted service
  inside `Detour.Api` (recommended — one process, one deployment) or a separate
  `Detour.Relay` executable mirroring `C2Server.Listener.Http`.

## 3. Repo-level scaffolding to copy

| Item | Source | Why |
|---|---|---|
| `backend/Directory.Build.props` | as-is | Wires `xunit.runner.json` into every `*Tests` project so parallel Testcontainers suites don't starve CPU. |
| `backend/Directory.Packages.props` | trim | Central package version management (`ManagePackageVersionsCentrally`, `CentralPackageTransitivePinningEnabled`). One place for every version, including transitive pins for CVEs. Strip AI/C2/Azure/SIEM entries. |
| `backend/xunit.runner.json` | as-is | Caps xunit parallelism per assembly. |
| `dotnet-tools.json` | as-is | Pins `dotnet-ef` (currently 10.0.7) so migrations are reproducible. |
| `sonar-project.properties` | pattern | Per-solution Sonar config. Only if SonarQube is in scope. |
| `.editorconfig` / format rules | c7 root | `dotnet format style --severity info --verify-no-changes` is the CI gate. |
| `.gitattributes` | as-is | Generated-file and line-ending handling. |

Runtime baseline: **.NET 10**, `ImplicitUsings` and `Nullable` enabled in every
project.

## 4. Shared projects

### 4.1 Dependency graph (the part that matters)

```
Shared.Domain
  └─ Shared.Database
       ├─ Shared.Identity
       ├─ Shared.Permissions
       ├─ Shared.Audit
       └─ Shared.Api ── Shared.Extensions, Shared.Sse
Shared.MultiTenancy  ←── pulled in by Shared.Keycloak, Shared.Rbac, Shared.BackgroundJobs
```

**`Shared.Keycloak` hard-references `Shared.MultiTenancy`.** So do `Shared.Rbac`
and `Shared.BackgroundJobs`. This is the single biggest porting decision — see
§5.

### 4.2 Take

| Project | What it gives Detour |
|---|---|
| `Shared.Domain` | `Entity` base, `IRepository<T>`, `IPostCommitActionScheduler`, `ResultErrorMatching`, SSRF/URL guards. Foundation for everything else. |
| `Shared.Database` | `BaseDbContextFactory`, `BaseRepository`, `PaginatedResult`, `NpgsqlDatabaseConventions` (**global snake_case naming — this is why almost every explicit `HasColumnName` is a bug**), `SmartEnumNameConverter`, soft-delete query filters, `ColumnTypes`, advisory-lock helpers. |
| `Shared.Api` | OpenAPI installer and transformers (bearer scheme, permission requirements, SmartEnum parameters), JWT HMAC signing/verification, `HostEnvironmentExtensions`, request-type context, private-IP guard. **Drop** the MCP transformers and `S2SBearer*` (service-to-service auth — Detour has one service). |
| `Shared.Api.ResultTypeUtils` | `ResultExceptionHandler` — turns a thrown `ResultException` into a localised 400 `ProblemDetails` instead of a 500. The other half of the Result convention. |
| `Shared.Api.HealthChecks` | `HealthController`, `HealthCheckReport`, response writer, `ChildServiceHealthCheck`. Pairs with the `dotnet-health-checks` skill. |
| `Shared.Api.RateLimiting` | `AddDefaultRateLimit`, `RateLimitSettings`. Directly replaces the hand-rolled per-IP auth limiter (functional spec §15.2). |
| `Shared.Identity` | `User`, `IUserRepository`, `UserRepository`, `UserConfiguration` — the local mirror of an identity-provider user. Needed: Detour's domain (friendships, group membership, ownership) keys on a local user row, not on a token claim. |
| `Shared.Permissions` | Permission entities, repositories, `IUserPermissionCodeLookup`, `ModelBuilderExtensions`. Detour's needs are thin (user vs. administrator), but the plumbing is what the `[Authorize(Policy = …)]` convention in the `dotnet-api-endpoint` skill assumes. |
| `Shared.Extensions` | String/JSON/int/`PathString` helpers used throughout the shared code. |
| `Shared.Configuration` | `DeploymentContext`, `AddFileBackedSecrets` (`<Key>_FILE` → read from path), `AddDevcontainerSettings`, `IdpConfigurationExtensions` (reads the Keycloak-provisioned `idp.json`), startup profile logging. |
| `Shared.Logging` | `ConfigureSerilog` + `SerilogConfiguration`. |
| `Shared.OpenTelemetry` | `SetupOpenTelemetry` — traces, metrics, logs to the LGTM stack. |
| `Shared.Caching` | `AddCaching` / `CacheConfiguration` / `CacheTimes` (FusionCache-style memory + Redis, with fail-safe durations). Replaces the ad-hoc 20-second dashboard cache. |
| `Shared.Translations` | `ITranslator`, `AddTranslations`, `Culture`. Required by `ResultExceptionHandler` for localised problem details. |
| `Shared.Sse` | `SseEventBus`, `SseEndpointExtensions`, keep-alive service. Candidate transport for circle presence events and, if the WebSocket relay is descoped, for low-cadence circle updates. |
| `Shared.Audit` (+ `.EF`, `.AspNetCore`) | `AuditService`, `AuditLog`, API-request and security audit middleware, retention job. Detour has **no** audit trail today; this is one of the "more security" wins. |
| `Shared.BackgroundJobs` | `BaseBackgroundJob`, cron scheduling (Cronos), transaction modes, retry policy, status broadcaster + SSE stream, job inventory/metrics. Needed for token/invite/reset cleanup, presence-event retention, cache warming. **Drags in `Shared.MultiTenancy`** — see §5. |

### 4.3 Take only if the feature lands

| Project | Condition |
|---|---|
| `Shared.S3ObjectStorage.*` | Only if trace/trip blobs or exports move to object storage. Not needed for a straight port. |
| `Shared.Rbac` | Only if Detour grows real roles beyond user/administrator. Also drags in `Shared.MultiTenancy`. |
| `Shared.StateMachine` | Only if something in Detour becomes a workflow. Nothing does today. |
| `Shared.Reporting.*` | PDF/report generation. No Detour feature needs it. |
| `Shared.Net`, `Shared.Ssh`, `Shared.Git`, `Shared.Docker` | No Detour use. |

### 4.4 Leave behind (c7 product surface)

`Shared.Ai*`, `Shared.Mcp`, `Shared.Mitre.*`, `Shared.Siem*`, `Shared.Stixnet`,
`Shared.C2Server.*`, `Shared.Hackerflow.Client`, `Shared.Steward.*`,
`Shared.Api.Mtls`, `Shared.ApiKeys*`¹, `Shared.MultiTenancy.AzureKeyVault`,
`Shared.FeatureFlags`, `Shared.Domain.Caching`, `tools/McpToolGenerator`,
`scripts/generate-*.sh`.

¹ `Shared.ApiKeys` is tempting — Detour *does* need read-only dashboard keys
(functional spec §12). But c7's implementation is tenant-scoped HMAC-JWT with a
tenant revocation cache. Read it for the design, write a single-tenant version.

## 5. The Keycloak / multi-tenancy decision

**This needs a decision before any code is written.**

`Shared.Keycloak` provides exactly what the rewrite wants — `IdpTokenService`,
`KeycloakAdminService`, `IdpRoleTransformer`, `IdpBackchannelHandler`,
`IdpSettings`, permission-claim caching — but its only authentication installer
is `AddMultiTenantKeycloak<TTenant, TUser, TDbContext>`, built on a custom
`MultiTenantJwtBearerHandler` that resolves issuer and audience per tenant.

Two options:

| Option | Shape | Cost |
|---|---|---|
| **A — take it whole** | Copy `Shared.Keycloak` + `Shared.MultiTenancy`, run with exactly one tenant (the `SingleTenant` profile c7 already supports). | Every entity carries tenancy it never uses; per-tenant datasource cache, tenant resolution middleware, `X-Tenant` header handling, credential store — all dead weight in a consumer app. Also drags tenancy into `Shared.BackgroundJobs` and `Shared.Rbac`. |
| **B — fork single-tenant (recommended)** | Copy `IdpSettings`, `IdpTokenService`, `KeycloakAdminService`, `IdpRoleTransformer`, `KeycloakUtils`, `Constants`, `BaseUser`/`DefaultAuthUser`/`SharedUserService`. Replace `AddMultiTenantKeycloak` with stock `AddAuthentication().AddJwtBearer()` bound to one issuer/audience, plus the permission-claim middleware. | One installer to write (~50 lines). Removes `Shared.MultiTenancy` from the graph entirely, and with it the tenancy variants of `Shared.BackgroundJobs` and `Shared.Rbac`. |

Take **B** unless multi-tenancy is a real Detour roadmap item.

### 5.1 What Keycloak replaces

Everything in functional spec §4 and most of §13:

| Current home-grown | Keycloak |
|---|---|
| Password hashing (PBKDF2, per-user salt) | Keycloak credential store |
| Bearer session tokens + idle expiry | OIDC access + refresh tokens |
| Password reset links, single-use, TTL | Keycloak "forgot password" flow |
| Invite codes, single-use, expiring | Keycloak registration policy / an invite client, **or** keep as domain state and gate registration on it |
| Admin browser session + CSRF token | Keycloak admin console, or an OIDC-authenticated admin surface |
| `is_admin` flag | Realm role |
| Per-IP auth rate limiting | Keycloak brute-force detection **and** `Shared.Api.RateLimiting` |

**Do not lose the rules while moving the mechanism.** Fail-closed registration,
identical answers for "no such account"/"wrong password"/"not an admin", one
live reset link per account, and "a reset signs you out everywhere" are all
Keycloak realm configuration, not accidents of the old implementation.

### 5.2 Keycloak provisioning

c7 provisions realms with **Terraform in a container**
(`ghcr.io/crimson7research/hackerflow-terraform-keycloak`), run once as a compose
service, writing the resulting client config to shared volumes as
`idp.backend.json` / `idp.frontend.json`, which the apps then read via
`Shared.Configuration`'s `IdpConfigurationExtensions`.

The image is c7-internal. Detour needs its own equivalent: either a small
Terraform module (realm, backend client, mobile public client with PKCE, roles,
`redirect_uris`, `post_logout_redirect_uris`) or a realm-import JSON. **The
config-file-on-a-volume handoff pattern is worth copying either way** — it keeps
client secrets out of appsettings and out of environment variables.

Mobile-specific consideration not present in c7: Detour's client is a native app,
so it needs a **public client with PKCE and a custom-scheme redirect URI**, not
c7's confidential web client behind a BFF. c7's `Shared.Bff` is for browsers and
does not apply.

## 6. Local development stack

c7's `.devcontainer/` is a modular compose stack. Copy the structure, take the
services Detour needs.

`docker-compose.yml` uses `include:` to pull per-service files from
`services/`, all under one project name, with `profiles: [devcontainer, local]`
so the same files serve both the in-container and host-run workflows.

| Service | File | Detour |
|---|---|---|
| **Postgres** (app) | `docker-compose.postgres.yml` | **Yes** — `postgres:18-alpine`, healthcheck, internal network. |
| **Keycloak + its own Postgres + Terraform provisioner** | `docker-compose.keycloak.yml` | **Yes** — `keycloak:26.x`, separate `keycloak-db` (parity with deploy), `keycloak-setup` one-shot provisioner, `keycloak-export` utility profile. Replace the c7 image and Terraform variables. |
| **Traefik** | `docker-compose.traefik.yml` | **Yes** — `*.localhost` routing on :8000, near-prod origins, makes OIDC redirect URIs realistic. |
| **Redis** | `docker-compose.redis.yml` | **Yes** — distributed cache backing `Shared.Caching`; also the natural fan-out bus if the relay ever runs more than one instance. |
| **LGTM** (Grafana + Loki + Tempo + Prometheus + Pyroscope) | `docker-compose.grafana.yml` + `grafana/dashboards/*.json` + `grafana/provisioning/` | **Yes** — target for `Shared.OpenTelemetry`. Take the `api-overview`, `dotnet-runtime`, `errors-exceptions`, `performance`, `trace-debug`, `background-jobs` dashboards; drop `frontend-performance`. |
| **RustFS** (S3) | `docker-compose.rustfs.yml` | Only with §4.3's object storage. |
| **Zitadel** | `docker-compose.zitadel.yml` | No — alternative IdP, unused. |

Also copy:

- `.devcontainer/devcontainer.json`, `Dockerfile`, `bootstrap.sh` — the container
  the agent and the developer both work in.
- `.env.example` — documented knobs (`PROJECT_NAME`, `KC_*`, `POSTGRES_*`,
  `DOCKER_SOCK_GID`).
- `docker-compose.override.yml.example` — the host-run variant that publishes
  ports for services started from an IDE.
- The **canonical port map discipline** from the `local-stack` skill: every
  service has one port, and a clash is fixed by killing the occupant, never by
  relocating the service (a renumber invalidates Keycloak redirect URIs and
  requires a Keycloak volume wipe).

Detour's own port allocation still has to be chosen (c7 uses 7100/7300/7400/7600
for APIs, 6xxx for BFFs, 3xxx for SPAs).

## 7. Production deployment shape

| Item | Source | Notes |
|---|---|---|
| `docker/prod/Dockerfile.*` | one per service | Multi-stage .NET publish. Take the pattern, one Dockerfile for `Detour.Api`. |
| `docker/prod/docker-compose.yml` | as-is pattern | Production topology. |
| `steward-template/package/deployment/compose/*.yml` | pattern | Per-service compose fragments: `postgres.yml`, `keycloak.yml`, `traefik.yml`, `lgtm.yml`. Cleanest reference for a self-hosted single-box deployment. |
| `steward-template/package/deployment/config/postgres/init/10-create-app-roles.sh` | as-is | Least-privilege app role separate from the superuser — a concrete security win over the current SQLite file. |
| `steward-template/package/deployment/config/traefik/` | pattern | Per-environment Traefik configs generated from a template, plus TLS and middleware dynamic config. |

**Do not** take the Steward packaging machinery itself (signed `manifest.jwt`,
vendor key trust anchor, `update.zip`, phone-home). That exists to ship a product
to third-party operators; Detour self-hosters install from the repo.

The current `server/install.sh` (Proxmox LXC creation, systemd units, GraphHopper
and Photon setup) has **no c7 equivalent** and must be rewritten or retired as
part of the migration — it is the current install story and 900+ lines of it.

## 8. CI/CD

c7's `.github/workflows/`:

| Workflow | Take? |
|---|---|
| `pr-backend.yml` | **Pattern yes.** `dorny/paths-filter` change detection → per-solution build/test/format jobs → conditional image builds. Detour has one solution, so it collapses to a handful of jobs. Keep the draft gate (`if: draft == false`, `types: [..., ready_for_review]`). |
| `format.yml` | **Yes** — `dotnet format style --severity info --verify-no-changes` on changed files, excluding `Migrations/`. |
| `_pr-image-build.yml`, `_push-image.yml` | **Pattern yes** — reusable workflows for build-and-push. |
| `release.yml`, `nightly.yml` | Pattern, if Detour wants tagged releases and a nightly. |
| `sonarqube-*.yml` | Only with SonarQube. |
| `pr-frontend.yml`, `pr-routines.yml` | No. |

Also: `scripts/check-migration-n1.py` (guards against N-1 incompatible
migrations) and `scripts/check-root-solution-membership.sh` (every project is in
the root solution) are both cheap and worth porting.

Note c7 runs on paid Blacksmith runners, which is why the `local-ci-act` skill
exists. Detour on GitHub-hosted runners does not need that discipline — which is
why that skill was **not** copied (§10).

## 9. Conventions the shared code enforces

These are the "structure and clarity" the rewrite is for. Each is documented in a
copied skill.

| Convention | Where | Skill |
|---|---|---|
| **Result type over exceptions** — domain and repositories return `Result`/`Result<T>` (`JV.ResultUtilities`); controllers call `ThrowIfFailure`; the global handler turns it into a localised 400. | `Shared.Api.ResultTypeUtils`, `Shared.Domain` | `dotnet-result-type` |
| **SmartEnum over `enum`** (`Ardalis.SmartEnum`) — serialised by name, converted in EF by `SmartEnumNameConverter`, surfaced in OpenAPI by a transformer. | `Shared.Database`, `Shared.Api` | `dotnet-smart-enum` |
| **snake_case column naming applied globally by convention** — an explicit `HasColumnName` is almost always a mistake. | `NpgsqlDatabaseConventions` | `dotnet-entity-configuration` |
| **Entity = private setters + named update methods + validation keys**, never public setters. | `Shared.Domain.Entity` | `dotnet-domain-entity` |
| **Repository interfaces in Domain, implementations in Database**, `BaseRepository` for the common shape. | `Shared.Domain` / `Shared.Database` | `dotnet-domain-entity` |
| **Controllers: auth attribute + permission policy + OpenAPI metadata (`EndpointSummary`, `EndpointDescription`, `ProducesResponseType`) + response DTOs with static `Map` methods.** | `Shared.Api` | `dotnet-api-endpoint` |
| **Background jobs as `IHostedService`** with cron schedules, declared transaction mode, retry policy and a status stream. | `Shared.BackgroundJobs` | `dotnet-background-job` |
| **Health checks with `tags: ["critical"]`** and a standard report writer. | `Shared.Api.HealthChecks` | `dotnet-health-checks` |
| **Migrations are the only sanctioned schema change**, per-project startup project, never hand-edited. | `dotnet-ef` tool | `ef-migrations`, `generated-code-guardrail` |
| **Transaction middleware after authentication**, so unauthenticated requests never open a transaction. | `Shared.Api` middlewares | — |
| **ISO 25010 / 5055 maintainability limits** (cyclomatic ≤10, function ≤50 LoC, nesting ≤4, params ≤5). | — | `maintainable-coding` |

The **middleware order** is itself a convention worth copying verbatim from
`SevenHunter.Api/Startup.cs` — the ordering comments there record real bugs
(audit before rate limiting, security audit before the limiter, transactions
after auth).

## 10. Agent tooling

### 10.1 Copied into this repo

`.claude/skills/` — 17 skills, plus `.claude/hooks/generated-code-guard.sh` and
`.claude/settings.json` (which wires the hook as a `PreToolUse` guard):

`debugging`, `dotnet-api-endpoint`, `dotnet-background-job`, `dotnet-build-test`,
`dotnet-domain-entity`, `dotnet-entity-configuration`, `dotnet-health-checks`,
`dotnet-result-type`, `dotnet-smart-enum`, `dotnet-tests`, `ef-migrations`,
`file-bug-issue`, `generated-code-guardrail`, `git-pr-workflow`, `local-stack`,
`maintainable-coding`, `pr-review`.

They are copied **verbatim** and still name c7 paths, solutions and services. See
[.claude/skills/PORTING.md](../../.claude/skills/PORTING.md) for which ones need
retargeting and what to change.

### 10.2 Deliberately not copied

| Skill | Why |
|---|---|
| `frontend-build-check`, `frontend-component-creation`, `frontend-folder-creation`, `navigation-scheme-decision`, `page-migration`, `page-review`, `page-tabs`, `react-custom-hook`, `tanstack-form`, `tanstack-query-hook`, `tanstack-wizard` | Bun/Turborepo/React/TanStack in `frontend/apps/*`. Detour's client is Kotlin Multiplatform. Nothing transfers. |
| `regenerate-mcp-tools` | Regenerates Hackerflow/C2Server/SevenHunter MCP toolsets. No Detour equivalent. |
| `kiota-dotnet-integration` | Generates .NET clients from c7 OpenAPI specs. Detour's only consumer is the Kotlin app. |
| `codebase-navigation` | A layout map of the Hackerflow monorepo plus a graphify index. Detour has neither, and is small enough not to need one. |
| `local-ci-act` | Exists because c7 pushes cost money on Blacksmith runners. Not Detour's problem. |

### 10.3 Available but not yet taken

- `.claude/commands/` — `fix-pr`, `format`, `sonar`, `sonar-auto`, `tackle-issue`,
  `grafana-error-scan`, `grafana-performance-scan`. Take `fix-pr` and `format`
  once CI exists; the Grafana and Sonar ones once that infrastructure is up.
- `.claude/routines/` — 20 scheduled agent routines (CI triage, PR self-heal,
  stale sweep, dependency digest, Sonar autofix). Heavily coupled to c7's
  GitHub project, labels and runners. Revisit once the repo has comparable CI.
- `.mcp.json` — SonarQube, Grafana and Excalidraw MCP servers, plus HTTP MCP
  endpoints for the c7 apps. Take the Grafana entry when LGTM is running.
- `bruno/` — API collections per environment, populated by the Keycloak
  Terraform run (`IDP.bru`). Worth mirroring for Detour's endpoints.
- `CONTRIB.md`, `CLAUDE.md`, `RELEASE.md` — c7's own conventions documents.
  Detour needs equivalents; c7's are the model, not the content.

## 11. Migration considerations specific to Detour

Things with no c7 precedent that the rewrite has to answer:

1. **SQLite → Postgres data migration.** One file per deployment today. Needs a
   one-shot importer for accounts, trips, traces, points, places, friendships,
   groups, shared routes and badges. Passwords do **not** migrate — every user
   must go through a Keycloak reset, which is a user-visible event that needs
   planning.
2. **The live relay — deliberately not built.** The convoy live surface (live
   position relay, push-to-talk, destination offers and votes) has no
   implementation in the rewrite yet, and that is a decision rather than an
   omission.

   The first attempt was Server-Sent Events out and REST in. The rules ported
   cleanly — membership per frame, the circle voice ban, the pause switch — but
   the transport does not: the old relay wrote base64 PCM16 chunks into an open
   socket, and one talking member becomes roughly 25 POSTs a second over REST.
   The rules survive the change; the efficiency does not.

   **Circles do not depend on it.** Positions and presence events are ordinary
   REST reads and writes, already built and tested, which is what the low-cadence
   design called for anyway. Only convoys lose their live feed.

   To revisit: SignalR (nearest c7 precedent, `C2Server.Api`'s hub), raw
   WebSockets, or SSE out with a socket kept only for audio. `Shared.Sse` is
   registered and unused, so the outbound half is a few lines whenever the
   inbound half is settled.
3. **The read-only dashboard API and its key type.** Single-tenant fork of
   `Shared.ApiKeys` (§4.4 footnote).
4. **Two server-rendered HTML pages** (the Home Assistant dashboard and the
   admin dashboard) currently live inside the Python file. Decide whether they
   stay backend-rendered, move to static assets served by the API (as Steward
   does with `UseDefaultFiles`/`UseStaticFiles`), or are retired in favour of
   Keycloak's admin console plus Grafana.
5. **Opaque JSON blobs.** Trips, routes and places are stored unparsed on
   purpose. Postgres `jsonb` with the existing caps preserves that; a fully
   normalised schema would not.
6. **Installer story.** `server/install.sh` has no c7 counterpart. Compose
   fragments from `steward-template` are the replacement shape.

## 12. Suggested order

1. Skeleton: solution, three projects, `Directory.*.props`, `.slnx`, CI format
   job. Nothing but a health endpoint.
2. Local stack: Postgres + Keycloak + Traefik + Redis + LGTM compose, realm
   provisioning, `idp.json` handoff.
3. Auth: single-tenant Keycloak installer, `Shared.Identity` user mirror,
   permission claims middleware, one authorised endpoint end to end.
4. Persistence: `Shared.Database` conventions, entity configurations,
   first migration, Testcontainers test project.
5. Port the functional spec in dependency order: accounts → sync → friends →
   fog → routes → groups → circles → places/events → dashboard API → admin.
6. Relay last (biggest unknown, and everything else works without it — the
   current system already degrades that way).
