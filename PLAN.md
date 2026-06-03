# Plan: AWS CDK EKS + S3 + IRSA Platform

A TypeScript CDK platform that provisions EKS (managed node groups), an S3 bucket, and an IRSA-backed service account scoped to that bucket. Resources compose into domain-shaped constructs (`WorkloadPlatform`, `AppStorage`, `DeployPipeline`) behind a typed extension contract. Environment configuration is a schema-validated domain model; the type system encodes invariants so invalid combinations (e.g. prod with `DESTROY`) cannot be authored. Destructive operations are gated by IAM trust policies and GitHub Environment protections. CI/CD runs on GitHub Actions via OIDC with PR-comment diffs, scheduled drift detection, and CDK assertion + snapshot tests gating merges. `dev` deploys today; `nonprod` and `prod` configs are defined for future use.

This repository is the **platform**. Application deployments live in their own repositories and consume a small, stable contract (cluster identity, per-tenant namespace + IRSA role + S3 binding, per-tenant deploy role, SSM-published outputs). Feature-branch preview environments are an application-pipeline concern; the platform provides the primitives (namespaces, IRSA, quotas, TTL reaping) that make previews safe and cheap.

## Purpose: observability backend benchmark

The platform's first concrete use case is a head-to-head **cost and query-quality benchmark** of three observability backends fed from the same OpenTelemetry pipeline:

- **ClickHouse** — self-hosted on EKS via the Altinity ClickHouse Operator, using S3 as the primary storage tier (`MergeTree` with the `s3` disk).
- **OpenSearch** — self-hosted on EKS via the OpenSearch Operator (`opensearch-project/opensearch-k8s-operator`), with EBS gp3 hot tier and S3 snapshot repo.
- **Datadog** — SaaS; cost computed from the equivalent log/metric/trace volume and cardinality observed at the OpenTelemetry Collector, evaluated against a versioned rate-card.

A single OpenTelemetry Collector receives **CLF-formatted web access logs** (replayed deterministically from a static corpus in S3) converted to **OTLP/OpenTelemetry-formatted messages**, then fans out to three exporters: ClickHouse, OpenSearch, and Datadog. A continuous query benchmark (every minute) runs a fixed query set against ClickHouse and OpenSearch and records latency + bytes scanned. Cost is derived from AWS Cost Explorer cost-allocation tags (ClickHouse, OpenSearch) and a Datadog rate-card calculator (Datadog), all surfaced on a single CloudWatch dashboard and persisted to a `benchmark-results` S3 bucket queryable via Athena.

The benchmark runs in `dev` with per-tenant capacity overrides so stateful targets get on-demand, dedicated nodes; the rest of the env keeps its cheap defaults.

## Steps

### Phase 1 — Project scaffolding
1. `cdk init app --language typescript`; pin Node/CDK; ESLint + Prettier; strict `tsconfig`; `.nvmrc`, `.gitignore`.
2. Add Renovate (or Dependabot) config for CDK, addons, GH Actions versions.
3. Add CODEOWNERS: `config/` and `lib/stacks/` reviewed separately from `lib/constructs/` (different blast radius).
4. Add `.envrc.example` (direnv): `AWS_PROFILE`, `AWS_REGION`, `ENV_NAME`.

