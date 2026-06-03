# Plan: AWS CDK EKS + S3 + IRSA Platform

A TypeScript CDK platform that provisions EKS (managed node groups), an S3 bucket, and an IRSA-backed service account scoped to that bucket. Designed as a small, testable software product — not a pile of scripts. Resources are composed into domain-shaped constructs (`WorkloadPlatform`, `AppStorage`, `DeployPipeline`) behind an explicit extension contract. Environment config is a parsed, schema-validated domain model that uses the type system to make invalid combinations (e.g. prod + `DESTROY`) unrepresentable. Safety rails for destructive operations live in IAM and GitHub Environment protections — not honor-system shell flags. CI/CD is GitHub Actions via OIDC with PR-comment diffs, scheduled drift detection, and CDK assertion + snapshot tests gating merges. Only `dev` deploys today; `nonprod`/`prod` configs exist but are not yet wired to deploy.

## Steps

### Phase 1 — Project scaffolding
1. `cdk init app --language typescript`; pin Node/CDK; ESLint + Prettier; strict `tsconfig`; `.nvmrc`, `.gitignore`.
2. Add Renovate (or Dependabot) config for CDK, addons, GH Actions versions.
3. Add CODEOWNERS: `config/` and `lib/stacks/` reviewed separately from `lib/constructs/` (different blast radius).
4. Add `.envrc.example` (direnv): `AWS_PROFILE`, `AWS_REGION`, `ENV_NAME`.

### Phase 2 — Environment config as a parsed domain model
5. `config/schema.ts` — Zod schemas for env config. Parsed once in `bin/app.ts`; bad config fails synth with a precise error.
6. `config/types.ts` — discriminated union making invariants type-enforced:
   - `type EnvProfile = DevProfile | NonprodProfile | ProdProfile`
   - `ProdProfile` requires `removalPolicy: 'RETAIN'`, `autoDeleteObjects: false`, `terminationProtection: true`, `humanDeployAllowed: false`
   - Common fields: `name`, `aws: { account, region }`, `tags`, `github: { owner, repo, branches[] }`
   - **Intent-shaped** sizing rather than implementation: `workload: { size: 's'|'m'|'l', capacity: 'spot'|'on-demand' }`. Constructs map intent → instance types in one place, so AWS instance-family churn touches one file.
   - `capabilities: Capability[]` — typed list (`{ kind: 's3-bucket' }`, future: `{ kind: 'redis', size: 's' }`); the `EksStack` iterates and attaches matching addons. This is the explicit extension seam.
7. `config/environments.ts` — frozen map of envs (`dev`, `nonprod`, `prod`); type-checked against the union. Authoring `prod` with `removalPolicy: 'DESTROY'` fails `tsc`.
8. `config/index.ts` — `loadEnvConfig(name)` parses with Zod, returns the typed profile, throws structured errors.

### Phase 3 — Domain-shaped constructs (`lib/constructs/`)
Constructs describe capabilities the platform offers, not AWS service categories.
9. `network/network-construct.ts` — VPC, public + private-with-egress subnets, EKS subnet tags.
10. `platform/workload-platform-construct.ts` — EKS cluster + node group + core addons (VPC CNI, CoreDNS, kube-proxy, EBS CSI, AWS LBC via Helm) + IRSA OIDC provider. Exposes typed methods: `addServiceAccount`, `grantClusterAdmin`, `attach(addon)`.
11. `platform/platform-addon.ts` — explicit extension contract:
    ```ts
    interface PlatformAddon {
      readonly id: string;
      attach(platform: WorkloadPlatform, env: EnvProfile): void;
    }
    ```
    New capability = implement this + register a factory keyed by `Capability.kind`. `EksStack` does not change.
12. `storage/app-storage-construct.ts` — bucket (SSE, BPA, versioning, removal policy / autoDelete from config) + `bindToServiceAccount(sa, access)` for narrow IAM grants. Implements `PlatformAddon`. Prod attaches a bucket policy denying `s3:DeleteBucket`/`s3:DeleteObject*` outside the deploy role.
13. `cleanup/k8s-loadbalancer-cleanup.ts` — CDK custom resource (Lambda + provider framework) that, on stack delete, deletes k8s LoadBalancer Services / Ingresses and waits for ELB drain. Replaces the bash pre-destroy hook: owned, version-controlled, unit-testable, observable in CFN events.
14. `pipeline/github-oidc-construct.ts` — `iam.OpenIdConnectProvider` + deploy role with trust scoped to `repo:{owner}/{repo}:ref:refs/heads/{branch}`. Prod role's trust additionally requires the GitHub Environment claim (`environment:prod`) so only approved Actions runs can assume.

