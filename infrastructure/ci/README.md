# CI: Tekton + Trivy/Syft/Grype + cosign

Tekton has no official Helm chart, so this component vendors pinned upstream
release manifests instead of templating one:

- `manifests/pipeline-v1.4.0.yaml` — Tekton Pipelines
- `manifests/triggers-v0.32.0.yaml` + `triggers-interceptors-v0.32.0.yaml` — Tekton Triggers
- `manifests/chains-v0.24.0.yaml` — Tekton Chains (signs build outputs)
- `tasks/` — this repo's Trivy/Syft/Grype Tasks
- `pipelines/build-scan-sign.yaml` — reference Pipeline wiring clone → build & push → scan
- `triggers/forgejo-trigger.yaml` — EventListener that starts a PipelineRun on a Forgejo push webhook

To bump a version, re-download the release YAML for the new tag and replace
the file (same URL pattern: `https://storage.googleapis.com/tekton-releases/<component>/previous/<version>/release.yaml`).

## What you need to configure before this is usable

**1. Harbor push credentials** — the Pipeline's `dockerconfig` workspace reads
a `.dockerconfigjson` Secret:

```bash
kubectl create secret docker-registry harbor-push-credentials \
  --namespace tekton-pipelines \
  --docker-server=harbor.local \
  --docker-username=<robot-account-name> \
  --docker-password=<robot-account-token>
```

Create a Harbor robot account scoped to the project(s) your pipelines push
into, rather than using the Harbor admin credentials here.

**2. Tekton Chains signing key** — Chains needs a cosign key pair to sign
images after `build-scan-sign` pushes them:

```bash
cosign generate-key-pair k8s://tekton-chains/signing-secrets
```

This prompts for a passphrase and stores the key pair as a Secret Chains
already expects. Then point Chains at Harbor as its signed-artifact store and
turn on attestation storage by editing the `chains-config` ConfigMap the
vendored release creates:

```bash
kubectl patch configmap chains-config -n tekton-chains --type merge -p '{
  "data": {
    "artifacts.oci.storage": "oci",
    "artifacts.taskrun.format": "in-toto",
    "transparency.enabled": "true"
  }
}'
```

Export the public half of that key (`cosign public-key -k k8s://tekton-chains/signing-secrets > cosign.pub`)
and load it as the `cosign-public-key` Secret the policy engine checks — see
`infrastructure/policy/kyverno/policies/require-signed-images.yaml` and the
root README.

**3. A Forgejo webhook per repo** pointed at
`http://el-forgejo-listener.tekton-pipelines.svc.cluster.local:8080`
(content type `application/json`) — or use Forgejo Actions directly with a
`.forgejo/workflows/` file that calls `tkn pipeline start build-scan-sign`
instead of going through the EventListener. Either path ends up at the same
Pipeline.

## Single-node notes

Tekton's own controllers (pipelines-controller, pipelines-webhook,
triggers-controller, chains-controller) are single-replica by default and
need no changes for one node. The `build-scan-sign` Pipeline's `source` and
`sbom` workspaces request ephemeral PVCs per run — on a busy repo, size and
clean these up (`tkn pipelinerun delete --keep 5`) so completed-run PVCs
don't pile up against the node's single StorageClass.