### Phase 2 — Environment configuration
5. `config/schema.ts` — Zod schemas for environment configuration. Parsed once in `bin/app.ts`; invalid configuration fails synth with a precise error.
6. `config/types.ts` — discriminated union encoding per-env invariants in the type system:
   - `type EnvProfile = DevProfile | NonprodProfile | ProdProfile`
   - `ProdProfile` requires `removalPolicy: 'RETAIN'`, `autoDeleteObjects: false`, `terminationProtection: true`, `humanDeployAllowed: false`
   - Common fields: `name`, `aws: { account, region }`, `tags`, `github: { owner, repo, branches[] }`
   - **Intent-shaped sizing**: `workload: { size: 's'|'m'|'l', capacity: 'spot'|'on-demand' }`. Constructs map intent to instance types in one place, so AWS instance-family changes touch a single file.
   - `capabilities: Capability[]` — typed list (`{ kind: 's3-bucket' }`, future: `{ kind: 'redis', size: 's' }`); the `EksStack` iterates and attaches matching addons. This is the extension seam.
   - `tenants: Tenant[]` — list of application tenants onboarded to this env. Each tenant declares: `name`, `namespace`, `serviceAccount`, `s3Access: 'rw'|'ro'|'none'`, `github: { owner, repo, branches[] }`, optional `capacityOverrides: { capacity?: 'on-demand'|'spot', dedicatedNodeGroup?: boolean, taints?: Taint[] }` for stateful or benchmark workloads, optional `benchmark?: { target: 'clickhouse'|'opensearch'|'datadog', dataset: string }` to opt into the benchmark wiring. Empty in prod until tenants are added.
7. `config/environments.ts` — frozen map of envs (`dev`, `nonprod`, `prod`) type-checked against the union. Authoring `prod` with `removalPolicy: 'DESTROY'` is a compile error.
8. `config/index.ts` — `loadEnvConfig(name)` parses with Zod, returns the typed profile, throws structured errors.

### Phase 3 — Domain-shaped constructs (`lib/constructs/`)
Constructs describe capabilities the platform offers.
9. `network/network-construct.ts` — VPC, public + private-with-egress subnets, EKS subnet tags.
10. `platform/workload-platform-construct.ts` — EKS cluster + node group + core addons (VPC CNI, CoreDNS, kube-proxy, EBS CSI, AWS LBC via Helm) + IRSA OIDC provider. Exposes typed methods: `addServiceAccount`, `grantClusterAdmin`, `attach(addon)`.
11. `platform/platform-addon.ts` — extension contract:
    ```ts
    interface PlatformAddon {
      readonly id: string;
      attach(platform: WorkloadPlatform, env: EnvProfile): void;
    }
    ```
    Adding a capability means implementing this interface and registering a factory keyed by `Capability.kind`; `EksStack` is unchanged.
12. `storage/app-storage-construct.ts` — bucket (SSE, BPA, versioning, removal policy / autoDelete from config) + `bindToServiceAccount(sa, access)` for narrow IAM grants. Implements `PlatformAddon`. Prod attaches a bucket policy denying `s3:DeleteBucket`/`s3:DeleteObject*` outside the deploy role.
13. `tenants/tenant-construct.ts` — given a `Tenant`, provisions: a k8s `Namespace` (via `cluster.addManifest`) with `ResourceQuota` and `LimitRange`; an `eks.ServiceAccount` (IRSA) with the requested S3 access scoped to a per-tenant bucket prefix (`s3://<bucket>/<tenant>/*`); a per-tenant **app-CI deploy role** trusted to the tenant's GitHub repo via OIDC (scoped to deploying into its own namespace and pushing to its ECR repo — no cluster-wide IAM); writes `namespace`, `serviceAccountRoleArn`, `bucketName`, `bucketPrefix`, `deployRoleArn`, `clusterName`, `clusterOidcIssuer` to SSM under `/platform/<env>/tenants/<name>/*`. This is the platform-to-app contract.
14. `tenants/namespace-reaper.ts` — k8s `CronJob` (deployed by the platform) that deletes namespaces whose `platform.example.com/ttl` label is in the past. App pipelines set `ttl` on PR-preview namespaces; platform enforces it. Decouples policy (set by app) from enforcement (owned by platform).
15. `cleanup/k8s-loadbalancer-cleanup.ts` — CDK custom resource (Lambda + provider framework) that, on stack delete, deletes k8s LoadBalancer Services / Ingresses and waits for ELB drain. Lives in version control, has unit tests, and surfaces in CFN events.
16. `pipeline/github-oidc-construct.ts` — `iam.OpenIdConnectProvider` + platform deploy role with trust scoped to `repo:{owner}/{repo}:ref:refs/heads/{branch}`. The prod role's trust additionally requires the GitHub Environment claim (`environment:prod`). Per-tenant app-CI roles share this OIDC provider but are created by `TenantConstruct`.

