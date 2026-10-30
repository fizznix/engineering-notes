# Learning Path and Capstone: From Source to a Kubernetes Release

[Back to the learning package](README.md) · [Docker](docker.md) · [Kubernetes](kubernetes.md) · [Helm](helm.md)

This is a hands-on path for learning what each layer contributes. Use one small HTTP service that listens on a known port and has startup, readiness, and liveness behavior. Do not start with a large microservice or GenAI stack: one API plus one optional database is enough to reveal the important concepts.

## Environment

Install a container runtime, Docker CLI/Compose, `kubectl`, a local cluster such as `kind` or Minikube, and Helm. Use official installation instructions and keep the `kubectl` client within the supported minor-version skew for the cluster. If using Docker Desktop for commercial work, check its current subscription terms and system requirements.

For an Apple Silicon Mac with limited memory, choose one local Kubernetes option and run a small workload; avoid starting multiple clusters and heavy services at once. `kind` runs Kubernetes nodes in containers and supports loading a local image; Minikube is another local learning environment. Both still need enough host resources for the node and workload.

Before applying or deleting anything, check:

```sh
kubectl config current-context
kubectl get nodes
```

Use a local learning cluster and a dedicated namespace. Do not use these examples against a shared or production context.

## Eight-week progression

| Week | Learn | Deliverable / evidence |
| --- | --- | --- |
| 1 | Processes, HTTP ports, images vs. containers, Dockerfile, build context, run/log/inspect | Build and run the sample service; explain the `-p host:container` mapping |
| 2 | Image layers, cache, non-root execution, tags/digests, Compose services, networks, volumes | Run API plus a local dependency; show name-based service discovery and demonstrate data persistence |
| 3 | Cluster architecture, CRI, API objects, Pods, Deployments, labels/selectors | Apply a Deployment and explain how a controller recreates a deleted Pod |
| 4 | Services, DNS, probes, requests/limits, ConfigMaps and Secret handling | Access the service through a Service; diagnose a deliberately failing readiness probe |
| 5 | Storage, StatefulSet use cases, Job/CronJob, HPA, security, logs/events | Explain which workload fits a scenario and which infrastructure dependencies remain external |
| 6 | Helm chart layout, values, rendering, linting, release history | Render the same chart with two values files and explain every meaningful difference |
| 7 | Rollouts, rollback, failure diagnosis, image updates, registry workflow | Deploy a new release, inspect rollout status, and recover from a failed image/config |
| 8 | Review, optional Gateway/CI/observability extension, capstone demo | Demonstrate a repeatable end-to-end deployment and clearly explain trade-offs |

Adjust the calendar. Do not advance just because a week ended; move on when you can explain and demonstrate the checkpoint.

## Capstone phases

### Phase 1: Run a process, then build an image

1. Run the application directly and confirm its port and endpoints.
2. Add a Dockerfile and `.dockerignore`.
3. Build with a release-like tag and run with a host port mapping.
4. Inspect logs, exit status, environment, filesystem, and resource usage.
5. Rebuild after a source change and explain which build steps can be cached.

**Checkpoint:** You can run the exact same image twice and explain which parts are inside the image and which are runtime configuration.

Use the [Docker guide](docker.md) and Docker's official [build best practices](https://docs.docker.com/build/building/best-practices/).

### Phase 2: Add local dependencies with Compose

1. Add one database or cache service only if the application needs it.
2. Connect by the Compose service name, not a fixed container IP.
3. Use a named volume for development data and a health check for readiness where appropriate.
4. Inspect `docker compose config`, status, and logs.
5. Stop and recreate the database container; confirm the named-volume data remains.
6. If resetting the lab, understand that `docker compose down --volumes` deletes the named volume and its data.

**Checkpoint:** You can describe why `depends_on` or start order alone may not mean that a dependency is ready, and explain bind mount vs. named volume.

### Phase 3: Deploy the same image to Kubernetes

1. Start a local cluster and confirm its context and nodes.
2. Build the image locally and make it available to the cluster (for `kind`, use its image-loading command or use a local registry).
3. Create a namespace and apply a Deployment with at least two replicas.
4. Add a ClusterIP Service with matching labels/selectors.
5. Test using `kubectl port-forward` and verify the application response.
6. Delete one Pod and observe its controller create a replacement.
7. Change the image tag, apply the Deployment, and watch rollout status.

**Checkpoint:** You can draw the controller -> ReplicaSet -> Pod relationship, explain why a Pod IP is not a stable client address, and identify the CRI runtime as distinct from Docker Engine.

### Phase 4: Add operational behavior

1. Configure startup, readiness, and liveness probes based on the app's behavior.
2. Set CPU/memory requests and limits using measured needs rather than copying tutorial values into production.
3. Add non-secret configuration through a ConfigMap.
4. Use only a dummy test Secret; do not commit a real credential. Explain RBAC and encryption-at-rest requirements.
5. Inspect events and logs for a broken image, wrong container port, or intentionally failing readiness probe.
6. For a deliberately failed Deployment update, inspect the cause before restoring the known-good image and applying it again.

