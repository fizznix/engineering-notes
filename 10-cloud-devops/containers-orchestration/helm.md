# Helm: Package and Manage Kubernetes Releases

[Back to the learning package](README.md) · [Kubernetes](kubernetes.md) · [Learning path](learning-path.md)

## The problem Helm solves

A Kubernetes application may require a Deployment, Service, configuration, service account, autoscaler, and routing resources. Repeating similar YAML by hand across applications or environments can create drift and error-prone copies.

Helm packages related Kubernetes resources as a **chart**. Templates turn chart defaults and user-supplied values into Kubernetes manifests; an installed instance of a chart is a **release** with a name, namespace, and revision history. Helm can install, upgrade, inspect, and roll back those releases.

Helm is useful when you want versioned, reusable deployment configuration and meaningful variation between releases. It is not required for every Kubernetes application. A few stable manifests can be easier to review and operate directly.

Helm does not build/push the application image, run containers, or continuously reconcile the workload at runtime. Kubernetes controllers do that. Helm participates when someone or an automation system installs, upgrades, or rolls back a release.

## Chart, values, and release

| Term | Meaning | Do not confuse it with |
| --- | --- | --- |
| Chart | Versioned package of metadata, defaults, templates, and optional dependencies | A running application |
| `Chart.yaml` `version` | Version of the chart package | Version of the application binary |
| `Chart.yaml` `appVersion` | Informational application version for the chart | The chart package version or guaranteed image tag |
| `values.yaml` | Default configurable values for templates | A secure secret store |
| Template | Kubernetes manifest with Helm expressions | A Kubernetes object before rendering |
| Release | A named installation/upgrade history of a chart in a namespace | The image or a Kubernetes controller |
| Chart dependency | Another chart required by a parent chart | A replacement for service discovery or runtime dependencies |

A basic chart has a shape like:

```text
sample-api/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
```

Example metadata:

```yaml
apiVersion: v2
name: sample-api
description: Example HTTP service
type: application
version: 0.1.0
appVersion: "1.0.0"
```

The `apiVersion: v2` here is the chart format, not a Kubernetes API version. The Deployment inside the chart still uses its own `apiVersion: apps/v1`.

## How values become a release

1. Start with a working Kubernetes manifest. Understand the resources and their selectors before templating them.
2. Move only intentional configuration differences into `values.yaml` (image, replica count, resources, Service type, and similar choices).
3. Render the template using Helm and inspect the resulting Kubernetes YAML.
4. Lint the chart, then install it into a dedicated namespace.
5. Upgrade with a new image and verify the resulting Kubernetes rollout.
6. Inspect release history and practice rollback; separately handle data migrations and external side effects.

Values usually follow this precedence from least to most specific: chart defaults, values files (`-f`; later files override earlier files), then command-line `--set` values. Prefer a small number of documented values files over long, opaque `--set` strings for repeatable environments.

Example values:

```yaml
replicaCount: 2

image:
  repository: registry.example.com/team/sample-api
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8000
```

A template can refer to these values, for example:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Chart examples should include sensible defaults, clear value documentation, and validation for important values (for example, a schema). Keep templates readable; too many conditionals or arbitrary knobs can make a chart harder to use than plain manifests.

## A safe local workflow

The commands below assume a trusted local cluster, a chart named `sample-api`, and a namespace used only for this exercise. Pin a compatible Helm client and Kubernetes version for real work.

```sh
helm create sample-api
helm lint ./sample-api
helm template sample-api ./sample-api \
  --namespace learning \
  -f values-dev.yaml
```

Inspect the rendered resources before installing. Where permissions and API support allow, you can ask the target API server to validate rendered resources without persisting them:

```sh
helm template sample-api ./sample-api \
  --namespace learning \
  -f values-dev.yaml \
  | kubectl apply --dry-run=server -f -
```

Install or upgrade a release:

```sh
helm upgrade --install sample-api ./sample-api \
  --namespace learning \
  --create-namespace \
  -f values-dev.yaml \
  --wait \
  --timeout 2m

helm status sample-api --namespace learning
helm history sample-api --namespace learning
```

After confirming a previous revision exists, a rollback can restore its rendered Kubernetes resources:

```sh
helm rollback sample-api 1 --namespace learning
```

The revision number is an example; inspect `helm history` and choose deliberately. A Helm rollback does not revert a database schema, restore PVC contents, undo an external API call, or guarantee that an old binary can read newly written data. Plan backward-compatible application/data changes and backups independently.

For the full local install/upgrade/uninstall flow, see the [capstone lab](learning-path.md).

## Environment variation without duplication

Use one chart when multiple deployments share a resource structure but need different configuration. For example:

```text
values-dev.yaml       # small resource requests; non-production image tag
values-staging.yaml   # staging image tag and configuration
values-prod.yaml      # production-sized resources and release image
```

Keep the differences explicit, code-reviewed, and small. Do not use Helm templating just to avoid writing a few clear YAML manifests. Avoid putting real passwords, API keys, or certificates in chart defaults, committed values files, `--set`, or shell history. Kubernetes Secrets are base64-encoded and stored unencrypted by default unless the cluster is configured otherwise; use a secret manager and appropriately restricted access.

## Helm 4 and chart versioning

At research time (2026-10-30), the official Helm documentation describes Helm 4. It states that existing chart v2 format continues to work, while chart v3 is experimental. The examples here use `apiVersion: v2` and avoid experimental chart features.

Helm 4 includes behavior and plugin changes; for example, it defaults to server-side apply for new releases, while releases created with Helm 3 retain their prior apply method when upgraded unless overridden. Check the current [Helm 4 overview](https://helm.sh/docs/overview/), [version-skew policy](https://helm.sh/docs/topics/version_skew/), and migration documentation before upgrading a production release or plugin. The official chart, values, and OCI reference pages themselves warn that some content has not yet been fully updated for Helm 4; treat version-specific instructions accordingly.

Chart package versioning should be deliberate:

- Increment `version` when the chart's packaged contents change.
- Set `appVersion` to describe the application version; quote it if it is a string.
- Use a unique image tag or digest and explicitly configure it; `appVersion` does not automatically set the image tag.
- Keep release history and configuration consistent with the chosen Helm/Kubernetes compatibility policy.

## When Helm is not the right next step

- A few static manifests are already understandable and easy to maintain.
- Your team has standardized on Kustomize, an operator, or a GitOps workflow and does not need another packaging layer.
- The main problem is building/publishing an image, managing secrets, stateful data, or CI/CD; Helm alone does not solve those.
- The chart would expose many knobs without real use cases or tests.

Do not add a chart until the Kubernetes resources it will produce are clear. A confusing template can hide selector mismatches, insecure defaults, and invalid rollout behavior.

## Further reading

- [Helm Quickstart](https://helm.sh/docs/intro/quickstart/)
- [Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Charts and chart structure](https://helm.sh/docs/topics/charts/)
- [Values files](https://helm.sh/docs/chart_template_guide/values_files/) and [values best practices](https://helm.sh/docs/chart_best_practices/values/)
- [Helm 4 overview](https://helm.sh/docs/overview/) and [version-skew policy](https://helm.sh/docs/topics/version_skew/)
- [OCI-based registries](https://helm.sh/docs/topics/registries/) and [Artifact Hub](https://artifacthub.io/)
