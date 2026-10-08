# Bootstrap

Argo CD can't install itself via GitOps, so this is the one place in the repo
with an imperative install step. Everything after this is declarative and
lives under `infrastructure/` and `apps/`.

Two ways to do it — pick one:

## Option A: one command (this chart)

`bootstrap/` is itself a Helm chart: it installs Argo CD (as a dependency,
sized for a single node) and a post-install job applies the app-of-apps
`Application` for you, with whichever component groups you choose.

```bash
cd bootstrap
helm dependency update
helm install next-gen-cicd . --wait --timeout 10m
```

`--wait` matters here: it's what guarantees Argo CD's `Application` CRD is
registered before the post-install job tries to use it (the job also
retries for several minutes on its own, in case you drop `--wait`).

Toggle a component group off before installing:

```bash
helm install next-gen-cicd . --wait --timeout 10m --set components.observability=false
```

or after:

```bash
helm upgrade next-gen-cicd . --reuse-values --set components.observability=false
```

Already running Argo CD elsewhere and just want this repo's app-of-apps
applied against it? Set `argo-cd.enabled=false`:

```bash
helm install next-gen-cicd . --set argo-cd.enabled=false
```

`helm install` prints next steps (admin password, port-forward command) via
`NOTES.txt` when it finishes. From here on, everything is managed by Argo CD
syncing this Git repository — see the root `README.md` for what each
component needs configured before it's actually usable.

## Option B: step by step

Useful if you want to see (or control) each step individually, or don't want
a Helm release object representing Argo CD itself.

### 1. Install Argo CD

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm install argocd argo-cd/argo-cd \
  --version 10.10.1 \
  --namespace argocd \
  -f bootstrap/values.yaml --reuse-values=false -f <(yq '.["argo-cd"]' bootstrap/values.yaml)
```

(That last line extracts just the `argo-cd:` block from `bootstrap/values.yaml`
so you're using the exact same single-node sizing as option A. If you don't
have `yq`, copy that block into its own file by hand instead.)

Wait for it to come up:

```bash
kubectl -n argocd rollout status deploy/argocd-server
```

### 2. Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Log in (`admin` / the password above) once you can reach the UI — see the
root README's "Reaching each UI" section for port-forward and ingress
options. Change this password after first login.

### 3. Point Argo CD at this repository and apply the app-of-apps

```bash
kubectl apply -n argocd -f apps/root-application.yaml
```

That single `Application` renders `apps/` (the local Helm chart that is the
app-of-apps) and fans out into one child `Application` per enabled component
group under `infrastructure/`. Edit `apps/values.yaml` directly (and commit
it) to toggle component groups with this path, rather than `--set`.