### Phase 4 — Stack composition (blast-radius isolation)
17. `lib/stacks/bootstrap-stack.ts` — `GithubOidcConstruct`. One per account.
18. `lib/stacks/network-stack.ts` — `NetworkConstruct`. Exposes `vpc`.
19. `lib/stacks/storage-stack.ts` — `AppStorageConstruct`. Exposes `bucket`.
20. `lib/stacks/eks-stack.ts` — `WorkloadPlatformConstruct` + iterates `env.capabilities` and calls `platform.attach(addonFor(cap))`. Includes `K8sLoadBalancerCleanup` and `NamespaceReaper`.
21. `lib/stacks/tenants-stack.ts` — iterates `env.tenants` and instantiates one `TenantConstruct` per entry. Separate stack so tenant changes don't redeploy the cluster.
22. All stacks: apply tags from config, set `terminationProtection`, env-prefixed names (`dev-EksStack`).
23. `bin/app.ts` — `loadEnvConfig(process.env.ENV_NAME)` then instantiate stacks.

### Phase 5 — Tests
24. `test/config.test.ts` — every env round-trips through Zod; discriminated-union exhaustiveness check; tenant names unique per env.
25. `test/storage-stack.test.ts` — encryption on, public access blocked, removal policy + autoDelete match config; prod has the delete-deny bucket policy.
26. `test/eks-stack.test.ts` — node group instance type derives correctly from intent sizing; the IRSA role's trust `Condition` restricts `sub` to `system:serviceaccount:<ns>:<sa>`.
27. `test/tenants-stack.test.ts` — each tenant gets a namespace, IRSA role, and deploy role; IRSA S3 grants are scoped to the tenant prefix and deny cross-tenant access; tenant deploy role trust is scoped to that tenant's GitHub repo only; SSM parameter paths match the contract.
28. `test/github-oidc.test.ts` — trust restricts `repo:` AND `ref:`; the prod role additionally requires the `environment:prod` claim.
29. `test/snapshots/*.snap` — per-stack snapshots; CI surfaces unintended template diffs.
30. `test/cleanup-handler.test.ts` — unit tests for the cleanup Lambda handler (mock AWS SDK + kube client).
31. `integ/` — `@aws-cdk/integ-tests-alpha` against a sandbox account on a manual-trigger workflow.

### Phase 6 — Safety via IAM & GitHub Environments
32. **Prod deploy role assumable only by GitHub Actions.** Trust policy lists the GitHub OIDC provider with the `environment:prod` claim required. Local `ENV_NAME=prod cdk deploy` fails authentication at the AWS layer.
33. **GitHub Environment `prod`** requires reviewer approval and is restricted to `main`.
34. **Prod S3 bucket policy** denies destructive actions to any principal except the deploy role.
35. **CFN `terminationProtection: true`** on prod stacks (from config).
36. **Per-tenant app-CI roles** are scoped to: assume from the tenant's GitHub repo only; ECR push to the tenant's repo; `eks:DescribeCluster` for kubeconfig; no cluster-wide IAM. Apps cannot escalate beyond their namespace.

