# Plan: AWS CDK EKS + S3 + IRSA with GitHub Actions

Bootstrap a TypeScript CDK project that provisions EKS (managed node groups), an S3 bucket, and an IRSA service account scoped to that bucket. Resources live in reusable L3 constructs and are split across multiple stacks (`Bootstrap`, `Network`, `Storage`, `Eks`) so partial teardown is a one-liner. A typed env-config layer supports single- or multi-account setups; only `dev` deploys today. GitHub Actions deploys via OIDC. A `Makefile` (plus mirroring npm scripts and a `.envrc.example`) gives repeatable local setup, deploy, and teardown — including a pre-destroy hook that drains k8s load balancers.

## Steps

### Phase 1 — Project scaffolding
1. `cdk init app --language typescript`; pin Node/CDK; ESLint + Prettier; strict `tsconfig`; `.nvmrc`, `.gitignore`.
2. Add `.envrc.example` (direnv) with `AWS_PROFILE`, `AWS_REGION`, `ENV_NAME`.

### Phase 2 — Environment config layer (extensibility backbone)
3. `config/types.ts` — `EnvironmentConfig` interface:
   - `name: 'dev' | 'nonprod' | 'prod'`
   - `aws: { account, region }` (per-env → single OR multi-account)
   - `eks: { version, endpointAccess, nodeGroup: { instanceTypes, min/desired/maxSize, diskSize, capacityType } }`
   - `s3: { versioned, removalPolicy, autoDeleteObjects }`
   - `terminationProtection: boolean`
   - `tags: Record<string,string>`
   - `github: { owner, repo, branches[] }`
4. `config/environments.ts` — map of envs:
   - `dev`: t3.small SPOT, `removalPolicy=DESTROY`, `autoDeleteObjects=true`, `terminationProtection=false`
   - `nonprod`: similar to dev but ON_DEMAND
   - `prod`: m5.large ON_DEMAND, `removalPolicy=RETAIN`, `autoDeleteObjects=false`, `terminationProtection=true`
5. `config/index.ts` — `getEnvConfig(name)` helper.

### Phase 3 — Reusable L3 constructs (`lib/constructs/`)
6. `network-construct.ts` — VPC, public + private-with-egress subnets, EKS subnet tags.
7. `eks-cluster-construct.ts` — `eks.Cluster` (KubectlV31 layer), IRSA enabled, managed node group from config, add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI, AWS Load Balancer Controller via Helm).
8. `s3-bucket-construct.ts` — SSE-S3, block public access, versioning + removal policy + autoDeleteObjects from config.
9. `irsa-construct.ts` — generic: cluster, namespace, SA name, policy attachments. For this app: `bucket.grantReadWrite(sa.role)`.
10. `github-oidc-construct.ts` — `iam.OpenIdConnectProvider` + deploy role trusted by `repo:{owner}/{repo}:ref:refs/heads/{branch}`.

### Phase 4 — Multi-stack composition (enables partial teardown)
11. `lib/stacks/bootstrap-stack.ts` — `GithubOidcConstruct`. Deployed once per account.
12. `lib/stacks/network-stack.ts` — `NetworkConstruct`. Exposes `vpc`.
13. `lib/stacks/storage-stack.ts` — `S3BucketConstruct`. Exposes `bucket`.
14. `lib/stacks/eks-stack.ts` — `EksClusterConstruct` (takes `vpc`) + `IrsaConstruct` (takes `cluster`, `bucket`).
15. All stacks accept `EnvironmentConfig`; apply `Tags.of(this).add(...)` and `terminationProtection` from config; stack names prefixed with env (e.g. `dev-EksStack`).
16. `bin/app.ts` — reads `ENV_NAME` (default `dev`), instantiates Network → Storage → Eks (Bootstrap is its own deploy). Cross-stack refs by direct property passing within one `cdk.App`.

### Phase 5 — GitHub Actions
17. `.github/workflows/deploy.yml` — push `main` deploys dev; `workflow_dispatch` with environment input; `id-token: write`; `aws-actions/configure-aws-credentials@v4` OIDC; `cdk deploy --all`. GitHub Environments for approval gates + per-env role ARN.
18. `.github/workflows/pr.yml` — `cdk synth` + `cdk diff` on PRs (read-only role).

