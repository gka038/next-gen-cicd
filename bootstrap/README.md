# Bootstrap

Argo CD can't install itself via GitOps, so this is the one place in the repo
with an imperative install step. Everything after this is declarative and
lives under `infrastructure/` and `apps/`.

## 1. Install Argo CD

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm install argocd argo-cd/argo-cd \
  --version 10.10.1 \
  --namespace argocd \
  -f bootstrap/argocd-values.yaml
```

Wait for it to come up:

```bash
kubectl -n argocd rollout status deploy/argocd-server
```

## 2. Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Log in (`admin` / the password above) once you can reach the UI — see the
root README's "Reaching each UI" section for port-forward and ingress
options. Change this password after first login.

## 3. Point Argo CD at this repository and apply the app-of-apps

```bash
kubectl apply -n argocd -f apps/root-application.yaml
```

That single `Application` renders `apps/` (the local Helm chart that is the
app-of-apps) and fans out into one child `Application` per enabled component
group under `infrastructure/`. From here on, everything is managed by Argo CD
syncing this Git repository — see the root `README.md` for how to toggle
components and what each one needs configured before it's actually usable.
