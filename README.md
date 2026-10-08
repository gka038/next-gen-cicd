# next-gen-cicd

An AI-native CI/CD stack for a single-node Kubernetes cluster, built entirely
from open-source components and installed GitOps-style with Argo CD. It
gives you:

- **Source control + PR review + CI triggers** — Forgejo, with Forgejo
  Actions and/or Tekton Triggers kicking off pipelines on push.
- **Build/scan/sign** — Tekton Pipelines, Trivy + Grype (vulnerability
  scanning), Syft (SBOMs), cosign via Tekton Chains (signing).
- **Registry** — Harbor, itself backed by its own bundled Trivy scanner.
- **GitOps delivery** — Argo CD (app-of-apps, this whole repo), Argo
  Rollouts (canary/progressive delivery with metric-based auto-rollback),
  optionally Kargo (promotion across stages).
- **AI agent orchestration** — Argo Workflows + KEDA, running both the
  nightly overview agent below and whatever agent workloads the product
  itself uses.
- **Observability** — kube-prometheus-stack, Loki, Tempo, an OpenTelemetry
  Collector (the LGTM stack), plus Langfuse for LLM/agent-specific tracing,
  token and eval observability.
- **Policy + secrets** — Kyverno (e.g. signed-image-only admission),
  External Secrets Operator + Vault.
- **A nightly overview agent** — an open-source coding agent (OpenHands)
  that reads the observability data, open issues, and this repo's own
  manifests, then opens PRs against this repo and/or a product repo through
  the same Forgejo PR path as any human contributor — never auto-merges.

## Repository layout

```
bootstrap/        One-time imperative steps to get Argo CD itself running.
infrastructure/   One Argo CD Application (± Helm values) per platform
                   component, grouped by concern: scm, ci, registry,
                   delivery, workflows, observability, policy, secrets.
apps/             The app-of-apps: a small Helm chart whose values.yaml
                   toggles each infrastructure/ group on or off.
agents/           The nightly overview-agent CronWorkflow, its RBAC, and
                   its prompt.
```

Every `infrastructure/<group>/<component>/application.yaml` is a plain Argo
CD `Application` resource. For Helm-based components it's a multi-source
Application: one source pulls the pinned upstream chart, the other points
back at this repo for the sibling `values.yaml` (the
`helm.valueFiles: [$values/...]` pattern). Tekton has no official chart, so
`infrastructure/ci/tekton/` instead vendors pinned upstream release YAML
plus this repo's own Task/Pipeline/Trigger definitions, applied as a plain
directory source.

## Prerequisites

- A single-node Kubernetes cluster (k3s, kind, k0s, minikube, etc.) with a
  **default StorageClass already configured** — nothing here sets one up.
- `kubectl` and `helm` (v3) pointed at that cluster.
- Enough node capacity for whatever you enable — see **Sizing and
  caveats** below before turning everything on at once.

## Bootstrap order

1. **Install Argo CD** — see `bootstrap/README.md`. This is the one
   imperative step; Argo CD can't GitOps-install itself.
2. **Apply the app-of-apps** — `kubectl apply -n argocd -f apps/root-application.yaml`.
   This points Argo CD at `apps/`, which renders one child `Application` per
   enabled component group, which in turn each render the component(s)
   inside it.
3. Everything from here on is managed by Argo CD syncing this Git
   repository. Configure the secrets each component needs (below) as you
   turn components on.

## Toggling components on/off

Edit `apps/values.yaml`:

```yaml
components:
  scm: true
  ci: true
  registry: true
  delivery: true
  workflows: true
  observability: true
  policy: true
  secrets: true
  agents: true
```

Flip a group to `false`, commit, and Argo CD prunes it on the next sync
(self-heal is on, so it reconciles automatically). The eight infra groups
match `infrastructure/`'s eight subdirectories; `agents` controls the
overview-agent CronWorkflow separately since it depends on `ci`/`workflows`
being present to do anything useful.