### Phase 4 — Stack composition (blast-radius isolation)
15. `lib/stacks/bootstrap-stack.ts` — `GithubOidcConstruct`. One per account.
16. `lib/stacks/network-stack.ts` — `NetworkConstruct`. Exposes `vpc`.
17. `lib/stacks/storage-stack.ts` — `AppStorageConstruct`. Exposes `bucket`.
18. `lib/stacks/eks-stack.ts` — `WorkloadPlatformConstruct` + iterates `env.capabilities` and calls `platform.attach(addonFor(cap))`. Includes `K8sLoadBalancerCleanup` custom resource.
19. All stacks: apply tags from config, set `terminationProtection`, env-prefixed names (`dev-EksStack`).
20. `bin/app.ts` — `loadEnvConfig(process.env.ENV_NAME)` then instantiate stacks.

### Phase 5 — Tests (gate CI on these)
21. `test/config.test.ts` — every env round-trips through Zod; discriminated-union exhaustiveness check.
22. `test/storage-stack.test.ts` — encryption on, public access blocked, removal policy + autoDelete match config; prod has the delete-deny bucket policy.
23. `test/eks-stack.test.ts` — node group instance type derives correctly from intent sizing; IRSA role's trust `Condition` restricts `sub` to the exact `system:serviceaccount:<ns>:<sa>` (security boundary).
24. `test/github-oidc.test.ts` — trust restricts `repo:` AND `ref:`; prod role additionally requires `environment:prod` claim. Prevents the most common OIDC misconfig ("any repo can deploy").
25. `test/snapshots/*.snap` — per-stack snapshots; CI surfaces unintended template diffs.
26. `test/cleanup-handler.test.ts` — unit tests for the cleanup Lambda handler (mock AWS SDK + kube client).
27. `integ/` — `@aws-cdk/integ-tests-alpha` against a sandbox account on manual-trigger workflow only (slow/expensive).

### Phase 6 — Safety via IAM & GitHub Environments (not honor-system flags)
28. **Prod deploy role unassumable by humans.** Trust policy lists only the GitHub OIDC provider with `environment:prod` claim. Local `ENV_NAME=prod cdk deploy` simply fails to authenticate — safety enforced by AWS, not bash.
29. **GitHub Environment `prod`** requires reviewer approval and restricts to `main`.
30. **Prod S3 bucket policy** denies destructive actions to any principal except the deploy role.
31. **CFN `terminationProtection: true`** on prod stacks (from config).
32. The Makefile contains *no* `I_KNOW_WHAT_IM_DOING` or typed-name confirmations. They're unnecessary because the destructive path doesn't exist for unauthorized callers.

### Phase 7 — Operator UX: thin & typed
33. `package.json` scripts cover the common path: `build`, `test`, `lint`, `synth`, `diff`, `deploy`, `destroy`, `kubeconfig`. Each reads `ENV_NAME` from env (direnv) or accepts `--env`.
34. Optional: a thin Node CLI (`commander`, typed args, unit-tested) only if scripting needs grow beyond what npm scripts cleanly express. Skip for now (YAGNI).
35. `Makefile` reduced to ~10 lines of two-line aliases over npm scripts (or omitted). No safety logic in Make.
36. `scripts/check-tools.sh` — small, single-purpose: verify `node`, `aws`, `cdk`, `kubectl` versions on `npm run setup`.

### Phase 8 — CI/CD with feedback loops
37. `.github/workflows/pr.yml`:
    - `npm ci && npm run lint && npm test` — gate merge.
    - OIDC assume read-only role; `cdk synth` + `cdk diff`; **post diff as a PR comment**. Branch protection requires the diff comment + green CI before merge.
38. `.github/workflows/deploy.yml`:
    - Push `main` → deploy `dev` automatically.
    - `workflow_dispatch` with environment input → `nonprod`/`prod` (gated by GH Environment approval).
    - Steps: checkout → setup-node → `npm ci` → `npm test` → OIDC assume per-env deploy role → `cdk deploy --all --require-approval never`.
39. `.github/workflows/drift.yml` — scheduled daily `cdk diff` against deployed stacks per env; opens an issue on drift. "If `main` doesn't match production, that's a bug."
40. **CDK Pipelines** as a roadmap follow-up: self-mutating pipeline so deploy infra is itself code under the same review process.

### Phase 9 — Observability as a first-class deliverable
41. `lib/observability/platform-alarms.ts` — CloudWatch alarms declared alongside the resources they watch:
    - EKS control plane error rate, node group `NotReady` count
    - IRSA token issuance failures (CloudTrail metric filter on `sts:AssumeRoleWithWebIdentity` failures)
    - S3 4xx/5xx rates on the app bucket
    - CFN `UPDATE_ROLLBACK` events → SNS topic
42. SNS topic per env (subscriber config out of scope today; topic + policy in code).
43. Rule: a new construct ships with its alarms, or it's not done.