**Checkpoint:** You can explain how a bad liveness probe can worsen an incident and why readiness, resources, and a healthy process are separate concerns.

### Phase 5: Package it with Helm

1. Start from the working Kubernetes manifests.
2. Create a chart with `Chart.yaml`, defaults in `values.yaml`, and templates for the Deployment and Service.
3. Parameterize only meaningful differences: image, replica count, resources, and service settings.
4. Render with `helm template`; inspect the YAML and compare the development and staging configurations.
5. Run `helm lint` and, where appropriate, server-side dry-run validation against the local API server.
6. Install with `helm upgrade --install`, check `helm status` and Kubernetes rollout status.
7. Change the image and upgrade. Inspect `helm history`, then roll back to an earlier revision.
8. Confirm what the rollback did **not** restore (database contents, schema migration, external side effects).
9. Uninstall only the local learning release and verify the namespace/resources afterward.

**Checkpoint:** You can explain chart version vs. app version, value precedence, what Helm stores as release history, and why secrets must not be put in values files.

## Failure-driven exercises

| Induced symptom in the local lab | Investigate | What it teaches |
| --- | --- | --- |
| Container exits immediately | Docker logs, command, exit code, required files | Image startup is separate from Kubernetes |
| Compose API cannot connect to DB | Service DNS name, container port, DB readiness | Container network identity and startup/readiness |
| Kubernetes Pod is Pending | Pod events, requests, PVC, node constraints | Scheduler decisions and unsatisfied resource/storage needs |
| Kubernetes Pod has `ImagePullBackOff` | Image reference, registry, credentials, architecture | Cluster nodes must be able to obtain the image |
| Pod is Running but Service does not send traffic | Readiness, Service selector, Pod labels, EndpointSlices | Running is not equivalent to ready |
| New version never becomes available | Probe path/port, logs, rollout status/events | Controlled rollout and failure diagnosis |
| Data disappears when a container is recreated | Writable layer vs. mounted volume | Persistence must be explicit |
| Helm install renders invalid resources | `helm lint`, `helm template`, API dry-run | Templates must produce valid, inspectable manifests |
| `helm rollback` restores old app but data is incompatible | Migration strategy and backup/restore | Resource history cannot reverse arbitrary state changes |

Use the official [Kubernetes troubleshooting guide](https://kubernetes.io/docs/tasks/debug/debug-application/) rather than guessing from a Pod status alone.

## Optional extensions

Complete these only after the basic path works:

- **Autoscaling:** Add HPA, provide metrics, set requests, generate safe local load, and inspect scale-up/scale-down behavior. HPA requires a metrics source; it is not automatic just because the resource exists.
- **External routing:** Compare a Service of type `LoadBalancer`, an existing Ingress controller, and Gateway API. A routing resource needs a compatible controller and infrastructure.
- **Persistent workload:** Add a stateful application only if needed. Configure and test volume lifecycle, backup, restore, and upgrade behavior.
- **Delivery pipeline:** Build, test, scan, tag, and push an image; deploy by immutable tag or digest. Do not expose credentials in logs or source.
- **Observability:** Add structured logs, metrics, and traces. Use readiness, saturation, errors, latency, and rollout health to inform operations.
- **Inference deployment:** If relevant, connect the container image and cluster deployment to the existing [inference engineering](../../08-inference-engineering/README.md) domain; serving models introduces additional GPU, memory, scheduling, and autoscaling constraints.
- **Cluster internals:** Read [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) after learning the API/controller model. It is a version-specific, non-production exercise and is not required for deploying applications.

## Safe local cleanup

Only run cleanup after confirming the context is the local learning cluster and `learning` is the exercise namespace:

```sh
helm uninstall sample-api --namespace learning
kubectl delete namespace learning
kind delete cluster --name learning
```

The commands are examples for named lab resources; use the actual release, namespace, and cluster names you created. Do not run broad system-prune commands as routine cleanup, and do not assume uninstalling a chart or deleting a namespace is a backup strategy.

## Completion criteria

You are ready to move beyond the tutorial when you can:

- Explain what Docker, Compose, Kubernetes, and Helm each own—and what they do not own.
- Build and identify a reproducible image, configure it at runtime, and avoid baking secrets into it.
- Explain the Kubernetes reconcile loop, Deployment/Pod relationship, Service selectors, and probe roles.
- Diagnose `Pending`, image-pull errors, crash loops, unready Pods, and missing Service endpoints using events, logs, and resource descriptions.
- Render, lint, install, upgrade, inspect, and roll back a Helm release without confusing chart/app versions.
- Describe the data, migration, security, and observability work that still remains outside a successful `kubectl apply` or `helm upgrade`.