Within the `delivery` and `observability` groups, more than one component
shares a toggle (Argo Rollouts + Kargo; kube-prometheus-stack + Loki + Tempo
+ OTel Collector + Langfuse + the ClickHouse Operator it needs). To drop
just one of those — Kargo, say, or Langfuse — delete that component's
subdirectory instead of flipping the whole group off.

## Secrets you need to supply

Nothing in this repo contains a real credential. Every value below is a
placeholder marked `CHANGE_ME_PLACEHOLDER` in the relevant `values.yaml`, or
a `kubectl create secret` command you run by hand (never committed).

| What | Where | How |
|---|---|---|
| Forgejo admin password | `infrastructure/scm/forgejo/values.yaml` (`gitea.admin.password`) | Edit before first sync, or set `gitea.admin.existingSecret` to a Secret you create instead. |
| Harbor admin password + internal DB password | `infrastructure/registry/harbor/values.yaml` | Edit before first sync. |
| Grafana admin password | `infrastructure/observability/kube-prometheus-stack/values.yaml` (`grafana.adminPassword`) | Edit before first sync, or point `grafana.admin.existingSecret` at a Secret instead. |
| Kargo admin password hash + token signing key | `infrastructure/delivery/kargo/values.yaml` | Generate per the comments in that file. |
| cosign signing key (Tekton Chains) | — | `cosign generate-key-pair k8s://tekton-chains/signing-secrets` — see `infrastructure/ci/README.md`. |
| cosign public key (Kyverno admission check) | Secret `cosign-public-key` in `kyverno` namespace | Export from the key above; see `infrastructure/ci/README.md` and `infrastructure/policy/kyverno/policies/`. |
| Harbor robot account for CI pushes | Secret `harbor-push-credentials` in `tekton-pipelines` | `kubectl create secret docker-registry ...` — see `infrastructure/ci/README.md`. |
| Overview agent: Forgejo token + LLM API key | Secret `overview-agent-secrets` in `agents-system` | `kubectl create secret generic ...` — see `agents/overview-agent/README.md`. |
| Vault unseal / init | — | `vault operator init` + `vault operator unseal`, run by hand after first deploy — see **Vault** below. |

If `secrets` (External Secrets Operator + Vault) is enabled, you can instead
keep these in Vault and let ESO sync them into the right namespaces as
`ExternalSecret`/`Secret` pairs, rather than running one-off `kubectl create
secret` commands per component.

### Vault, specifically

Vault deploys in standalone mode (file storage, one pod, one PVC) and starts
**sealed and uninitialized** — that's intentional; nothing auto-unseals it.
After its first sync:

```bash
kubectl exec -n vault vault-0 -- vault operator init -key-shares=1 -key-threshold=1
# save the one unseal key and the root token it prints — nowhere in this repo, obviously
kubectl exec -n vault vault-0 -- vault operator unseal <unseal-key>
```

You'll need to unseal it again after every pod restart (it's single-node,
file-backed, non-HA by design). Then enable Kubernetes auth so External
Secrets Operator can authenticate to it:

```bash
kubectl exec -n vault vault-0 -- vault login <root-token>
kubectl exec -n vault vault-0 -- vault auth enable kubernetes
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"
kubectl exec -n vault vault-0 -- vault policy write external-secrets - <<'EOF'
path "secret/data/*" { capabilities = ["read"] }
EOF
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets ttl=1h
```

If this ceremony is more than you want, skip Vault: point External Secrets
Operator's `ClusterSecretStore` at a SOPS-encrypted file provider instead
(not wired up here, but a one-file change — see
`infrastructure/secrets/external-secrets/extra/clustersecretstore-vault.yaml`
for the shape to replace), or just use `kubectl create secret` directly per
component as in the table above and skip the `secrets` component entirely.

## Reaching each UI

This is a single-node cluster, so there's usually no cloud LoadBalancer.
Every UI defaults to `ClusterIP` with an optional ingress behind
`ingressClassName: nginx` you can flip on in that component's `values.yaml`.