### Phase 6 — Local orchestration: Makefile + scripts
19. `Makefile` (default target = `help`, self-documenting via `## comment` on each target). Variables: `ENV ?= dev`, `CONFIRM ?=`, `FORCE ?=`, `I_KNOW_WHAT_IM_DOING ?=`. Targets:
    - `help` — list targets with descriptions
    - `setup` — runs `check-tools` → `npm ci` → prints next steps
    - `check-tools` — verifies `node`, `npm`, `aws`, `cdk`, `kubectl`, `jq`; prints install hints on miss
    - `bootstrap` — `cdk bootstrap` for the env's account/region (idempotent)
    - `deploy-bootstrap` — `cdk deploy <env>-BootstrapStack` (one-time per account)
    - `synth` / `diff` / `deploy` — wrap `cdk` for app stacks, scoped by `ENV`
    - `kubeconfig` — runs `aws eks update-kubeconfig`
    - `destroy` — destroys app stacks (Eks, Storage, Network) with confirmation; runs pre-destroy hook first
    - `destroy-eks` — destroys only `<env>-EksStack` (after pre-destroy hook)
    - `destroy-storage` — destroys only `<env>-StorageStack` (refuses if `removalPolicy=RETAIN` unless `FORCE=1`)
    - `destroy-network` — destroys only `<env>-NetworkStack`
    - `nuke` — destroys all including `BootstrapStack`; requires typed env name confirmation
    - `clean` — removes `cdk.out/`, `node_modules/`
    Safety: any destroy of `prod` requires `I_KNOW_WHAT_IM_DOING=1`; non-interactive runs require `CONFIRM=yes`.
20. `scripts/check-tools.sh` — version checks + helpful errors.
21. `scripts/pre-destroy.sh ENV` — runs `kubectl delete svc,ingress --all-namespaces` for LoadBalancer/Ingress resources, polls until ELBs/NLBs/ALBs are gone (timeout w/ clear message). No-op if cluster unreachable.
22. `scripts/confirm.sh PROMPT EXPECTED` — typed-confirmation helper used by `nuke` and prod destroys.
23. `package.json` scripts mirror common Make targets (`build`, `synth`, `deploy:dev`, `destroy:dev`) for those preferring `npm run`.

### Phase 7 — Docs
24. `README.md`:
    - **Quickstart**: `cp .envrc.example .envrc && direnv allow` → `make setup` → `make deploy-bootstrap` (one-time, local creds) → set GH var `AWS_DEPLOY_ROLE_ARN_DEV` → push to `main`.
    - **Common workflows**: deploy, diff, kubeconfig, partial teardown matrix.
    - **Teardown order & gotchas**: drain LBs first, S3 autoDelete, terminationProtection on prod.

## Relevant files (to be created)
- `bin/app.ts`
- `config/types.ts`, `config/environments.ts`, `config/index.ts`
- `lib/constructs/{network,eks-cluster,s3-bucket,irsa,github-oidc}-construct.ts`
- `lib/stacks/{bootstrap,network,storage,eks}-stack.ts`
- `.github/workflows/{deploy,pr}.yml`
- `Makefile`
- `scripts/{check-tools,pre-destroy,confirm}.sh`
- `package.json`, `tsconfig.json`, `cdk.json`, `.nvmrc`, `.envrc.example`, `.gitignore`, `README.md`

## Verification
1. `make setup` on a fresh clone succeeds end-to-end on macOS/Linux.
2. `make synth ENV=dev` produces clean templates for all four stacks.
3. `make deploy-bootstrap` then `make deploy ENV=dev`; second `make deploy` shows no changes (idempotent).
4. `kubectl get nodes` shows nodes Ready; IRSA test pod can `aws s3 ls s3://<bucket>` but not other buckets.
5. `make destroy-eks ENV=dev` removes only the EKS stack; `make synth ENV=dev` after still shows Network/Storage intact.
6. `make destroy ENV=dev` removes all app stacks cleanly; pre-destroy hook removes any LB Services first; S3 bucket is auto-emptied in dev.
7. `make destroy ENV=prod` is blocked without `I_KNOW_WHAT_IM_DOING=1`; `make destroy-storage ENV=prod` is blocked without `FORCE=1`.
8. GH Actions: OIDC assume succeeds, `cdk deploy --all` completes, re-run shows no diff.

## Decisions
- **Compute**: EKS managed node groups (EC2), per-env sizing.
- **Auth**: GitHub OIDC; repo/branch placeholders in config until provided.
- **Extensibility**: reusable L3 constructs + multi-stack composition. New resource = new construct + add to relevant stack (or new stack file).
- **Multi-account ready**: each env carries `aws.account`/`aws.region`; same values = single account.
- **Stack split**: `Bootstrap`, `Network`, `Storage`, `Eks` — enables `cdk destroy <env>-EksStack` for partial teardown without custom scripting.
- **Removal policy**: dev/nonprod = `DESTROY` + autoDelete; prod = `RETAIN` + no autoDelete.
- **Termination protection**: prod only.
- **Local orchestration**: `Makefile` (primary) + npm scripts (mirrored) + `direnv` (`.envrc.example`).
- **Pre-destroy hook**: deletes k8s LB Services/Ingresses and waits for ELB drain before `cdk destroy` of EKS.
- **Out of scope today**: deploying nonprod/prod, app workloads beyond LBC, observability, RDS, custom domain/ACM.

## Further considerations
1. **Deploy role permissions**: Admin (fast) vs scoped (secure). Recommend Admin to start, tighten after first deploy.
2. **Cluster admin access**: GH deploy role + a named human-operator role via `cluster.awsAuth.addMastersRole`.
3. **Bootstrap lifecycle**: `BootstrapStack` deployed once per account with local creds before GH Actions can run — covered by `make deploy-bootstrap` and README.