### Phase 7 — Platform/app boundary
This repository owns the platform; apps live in their own repositories. The contract between them is small, stable, and discovered via SSM — not hardcoded.
37. **What the platform publishes per tenant** (under `/platform/<env>/tenants/<name>/`):
    - `clusterName`, `clusterOidcIssuer`, `region`
    - `namespace`
    - `serviceAccountRoleArn` (IRSA role the tenant's pods assume)
    - `bucketName`, `bucketPrefix` (the tenant's S3 prefix)
    - `deployRoleArn` (the role the tenant's GitHub Actions assume)
    - `ecrRepository` (created by `TenantConstruct`)
38. **What the app pipeline does** (in the app's own repo): build image → `docker push` to ECR → `aws eks get-token` → `helm upgrade` (or Argo CD sync) into its assigned namespace. Reads all coordinates from SSM. Has no IAM permissions outside its tenant.
39. **Feature-branch previews are an app-pipeline responsibility.** The platform provides the primitives:
    - **Per-PR namespaces**: app pipelines create `<tenant>-pr-<num>` namespaces with `platform.example.com/ttl=<rfc3339>` and `platform.example.com/tenant=<name>` labels. The platform's `NamespaceReaper` CronJob deletes expired ones.
    - **Per-branch S3 isolation via prefix**: the IRSA role grants `s3://<bucket>/<tenant>/*`; the app pipeline writes per-branch keys under `<tenant>/pr-<num>/...`. No new IAM per PR.
    - **Resource quotas + LimitRanges** are inherited from a platform-defined template (configurable per tenant); a runaway preview cannot starve the cluster.
    - **Image isolation** via ECR tag conventions (`pr-<num>-<sha>`); no new ECR repo per PR.
    The platform CDK does not redeploy on app PRs. App-only changes never touch this repo.
40. **Platform's own feature-branch testing** is rare and addressed by: assertion + snapshot tests on every PR (cheap, fast); manual-trigger integ tests against a sandbox account; an optional future leased pool of pre-warmed dev clusters when the team feels the pain. Documented as a roadmap item; not built today.
41. **Versioning the contract**: SSM parameter paths and the `Tenant` config shape are the platform's public API. Breaking changes require a changeset entry and a deprecation window; a `test/contract.test.ts` asserts the published parameter set matches a frozen snapshot.

### Phase 8 — Operator UX
42. `package.json` scripts cover the common path: `build`, `test`, `lint`, `synth`, `diff`, `deploy`, `destroy`, `kubeconfig`. Each reads `ENV_NAME` from env (direnv) or accepts `--env`.
43. Optional `Makefile` with short aliases over npm scripts for muscle-memory operators.
44. `scripts/check-tools.sh` — verifies `node`, `aws`, `cdk`, `kubectl` versions on `npm run setup`.
45. A typed Node CLI (`commander`) is a future option if scripting needs outgrow npm scripts.

### Phase 9 — CI/CD with feedback loops
46. `.github/workflows/pr.yml`:
    - `npm ci && npm run lint && npm test` — gate merge.
    - OIDC assume read-only role; `cdk synth` + `cdk diff`; **post diff as a PR comment**. Branch protection requires the diff comment + green CI before merge.
47. `.github/workflows/deploy.yml`:
    - Push `main` → deploy `dev` automatically.
    - `workflow_dispatch` with environment input → `nonprod`/`prod` (gated by GH Environment approval).
    - Steps: checkout → setup-node → `npm ci` → `npm test` → OIDC assume per-env deploy role → `cdk deploy --all --require-approval never`.
48. `.github/workflows/drift.yml` — scheduled daily `cdk diff` against deployed stacks per env; opens an issue on drift.
49. **CDK Pipelines** as a future option: self-mutating pipeline so the deploy infrastructure follows the same review process as application code.

### Phase 10 — Observability
50. `lib/observability/platform-alarms.ts` — CloudWatch alarms declared alongside the resources they watch:
    - EKS control plane error rate, node group `NotReady` count
    - IRSA token issuance failures (CloudTrail metric filter on `sts:AssumeRoleWithWebIdentity` failures)
    - S3 4xx/5xx rates on the app bucket
    - CFN `UPDATE_ROLLBACK` events → SNS topic
51. SNS topic per env (subscriber config out of scope today; topic + policy in code).
52. New constructs include their alarms as part of the construct's definition.

### Phase 11 — Benchmark targets & telemetry pipeline
The benchmark is composed of five tenants plus shared support; each tenant is just a `Tenant` entry in `dev` that declares its `benchmark` block and its `capacityOverrides`. The constructs below are `PlatformAddon`s registered against new `Capability.kind`s.
54. **Storage class capability** — `lib/constructs/platform/storage-classes.ts` registers a gp3 `StorageClass` (`platform-gp3`) used by all stateful tenants for parity. Implements `PlatformAddon`.
55. **Tenant: `loadgen`** — deterministic CLF replay from a static corpus in `s3://<bucket>/loadgen/corpus/`. Stateless `Deployment` with an OTel SDK that emits OTLP logs (with OTel semantic-convention attributes: `service.name`, `http.*`, `client.address`, etc.) at a configurable rate. Read-only S3 access to the corpus prefix. Run-id, target rate, and duration come from a `ConfigMap` written by the platform from the tenant config.
56. **Tenant: `otel-collector`** — a single Collector deployment (gateway pattern) running the contrib distribution. ConfigMap defines one logs pipeline with three exporters fanning out: `clickhouseexporter` (writes to ClickHouse `otel_logs` schema), `opensearch` exporter (writes to OpenSearch indices), `datadog` exporter (writes to Datadog API). Same batch size, same retry policy, same attribute set across exporters — the Collector config is the experiment definition and is reviewed as code. Exposes its own metrics via `prometheus` exporter on `:8888` for the billing calculator and dashboard.
57. **Tenant: `clickhouse`** — deployed via Altinity ClickHouse Operator (Helm). `ClickHouseInstallation` CR defines the cluster; `s3` disk uses the platform bucket prefix `s3://<bucket>/clickhouse/<run-id>/`; hot tier on gp3 PVCs. IRSA grants the exact S3 action set ClickHouse needs (`Get/Put/List/DeleteObject`, `*MultipartUpload*`) on its prefix. Schema preloaded by an init Job using the OTel logs schema pinned to the Collector's exporter version. `capacityOverrides`: `on-demand`, `dedicatedNodeGroup`, taint `benchmark=clickhouse:NoSchedule`. Exposes its own ServiceMonitor for the dashboard.
58. **Tenant: `opensearch`** — deployed via OpenSearch Operator. `OpenSearchCluster` CR defines a small data-node cluster with `platform-gp3` PVCs. IRSA grants S3 access for the snapshot repo (`s3://<bucket>/opensearch/snapshots/`). Static admin credentials in Secrets Manager (out-of-band populated). `capacityOverrides`: `on-demand`, `dedicatedNodeGroup`, taint `benchmark=opensearch:NoSchedule`.
59. **Tenant: `query-bench`** — a `Deployment` (continuous, one query per minute per backend) running a small TypeScript service that holds the canonical query set:
    - "Last 1h logs where `http.response.status_code` >= 500"
    - "p99 latency by `service.name` over 24h"
    - "Top 10 `user_agent.original` by request count, last 7d"
    Each query is expressed in three dialects (ClickHouse SQL, OpenSearch DSL, Datadog Logs Search) and run against each backend. Records `latency_ms`, `bytes_scanned` (where exposed), `result_row_count`, and a result-fingerprint for parity checking. Writes one row per execution to `s3://<benchmark-results-bucket>/query-bench/<run-id>/` as Parquet.
60. **Cost-allocation tagging invariant** — every taggable resource created for a benchmark tenant carries `benchmark=true`, `target=<clickhouse|opensearch|datadog>`, `run-id=<uuid>`. Enforced by an aspect in `lib/constructs/benchmark/tagging-aspect.ts` and asserted by `test/benchmark-tagging.test.ts`.
61. **Renovate group `benchmark-pinned`** for the OTel Collector contrib image, ClickHouse exporter schema version, OpenSearch operator + image, Altinity operator + ClickHouse image. Manual approval only (no auto-merge) since version bumps invalidate prior runs.

### Phase 12 — Cost & query-quality measurement
62. **`lib/constructs/benchmark/billing-calculator.ts`** — a Lambda + EventBridge schedule (every minute) that scrapes the OTel Collector's Prometheus endpoint via the cluster (over a private LB or in-cluster CronJob proxying out), reads the logs/traces/metrics ingestion counters per exporter, and emits CloudWatch custom metrics under namespace `Platform/Benchmark`:
    - `ClickHouse/BytesIngested`, `OpenSearch/BytesIngested`, `Datadog/BytesIngested`
    - `Datadog/MetricCardinalityActive` (computed from unique `(metric_name, attribute_set)` tuples observed)
    - `Datadog/EquivalentCostPerMin` (logs GB × ingest rate + indexed events × indexed rate + custom metrics × cardinality rate, from `config/datadog-rate-card.json`)
63. **`config/datadog-rate-card.json`** — hand-maintained, checked into the repo, reviewed via PR. Fields: `logs_ingested_usd_per_gb`, `logs_indexed_usd_per_million`, `metrics_custom_usd_per_metric_per_host_per_month`, `traces_ingested_usd_per_gb`, `traces_indexed_usd_per_million`. Documented source URL and capture date in a sibling `.md`.
64. **`lib/stacks/benchmark-results-stack.ts`** — separate stack with: a dedicated `benchmark-results` S3 bucket (versioned, RETAIN, no autoDelete — these are the experimental record); a Glue database `benchmark`; Glue tables `query_bench`, `cost_minutely`; an Athena workgroup `benchmark` with result location set to a workgroup-only prefix. Tagged `benchmark=true`.
65. **`lib/constructs/benchmark/dashboard-construct.ts`** — a CloudWatch dashboard that puts side-by-side:
    - $ / GB ingested per backend (AWS-derived for CH/OS via Cost Explorer + tags; Datadog from the calculator)
    - $ / GB stored per backend
    - p50/p95/p99 query latency per backend per query (from `query-bench`)
    - bytes scanned per query per backend (where reported)
    - storage growth on the ClickHouse S3 prefix (S3 Storage Lens) and OpenSearch EBS volumes
    - OTel Collector exporter queue depth + retry rates per exporter (parity check)
66. **Run lifecycle** — a `run-id` is a UUID + RFC3339 timestamp + dataset name. The platform writes the active `run-id` to `/platform/<env>/benchmark/run-id` via a small `npm run benchmark:start` script that also stamps the resource tags via an aspect. `npm run benchmark:stop` clears the parameter and snapshots the dashboard to PNG into `benchmark-results`. Each `run-id` is written to a separate prefix under both the data buckets and the results bucket so individual runs are isolatable and deletable. **Cleanup of an old run** is a dedicated `scripts/benchmark-cleanup.ts` that, given a `run-id`, drops the ClickHouse prefix, deletes the OpenSearch indices for that run, and deletes the Datadog logs via API; the targets themselves remain.
67. **Quality gate** — `query-bench` records a result fingerprint per query (sorted hash of result rows). A nightly job compares fingerprints across backends; mismatches open a GitHub issue. Cost without parity is meaningless; parity failures invalidate the run.

### Phase 13 — Docs
68. `README.md`:
    - **Architecture** — short diagram of stacks, capabilities, tenants, extension contract.
    - **Quickstart**: `cp .envrc.example .envrc && direnv allow` → `npm run setup` → `npm run deploy:bootstrap` (one-time, local creds) → set GH var `AWS_DEPLOY_ROLE_ARN_DEV` → push.
    - **Onboarding a tenant** — add an entry to `tenants[]`, open PR, review the `cdk diff`, merge; SSM parameters become available for the app pipeline.
    - **App-side integration guide** — how an application repo reads SSM, assumes its deploy role, and runs feature-branch previews (template GitHub workflow snippet).
    - **Benchmark playbook** — starting/stopping a run, interpreting the dashboard, updating `datadog-rate-card.json`, cleaning up a run-id, parity-failure triage.
    - **How to add a capability** — implement `PlatformAddon`, add a `Capability` variant, register factory, write tests.
    - **Operating playbook** — common workflows, drift response, rollback, incident links.
    - **Versioning** — the `Tenant` config shape and SSM contract are the platform's public API; breaking changes require a changeset entry.

## Relevant files (to be created)
- `bin/app.ts`
- `config/{schema,types,environments,index}.ts`
- `lib/constructs/network/network-construct.ts`
- `lib/constructs/platform/{workload-platform-construct,platform-addon}.ts`
- `lib/constructs/storage/app-storage-construct.ts`
- `lib/constructs/tenants/{tenant-construct,namespace-reaper}.ts`
- `lib/constructs/cleanup/k8s-loadbalancer-cleanup.ts` (+ Lambda handler under `cleanup/handler/`)
- `lib/constructs/pipeline/github-oidc-construct.ts`
- `lib/observability/platform-alarms.ts`
- `lib/stacks/{bootstrap,network,storage,eks,tenants,benchmark-results}-stack.ts`
- `lib/constructs/benchmark/{billing-calculator,dashboard-construct,tagging-aspect}.ts` (+ Lambda handler under `benchmark/handler/`)
- `lib/constructs/platform/storage-classes.ts`
- `config/datadog-rate-card.json` (+ sibling `datadog-rate-card.md` with source URL and capture date)
- `scripts/benchmark-cleanup.ts`
- `test/{config,storage-stack,eks-stack,tenants-stack,github-oidc,cleanup-handler,contract,benchmark-tagging}.test.ts` + `test/snapshots/`
- `integ/` (manual-trigger integ tests)
- `.github/workflows/{pr,deploy,drift}.yml`
- `package.json`, `tsconfig.json`, `cdk.json`, `.nvmrc`, `.envrc.example`, `.gitignore`, `README.md`, `CODEOWNERS`, `renovate.json`
- `scripts/check-tools.sh`, optional `Makefile` (thin aliases)

## Verification
1. `npm test` passes locally and gates CI; coverage includes config validation, IRSA + OIDC trust scoping, S3 hardening, tenant scoping, contract snapshot, and stack snapshots.
2. `npm run deploy:bootstrap` then `npm run deploy:dev`; second run shows zero diff.
3. `kubectl get nodes` Ready; the demo tenant's IRSA pod can `aws s3 ls s3://<bucket>/<tenant>/` and is denied on other tenants' prefixes and on any other bucket.
4. SSM contract: `aws ssm get-parameters-by-path --path /platform/dev/tenants/<name>` returns the documented set.
5. From the app repo, the per-tenant deploy role can be assumed via OIDC and used to push to the tenant's ECR + `helm upgrade` into the tenant's namespace; it cannot deploy outside that namespace or to another tenant.
6. `cdk destroy dev-EksStack` removes only EKS; the cleanup custom resource drains LBs cleanly; Network/Storage/Tenants remain.
7. `cdk destroy dev-TenantsStack` removes tenant resources without touching the cluster.
8. `ENV_NAME=prod npm run deploy` from a workstation fails authentication, confirming the IAM trust policy is correctly scoped.
9. PR with a stack change posts a `cdk diff` comment; merging is blocked until the diff comment exists and CI is green.
10. Drift workflow against an out-of-band change opens an issue within 24h.
11. Type test: authoring a `prod` env entry with `removalPolicy: 'DESTROY'` fails `tsc`.
12. `NamespaceReaper`: a namespace labeled `platform.example.com/ttl` in the past is deleted within the CronJob's interval.
13. **Benchmark tagging invariant**: `npm test` fails if any taggable resource produced by a benchmark tenant is missing `benchmark`, `target`, or `run-id` tags.
14. **End-to-end benchmark smoke** (manual): `npm run benchmark:start` → `loadgen` replays the corpus → within 5 minutes, `Platform/Benchmark/*/BytesIngested` is non-zero for all three exporters and `query-bench` results land in the `benchmark-results` bucket queryable from Athena.
15. **Parity check**: result fingerprints for the canonical query set match between ClickHouse and OpenSearch (Datadog excluded where its query language can't express the exact predicate); a deliberate fingerprint mismatch opens an issue.

## Decisions
- **Platform vs application split**: this repo is the platform; application deploys live in app repos. The seam is a small SSM-published contract plus per-tenant IAM roles.
- **Multi-tenant from day one**: `tenants[]` array in env config; one demo tenant in `dev` exercises the contract.
- **Feature-branch previews**: owned by app pipelines, not the platform. The platform provides primitives (namespaces, IRSA, S3 prefix grants, quotas, TTL reaper). Platform CDK does not redeploy on app PRs.
- **Operator UX**: typed configuration, npm scripts for the common path, and a Lambda-backed custom resource for cluster cleanup.
- **Safety**: enforced by IAM trust policies and GitHub Environment protections.
- **Configuration**: schema-validated (Zod) discriminated union; intent-shaped sizing; invariants encoded in types.
- **Constructs**: domain-shaped (`WorkloadPlatform`, `AppStorage`, `Tenant`, `DeployPipeline`) with a `PlatformAddon` extension contract for new capabilities.
- **Testing**: assertion, snapshot, contract, and Lambda-handler unit tests gate merges; integ tests run on manual trigger.
- **CI/CD feedback**: PR-comment diffs and scheduled drift detection.
- **Observability**: alarms defined alongside the resources they watch.
- **OIDC trust** is covered by tests so trust-policy regressions are caught at PR time.
- **Stack split** (`Bootstrap`/`Network`/`Storage`/`Eks`/`Tenants`) gives blast-radius isolation and per-stack teardown.
- **Account topology**: each env carries its own `aws.account`/`aws.region`; the same model works for single- or multi-account.
- **Benchmark targets are tenants**: ClickHouse, OpenSearch, the OTel Collector, the load generator, and the query bench are all just `Tenant` entries with `capacityOverrides` and a `benchmark` block. Nothing special-cased in the cluster.
- **One Collector, three exporters**: a single OTel Collector pipeline fans out to ClickHouse, OpenSearch, and Datadog so the input-side variables (batch size, attribute set, retries) are identical across backends. The Collector config is the experiment definition.
- **Datadog cost is calculated, not measured**: a hand-maintained `datadog-rate-card.json` is multiplied by Collector-observed volumes and cardinality. The rate card is reviewed in PRs; runs note the rate-card commit hash.
- **Run isolation**: every run gets a UUID `run-id` tag and prefix; data buckets, results bucket, and Datadog log indices partition on it. Cleanup deletes data per run, not per target.
- **Out of scope today**: deploying nonprod/prod, the application repositories themselves, RDS, custom domain/ACM, CDK Pipelines, leased dev-cluster pool for platform-change previews, multi-region benchmark replication, ingest-side fairness across non-OTel-native protocols (Datadog Agent, Beats), Datadog APM trace ingestion (logs first — traces and metrics are a follow-up).

## Further considerations
1. **Deploy role permissions**: start Admin scoped by `aws:RequestedRegion`, then tighten using CloudTrail-derived least-privilege after first stable deploy.
2. **Cluster admin access**: deploy role + a named human-operator role wired via `cluster.awsAuth.addMastersRole`; humans authenticate via SSO. Per-tenant app-CI roles are bound only to their namespace via Kubernetes RBAC (`Role` + `RoleBinding`), not `cluster-admin`.
3. **Bootstrap lifecycle**: `BootstrapStack` deployed once per account with local creds; subsequent self-mutation possible via CDK Pipelines (roadmap).
4. **Upgrade obligations**: Renovate handles CDK/Action versions; K8s minor-version bumps are quarterly maintenance with a documented runbook.
5. **Public-API discipline**: the `Tenant` shape and the SSM parameter contract are consumed by application repositories; breaking changes require a changeset entry and a deprecation window.
6. **Platform-change previews**: today, platform changes are validated by tests + manual integ runs; if pain emerges, add a leased pool of pre-warmed dev clusters as a follow-up.
7. **Benchmark fairness**: results are only meaningful at iso-input (same Collector, same dataset, same attributes). Cross-backend comparisons should always cite the `run-id` and the rate-card commit. A future improvement is replaying multiple datasets (CLF today; structured JSON logs and trace data next) so cost ratios aren't an artifact of one workload.
8. **Datadog egress costs**: Collector → Datadog traffic crosses the NAT gateway. Tag the NAT data-processing line items so the Datadog-side cost includes egress, not only the rate-card.
9. **OpenSearch storage tiering**: today gp3 hot only. UltraWarm/cold-via-S3-snapshot is a follow-up that would materially change the OpenSearch storage cost line and should be a separate run-id, not retrofitted.