### Phase 10 — Docs
44. `README.md`:
    - **Architecture** — short diagram of stacks, capabilities, extension contract.
    - **Quickstart**: `cp .envrc.example .envrc && direnv allow` → `npm run setup` → `npm run deploy:bootstrap` (one-time, local creds) → set GH var `AWS_DEPLOY_ROLE_ARN_DEV` → push.
    - **How to add a capability** — implement `PlatformAddon`, add a `Capability` variant, register factory, write tests.
    - **Operating playbook** — common workflows, drift response, rollback, incident links.
    - **Versioning** — config shape is the public API to operators; breaking changes require a changeset entry.

## Relevant files (to be created)
- `bin/app.ts`
- `config/{schema,types,environments,index}.ts`
- `lib/constructs/network/network-construct.ts`
- `lib/constructs/platform/{workload-platform-construct,platform-addon}.ts`
- `lib/constructs/storage/app-storage-construct.ts`
- `lib/constructs/cleanup/k8s-loadbalancer-cleanup.ts` (+ Lambda handler under `cleanup/handler/`)
- `lib/constructs/pipeline/github-oidc-construct.ts`
- `lib/observability/platform-alarms.ts`
- `lib/stacks/{bootstrap,network,storage,eks}-stack.ts`
- `test/{config,storage-stack,eks-stack,github-oidc,cleanup-handler}.test.ts` + `test/snapshots/`
- `integ/` (manual-trigger integ tests)
- `.github/workflows/{pr,deploy,drift}.yml`
- `package.json`, `tsconfig.json`, `cdk.json`, `.nvmrc`, `.envrc.example`, `.gitignore`, `README.md`, `CODEOWNERS`, `renovate.json`
- `scripts/check-tools.sh`, optional `Makefile` (thin aliases)

## Verification
1. `npm test` passes locally and gates CI; coverage includes config validation, IRSA + OIDC trust scoping, S3 hardening, snapshot diffs.
2. `npm run deploy:bootstrap` then `npm run deploy:dev`; second run shows zero diff.
3. `kubectl get nodes` Ready; IRSA pod can `aws s3 ls s3://<bucket>` and is denied on any other bucket.
4. `cdk destroy dev-EksStack` removes only EKS; the cleanup custom resource drains LBs cleanly; Network/Storage remain.
5. `ENV_NAME=prod npm run deploy` from a workstation **fails to authenticate** — proves IAM-enforced safety rail.
6. PR with a stack change posts a `cdk diff` comment; merging is blocked until the diff comment exists and CI is green.
7. Drift workflow against an out-of-band change opens an issue within 24h.
8. Type test: authoring a `prod` env entry with `removalPolicy: 'DESTROY'` fails `tsc`.

## Decisions
- **Operator UX is a product, not a recipe**: typed config + thin npm scripts + tested Lambda for cleanup, not Make + bash + env-var safety flags.
- **Safety lives in IAM and GH Environments**, not honor-system shell guards.
- **Config is parsed and type-encoded**: discriminated union + Zod, intent-shaped sizing, illegal combos unrepresentable.
- **Constructs are domain-shaped** (`WorkloadPlatform`, `AppStorage`, `DeployPipeline`) with an explicit `PlatformAddon` extension contract.
- **Tests gate merges**: assertion, snapshot, and Lambda-handler unit tests in CI; integ tests on manual trigger.
- **Feedback loops in CI/CD**: PR-comment diffs, scheduled drift detection.
- **Observability ships with the resource**, not later.
- **OIDC trust is tested** because misconfig = "any repo can deploy."
- **Stack split** (`Bootstrap`/`Network`/`Storage`/`Eks`) for blast-radius isolation and partial teardown via `cdk destroy <stack>`.
- **Multi-account ready**: each env carries `aws.account`/`aws.region`; same values = single account.
- **Out of scope today**: deploying nonprod/prod, app workloads beyond LBC, RDS, custom domain/ACM, full CDK Pipelines (roadmap).

## Further considerations
1. **Deploy role permissions**: start Admin scoped by `aws:RequestedRegion`, then tighten using CloudTrail-derived least-privilege after first stable deploy.
2. **Cluster admin access**: deploy role + a named human-operator role wired via `cluster.awsAuth.addMastersRole`; humans authenticate via SSO.
3. **Bootstrap lifecycle**: `BootstrapStack` deployed once per account with local creds; subsequent self-mutation possible via CDK Pipelines (roadmap).
4. **Upgrade obligations**: Renovate handles CDK/Action versions; K8s minor-version bumps are quarterly maintenance with a documented runbook.
5. **Public-API discipline**: changes to `EnvironmentConfig` shape require a changeset entry — operators consume this type.