**Port-forward (no extra setup):**

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443        # https://localhost:8080
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80
kubectl -n forgejo port-forward svc/forgejo-http 3001:3000
kubectl -n harbor port-forward svc/harbor-portal 3002:80
kubectl -n observability port-forward svc/langfuse-web 3003:3000
kubectl -n argo-workflows port-forward svc/argo-workflows-server 2746:2746
```

**Ingress (if you'd rather have stable hostnames):** install ingress-nginx
yourself (it's deliberately not part of this repo — it's cluster-wide
infrastructure most single-node distros already ship, e.g. k3s's Traefik or
a `helm install ingress-nginx ingress-nginx/ingress-nginx`), flip
`ingress.enabled: true` in the component(s) you want, and add the `*.local`
hostnames each `values.yaml` uses to your `/etc/hosts` pointing at the
node's IP.

## Sizing and caveats for one node

Every component here is already sized down for one node: single replicas,
trimmed resource requests, monolithic/standalone modes where the chart
offers one (Loki and Tempo single-binary, Vault standalone), and no
hard anti-affinity, required PodDisruptionBudgets, or multi-node topology
spread. A few are still genuinely heavy:

- **Langfuse** is the heaviest single toggle in this repo: it bundles
  Postgres, Valkey, SeaweedFS (S3-compatible storage), *and* a ClickHouse
  Operator managing a ClickHouseCluster + KeeperCluster — five extra
  stateful workloads for LLM/agent trace storage. If detailed LLM tracing
  isn't a priority yet, leave `observability` on but delete
  `infrastructure/observability/langfuse/` and
  `infrastructure/observability/clickhouse-operator/`; you still get
  metrics/logs/traces for the deployed services and agent workloads via
  OTel Collector → Tempo/Prometheus/Loki.
- **kube-prometheus-stack** disables the controller-manager/scheduler/etcd/
  kube-proxy `ServiceMonitor`s by default, because most single-node distros
  (k3s, kind, k0s, minikube) don't expose those endpoints the way a kubeadm
  cluster does — they'd otherwise sit permanently "down". Re-enable the
  ones your distro does expose.
- **Harbor** runs its own internal Postgres + Redis + Trivy adapter rather
  than needing external ones, which is simpler for one node but still
  three extra pods on top of Harbor's own four.
- **Memcached-backed chunk/results caches** for Loki default to several GB
  of memory in upstream's own values and are disabled here — Loki falls
  back to its in-process cache, which is fine at low log volume.

If you're resource-constrained, the toggles most worth leaving off for a
first pass are `secrets` (skip Vault's unseal ceremony, use `kubectl create
secret` instead) and the Langfuse/ClickHouse piece of `observability`.

## What's been validated, and what hasn't

There's no live cluster in this environment to apply against, so validation
here was static:

- `helm lint` and `helm template <chart> -f <values>` against the real
  pinned chart version, for every Helm-based component.
- `kubeconform -strict` (with the datreeio CRDs-catalog schemas for
  Argo CD/Rollouts/Workflows/Kyverno/External Secrets CRDs) against every
  plain manifest — the vendored Tekton release YAML, this repo's own
  Tekton Tasks/Pipelines/Triggers, the Kyverno policy, the agent's
  CronWorkflow/RBAC, and every `Application` resource.
- The app-of-apps chart (`apps/`) renders and its toggles were exercised
  (each `components.*: false` correctly drops that group's `Application`).

What this **can't** verify without a real cluster: that Argo CD actually
reconciles all of this cleanly end-to-end, that the multi-source `$values`
Helm pattern behaves as expected against a live Argo CD (it's the documented
pattern, but untested here), that Tekton Chains' signing flow and the
Kyverno `verifyImages` policy actually agree on a real cosign key, that
Vault's init/unseal/Kubernetes-auth sequence above is typo-free end to end,
and that the overview agent's OpenHands invocation (CLI flags move fast
upstream) still matches the pinned image tag. Treat first-apply on a real
cluster as a smoke test, in roughly bootstrap order, one component group at
a time rather than all nine at once.
