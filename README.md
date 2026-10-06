# FlexGate on US staging: step by step

**Goal:** the bare minimum, one (org, model) served natively by FlexGate on **US staging only**, with correct billing.
**Scope:** US staging only (cluster `k8s-backoffice-gcp-1`, env `INTERNAL-PROD`, namespace `token-service`). EU staging stays off. Dev is out of scope.
**Last updated:** 2026-10-06 08:15 UTC. Status is from read-only checks of the live cluster and infra `main`.

**Status key:** DONE, NOT STARTED, BLOCKED, UNVERIFIED (I could not see it).

## Where staging is today

| Item | State |
|---|---|
| Staging release | RC `2026.10-W41-r2`: chart `0.4.0-rc.fdb4e31a`, db-migration `0.1.440-fdb4e31a`, Ready |
| FlexGate code (#2192, #2321 and the rest) | In the RC, on staging, switched off |
| Migrations 439 and 440, gateway tables | Applied (`gateway_usage_events`, `gw_changelog`, `gw_lease_ledger` exist) |
| LiteLLM | 4/4 on image `c247f28b`, which has `/app/hooks/gw_fence.py` (the v4 hook file is present) |
| Worker | 1/1 Running, 0 restarts |
| Gateway pool nodes | 4 exist |
| `flexgateMode` for staging in infra | **Unset (off)** |
| FlexGate workloads, `redis-gw`, `usage-ingest` | **None** |
| Database roles `flexgate_usage`, `flexgate_usage_login` | **Missing** |
| Open infra PRs for staging FlexGate | None (#6430 is US dev only) |

## The steps

### Step 1: Ops prerequisites (before any PR). NOT STARTED
Needs people with database and OpenBao access; I cannot see or do these.
- [ ] Create the `flexgate_usage` role and `flexgate_usage_login` on the staging database, then run the grant (infra `docs/onboarding-new-gke-cluster.md`, step 8b). **Verified missing today.**
- [ ] Seed OpenBao: the usage database URL (`.../token-service/flexgate`, key `USAGE_DATABASE_URL`), the abort token (`.../token-service/flexgate-abort`, key `ABORT_TOKEN`), and the `redis-gw` password. UNVERIFIED.
- [ ] Decide the capacity and admission numbers (start from dev's block, smaller).

### Step 2: Infra PR 1, staging `ready`. NOT STARTED
- [ ] Set `flexgateMode: "ready"` on staging and add the staging FlexGate values block: `activeColor: a` (a=1, b=0), Pub/Sub names, service accounts, `usageIngest.enabled: true`, `redisGw`, `networkPolicy.edgeNamespaces`, `vault.cluster`, cohort (your org), plane models (your model), chat route certified.
- [ ] Merge. Flux then renders `redis-gw`, `redis-ratelimit`, their secrets, the capacity buffer, the usage-log guard and the alerts (already declared for staging, gated by the mode).

### Step 3: Pulumi apply for the staging Pub/Sub stack. NOT STARTED
- [ ] `pulumi up` after Step 2 merges. The stack creates nothing until staging is `ready`.
- [ ] Check: `flexgate-a` Ready, `flexgate_snapshot_age_seconds` under 30, `redis-gw` 3/3, `usage-ingest` running.

### Step 4: Infra PR 2 and PR 3, the arm-cap store. NOT STARTED
- [ ] PR 2: `armSteerStore: "dual"`. Wait for the LiteLLM rollout to finish.
- [ ] PR 3: `armSteerStore: "redis-gw"`.

### Step 5: Infra PR 4, mode `on` plus color `b`. NOT STARTED
- [ ] `flexgateMode: "on"` and `colors.b.replicas: 1`. `on` turns on the LiteLLM fence, and `b` is the only pod that picks up the new template that certifies chat.
- [ ] Chat weight above 0 is required for certification. Plan: 100 for the test window (the cohort limits FlexGate to your org; everyone else is relayed to LiteLLM). Coordinate with QA first.

### Step 6: Infra PR 5, switch color. NOT STARTED
- [ ] After `flexgate-b` is 1/1 Ready: `activeColor: b`.

### Step 7: Operator steps. NOT STARTED
- [ ] Confirm every LiteLLM pod reports the v4 fence hook (runbook, "LiteLLM fence version", Rule 1). File present today; the metric check comes after the fence turns on.
- [ ] Fence switch for the org and model: dry run, then `--confirm` (`python3 -m scripts.flexgate_fence_switch --org-id <org> --model <model> --to flexgate`, from a portal-api pod).

### Step 8: Test. NOT STARTED
- [ ] Send chat requests, streaming and non-streaming.
- [ ] Pass if: a `gateway_usage_events` row appears per request, the balance moves by the same amount LiteLLM would charge, there are no false 402s, and a client disconnect is billed correctly.

**Rollback at any point:** `flexgateMode: "ready"` returns every path to LiteLLM. Move a model back first with the same script (`--to litellm`).

## Token-service PRs
**None needed for the bare minimum.** The code is on main and in RC r2. If the test turns up a bug, fix only that one, cut a new RC and re-test.

## Needed from you
- [ ] The org id and the model.
- [ ] Choice of weight for the test window (100 recommended).

## Not verified
- OpenBao contents and Cloud SQL role creation rights.
- Whether FlexGate can reach the model's engine from staging (I'll check once the model is known).
- The `gw_fence_hook_version` metric on every LiteLLM pod (only meaningful once the fence is on).
- Gateway pool capacity for the numbers chosen in Step 1.
