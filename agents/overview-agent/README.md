# Overview agent

A nightly `CronWorkflow` (`cronworkflow.yaml`, 03:00 UTC) that clones this
repo and the product repo it's pointed at, reads Prometheus/Loki/Tempo/
Langfuse plus open Forgejo issues, and runs OpenHands headlessly to open PRs
against whichever repo(s) it finds something worth fixing in. It never
pushes to a default branch or merges anything itself.

## What you need to configure before this does anything

**1. A Forgejo access token** the agent uses to clone over HTTPS and to open
PRs via the API — a dedicated bot account's token, not your own:

```bash
kubectl create secret generic overview-agent-secrets \
  --namespace agents-system \
  --from-literal=FORGEJO_URL="https://forgejo.local" \
  --from-literal=FORGEJO_OWNER="<org-or-user>" \
  --from-literal=FORGEJO_PLATFORM_REPO="next-gen-cicd" \
  --from-literal=FORGEJO_TOKEN="<bot-account-token>" \
  --from-literal=PRODUCT_REPO_URL="https://forgejo.local/<org>/<product-repo>.git" \
  --from-literal=LLM_MODEL="anthropic/claude-sonnet-4-5" \
  --from-literal=LLM_API_KEY="<llm-api-key>" \
  --from-literal=LLM_BASE_URL=""
```

Everything on the right of `=` above is a placeholder — nothing in this
repo's committed YAML contains real credentials; this Secret is created by
hand (or synced from Vault/SOPS through the `secrets` component, if that's
enabled) and is never checked in.

Leave `PRODUCT_REPO_URL` empty if there's no separate product repo yet — the
clone step skips it.

**2. Langfuse keys**, optional, only if the `observability` component's
Langfuse is enabled and you want the agent reading LLM traces:

```bash
kubectl create secret generic overview-agent-langfuse \
  --namespace agents-system \
  --from-literal=LANGFUSE_PUBLIC_KEY="<public-key>" \
  --from-literal=LANGFUSE_SECRET_KEY="<secret-key>"
```

and add an `envFrom` entry for it in `cronworkflow.yaml`'s `openhands`
template.

**3. The bot account itself** needs write access (not admin) to both repos
in Forgejo, and PR-open permission — add it as a collaborator the same way
you would a human contributor.

## Single-node notes

One Pod, once a night, for a few minutes — negligible steady-state cost. The
`clone-repos` step uses a shallow clone (`--depth 50`) to keep it light. The
`openhands` step runs with `SANDBOX_RUNTIME=local` rather than its default
Docker-in-Docker sandbox, so it doesn't need privileged access on a
single-node cluster — this trades away OpenHands' ability to execute
untrusted generated code in an isolated container, which is an acceptable
trade for a scoped, prompted review/PR task but revisit it if the prompt
scope grows.
