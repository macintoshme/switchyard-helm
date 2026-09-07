# NeMo Switchyard Helm chart

Helm chart that deploys **NVIDIA NeMo Switchyard** and its web configurator on a Kubernetes cluster. The chart installs the Switchyard proxy, the FastAPI configurator UI, and optional Prometheus `ServiceMonitor` and Grafana dashboard resources into a dedicated namespace. Requires **Helm 4+** (the chart relies on server-side apply, e.g. `--create-namespace` coexisting with the chart's own `Namespace` resource).

This repo contains only the chart. The images it deploys are built and published by other pipelines:

| Service      | Image | Port | Purpose |
|--------------|-------|------|---------|
| switchyard   | `ghcr.io/macintoshme/nemo-switchyard` (built by [macintoshme/Switchyard](https://github.com/macintoshme/Switchyard)) | 4000 | LLM proxy + `/metrics` |
| configurator | `ghcr.io/macintoshme/nemo-switchyard-configurator` (built by [macintoshme/switchyard-configurator](https://github.com/macintoshme/switchyard-configurator)) | 8080 | Web UI to edit `routes.toml` |

## Released artifacts

Every `v*` tag push packages the chart and pushes it to ghcr.io (see `.github/workflows/release.yml`):

```bash
helm upgrade --install switchyard oci://ghcr.io/macintoshme/charts/switchyard --version 0.2.8
```

The release workflow refuses a tag that does not match `version`, `appVersion` in `Chart.yaml`, and the default `configurator.image.tag` in `values.yaml`, because the chart defaults must point at a configurator image that already exists in ghcr.io. Cut the configurator release (tag `v*` in the switchyard-configurator repo) before tagging this one.

## Quick start

1. **Create provider secrets**. Each LLM client in your `routes.toml` that uses `api_key_env` needs a Kubernetes `Secret` holding that key. Example for the `openai` client used by `examples/routes.toml.example`:
   ```bash
   kubectl create secret generic openai-secret \
     --from-literal=token="YOUR_OPENAI_API_KEY"
   ```
   Then reference it in a values file (the `envVar` must match the `api_key_env` in your routes file):
   ```yaml
   providers:
     - envVar: OPENAI_API_KEY
       secret: openai-secret
   ```

2. **Install the chart**. Override any defaults via `--set` or a custom values file. All resources deploy into a dedicated `switchyard` namespace which the chart creates for you. The chart ships a minimal placeholder `routes.toml` (one passthrough route to a local endpoint); point it at your real config to get started:
   ```bash
   helm upgrade --install switchyard . \
     -n switchyard --create-namespace \
     --set-file routesToml=examples/routes.toml.example \
     -f my-values.yaml   # optional custom values
   ```
   You can also edit the configuration later in the configurator UI. To deploy into a different (or pre-existing) namespace, set `namespace.name`, or `namespace.name: ""` with `create: false` to use the plain `-n` release namespace.

3. **Access the configurator**:
   ```bash
   kubectl port-forward -n switchyard svc/switchyard-configurator 8080:8080
   ```
   The UI lets you add providers, create routes, and save the configuration. Saving PATCHes the config `ConfigMap` via the configurator's service account; if the Stakater Reloader is installed, its annotation on the switchyard `Deployment` rolls the pods so new routes are picked up, and every `helm upgrade` rolls them via a config checksum regardless. Provider tokens entered in the UI are persisted into a UI-managed Secret (`<release>-tokens`) that both deployments load via `envFrom`; the `ConfigMap` never contains token values. Disable this with `configurator.tokenSecret.enabled=false` to rely solely on the pre-provisioned `providers:` secretKeyRefs from step 1.

4. **Verify**:
   ```bash
   kubectl port-forward -n switchyard svc/switchyard 4000:4000
   curl http://localhost:4000/health
   curl http://localhost:4000/v1/models
   ```

## Values

Key values are documented inline in [`values.yaml`](values.yaml) and validated against [`values.schema.json`](values.schema.json). The ones you are most likely to set:

| Value | Default | Purpose |
|-------|---------|---------|
| `routesToml` | placeholder passthrough route | Full `routes.toml` content; seed for the runtime config `ConfigMap`. A changed value overrides UI-saved edits on the next configurator start. |
| `modelHints` | `""` | Optional override for the configurator's model hints (Models tab suggestions), mounted over the baked-in file. |
| `providers` | `[]` | Pre-provisioned token `Secret` references (`envVar`, `secret`, `secretKey`), injected as `secretKeyRef` env vars. |
| `namespace.create` / `namespace.name` | `true` / `switchyard` | Dedicated namespace for all chart resources; `helm uninstall` removes it. |
| `switchyard.image` | `ghcr.io/macintoshme/nemo-switchyard:main` | Server image; pin with `sha-<commit>` or an upstream `vX.Y.Z` tag. |
| `switchyard.nodeSelector` | `{}` | Constrain switchyard pods to specific nodes (e.g. when only some nodes' CPUs support the server binary's instruction set). |
| `configurator.image` | `ghcr.io/macintoshme/nemo-switchyard-configurator:v0.2.8` | Configurator image; follows the chart `appVersion` by default. |
| `configurator.tokenSecret.enabled` | `true` | UI-managed `<release>-tokens` Secret for provider tokens. |
| `serviceType` | `ClusterIP` | Service type for both services. |
| `serviceMonitor.enabled` | `false` | Prometheus Operator `ServiceMonitor` scraping `/metrics`. |
| `grafanaDashboard.enabled` | `false` | Grafana dashboard ConfigMap with sidecar discovery labels. |
| `reloader.enabled` | `true` | Stakater Reloader annotation on the switchyard `Deployment`. |
| `imagePullSecrets` | `[]` | Pull secrets for both deployments, e.g. `[{name: regcred}]` for private registries. |

To point the chart at your own images (for clusters that cannot reach ghcr.io, or locally built ones loaded into kind/minikube/Rancher Desktop):

```yaml
switchyard:
  image:
    registry: ghcr.io/your-org   # or "" for a locally loaded image
    repository: nemo-switchyard
    tag: v0.2.8                  # empty = chart appVersion
configurator:
  image:
    registry: ghcr.io/your-org
    repository: nemo-switchyard-configurator
```

## Routes

`examples/routes.toml.example` shows the configuration format: LLM clients, targets, and routes. Eight route types are supported:

- **Passthrough** – direct to a single target
- **Random** – weighted A/B split across targets
- **LLM classifier (capability)** – route based on model difficulty
- **LLM classifier (escalation)** – similar to capability but with confirmations
- **LLM classifier (custom)** – custom classifier with arbitrary targets, default target, and a policy selector
- **Stage router** – selects a tier based on tool/agent signals
- **Advisor gate** – executor runs, advisor reviews, then gates further execution
- **Composite** – a classifier determines the tier, then a stage router selects the concrete model

## Metrics

Switchyard exposes Prometheus metrics (request/error counters, latency histograms, token counters, routing-overhead and run-duration histograms). With `serviceMonitor.enabled=true` a Prometheus Operator installation scrapes them, and with `grafanaDashboard.enabled=true` the shipped Grafana dashboard shows request rate and traffic share by model, router and direct-upstream error rates, error-code distribution, token throughput (prompt/completion/reasoning), total/model-call/routing-overhead latency percentiles, and routing effectiveness (decisions, retries recovered, run duration). Prometheus and Grafana themselves are assumed to come from the cluster (e.g. kube-prometheus-stack).

## Notes

- Switchyard is pre-alpha upstream; the default `main` server tag rolls with every push to the fork. Pin it with `sha-<commit>` or an upstream `vX.Y.Z` release tag for reproducible installs.
- This deployment is intended for development/testing. For production you should add TLS, stricter RBAC, and external secret management.