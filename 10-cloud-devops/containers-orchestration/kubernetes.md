# Kubernetes: Orchestrate and Operate Workloads

[Back to the learning package](README.md) · [Docker](docker.md) · [Helm](helm.md) · [Learning path](learning-path.md)

## The problem Kubernetes solves

Running a process in a container is straightforward. Operating a changing set of containers across multiple machines is not. You must decide where workloads run, restore desired replica counts after failures, provide stable names and network endpoints as Pods change, and make controlled updates.

Kubernetes is a declarative orchestration system for this class of problem. It is useful when the operational benefits justify the cost of a cluster and its supporting systems. A Kubernetes API object does not itself guarantee application availability: code, configuration, capacity, probes, network policy, storage, upgrades, and monitoring still need sound design.

For a single small service, a VM, managed application platform, or single-host Compose deployment may be simpler and cheaper. Learn the [Docker layer](docker.md) first so that Kubernetes failures are not confused with image-build failures.

## How the control loop works

1. A user or tool submits desired Kubernetes objects to the API server.
2. The API server authenticates and validates the request and stores cluster state in the control plane's backing store.
3. Controllers observe objects and compare the desired state with the state they can observe.
4. A scheduler assigns newly created, unscheduled Pods to eligible nodes using requests, constraints, affinity, and other scheduling information.
5. The node agent (kubelet) asks the node's CRI-compatible runtime to run the Pod's containers.
6. Controllers keep observing and act when the actual state differs from the declared state.

This is asynchronous reconciliation, not a single command that guarantees success. A Deployment can exist while its Pods are still pending, unable to pull an image, or failing readiness.

### Cluster components in brief

| Component | Role |
| --- | --- |
| API server | Main API endpoint and validation/authentication front end for cluster objects |
| `etcd` | Consistent key-value store for Kubernetes cluster data; back it up and secure it |
| Scheduler | Selects a node for a newly created Pod that has no node assignment |
| Controller manager | Runs controllers that observe and reconcile resources |
| Kubelet | Node agent that manages the Pods assigned to its node |
| Container runtime | Runs containers through the Container Runtime Interface (CRI), commonly containerd or CRI-O |
| Network plugin | Implements Pod networking and may provide additional network policy capabilities |
| `kubectl` | Client that sends requests to a cluster using a kubeconfig context |

Kubernetes does not require Docker Engine as the runtime on worker nodes. Docker-built images use OCI-compatible formats; Kubernetes nodes use a CRI-compatible runtime. The historical in-tree `dockershim` was removed, so Docker Engine and Kubernetes are not the same component.

## Workloads: choose the right controller

| Workload | Use it when | Key trade-off |
| --- | --- | --- |
| Pod | Learning mechanics or as a controller-managed unit | Usually do not create a long-lived application Pod directly; it has no higher-level replica controller |
| Deployment | Stateless service replicas, declarative updates, and rollbacks | Replicas are interchangeable; persistent identity is not provided |
| StatefulSet | Each replica needs stable identity, ordered behavior, or a stable storage claim | StatefulSet does not automatically provide database replication, backups, or high availability |
| Job | A finite task should run to completion | Retries and idempotency need to be designed for the task |
| CronJob | A scheduled Job | Account for overlapping runs, missed schedules, and idempotent work |
| DaemonSet | A Pod should run on selected nodes, such as a node-level agent | Not a general application-replication controller |

One container per Pod is the common model. Multiple containers belong in one Pod only when they are tightly coupled and should share the Pod's network and storage context (for example, a sidecar).

## Deploy and expose a stateless service

This minimal manifest illustrates the relationships among a Deployment, Pods, labels, and a Service. In a real cluster, replace the example image with an image accessible to every node; use a release-specific tag or digest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-api
  labels:
    app.kubernetes.io/name: sample-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: sample-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sample-api
    spec:
      containers:
        - name: api
          image: registry.example.com/team/sample-api:1.0.0
          ports:
            - name: http
              containerPort: 8000
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 5
            failureThreshold: 12
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: sample-api
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: sample-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

The labels in the Deployment selector, Pod template, and Service selector must agree. `containerPort` documents the port the process listens on; the Service is what gives the group of matching Pods a stable in-cluster endpoint. `ClusterIP` is internal to the cluster.

For a local `kind` cluster, a locally built image must either be pushed to a registry reachable by the cluster or loaded into it with `kind load docker-image`. The local-development path is demonstrated in the [capstone](learning-path.md).

## Networking and exposure

The Kubernetes network model has distinct paths: container-to-container within one Pod, Pod-to-Pod, Pod-to-Service, and external-to-Service traffic. A network plugin implements the Pod networking model; a Service abstracts a changing set of back-end Pods.

| Resource / type | Typical purpose |
| --- | --- |
| `ClusterIP` Service | Stable in-cluster endpoint and service discovery |
| `NodePort` Service | Expose a port on cluster nodes; often a building block rather than the preferred public entry point |
| `LoadBalancer` Service | Ask a supported platform/controller for an external load balancer |
| Ingress | HTTP/HTTPS routing resource, but it has no effect without an Ingress controller |
| Gateway API | Extensible, role-oriented routing APIs; use a supported Gateway controller |

The Kubernetes project states that the Ingress API is frozen and recommends considering Gateway API for new functionality. Ingress remains available, and existing controllers/configurations still matter; choose based on your platform, controller, feature needs, and operational support. Creating an Ingress object alone does not install a controller or expose traffic.

Use DNS names and Services rather than depending on Pod IPs. For a service in another namespace, a fully qualified name typically includes the service, namespace, and `svc.cluster.local` domain; actual cluster DNS configuration can vary.

## Health, resources, and configuration

### Probes answer different questions

| Probe | Question | Failure effect |
| --- | --- | --- |
| Startup | Has this container completed initialization? | The kubelet restarts it according to restart policy after the configured failures; startup probe gates liveness/readiness checks until it succeeds |
| Readiness | Should this Pod currently receive Service traffic? | It is removed from ready Service endpoints while it is not ready |
| Liveness | Is the process so stuck that restarting it is appropriate? | Repeated failure causes the kubelet to restart the container |

Do not use a dependency outage as a reflexive liveness failure. A poorly chosen liveness probe can restart many otherwise-recoverable Pods during load or an upstream outage and worsen the incident. Probes should represent the application's behavior and startup time.

### Requests, limits, and autoscaling

- **Requests** inform scheduling and resource accounting. CPU and memory requests are not the same as a hard cap on actual usage.
- **CPU limits** are enforced through throttling; excessive throttling can harm latency.
- **Memory limits** may result in an OOM kill when the kernel detects memory pressure.
- **Horizontal Pod Autoscaler (HPA)** changes a workload's desired replica count based on metrics. Resource-based scaling needs a metrics API; CPU-utilization targets are calculated relative to requests, so missing requests can prevent useful utilization readings.
- More replicas do not fix a saturated database, a serialized bottleneck, bad caching, or a non-scalable workload. Measure the bottleneck and understand the metric before scaling.

Start with observed usage and load tests; example numbers in a tutorial are placeholders, not production sizing recommendations.

### Configuration, secrets, and access

- Use **ConfigMaps** for non-confidential configuration that differs between environments.
- Use **Secrets** for sensitive values, but do not assume a Kubernetes Secret is encrypted just because it is called a Secret. Kubernetes documents that Secret data is base64-encoded and stored unencrypted in etcd by default unless encryption at rest is configured.
- Limit access with RBAC and namespace/service-account boundaries. Consider an external secret store when appropriate.
- Never commit real credentials into source, Helm values, a container image, or a public manifest. Use test-only placeholder values in labs.
- Give each workload only the permissions it needs; avoid broad cluster-admin access for application workloads.

### Persistent state

A Pod's writable filesystem is not a durable data store. A PersistentVolume is storage in the cluster; a PersistentVolumeClaim requests storage; a StorageClass can enable dynamic provisioning. Availability, performance, access mode, failure recovery, and backups depend on the storage provider and application architecture.

Use a StatefulSet only when stable identity/ordering or per-replica storage is actually needed. It does not by itself make a database safe or highly available. For production data, define and test backups, restores, upgrades, and failure recovery separately.

## Rollouts and operations

For a Deployment update, change the Pod template (for example, its image), watch the rollout, and verify application readiness:

```sh
kubectl apply -f k8s/
kubectl rollout status deployment/sample-api -n learning
kubectl get deployments,pods,services -n learning
kubectl rollout history deployment/sample-api -n learning
```

A Deployment can keep the requested number of Pods, but it cannot prove the new code is functionally correct. Readiness, rollout strategy, tests, and observability determine whether a rollout is safe. A Deployment rollback restores an earlier Pod template; it does not undo an incompatible database migration or recover lost data.

Before any mutation or cleanup, verify `kubectl config current-context` and the namespace. Prefer a dedicated local learning cluster and namespace.

## Troubleshooting by symptom

| Symptom | First checks |
| --- | --- |
| Pod is `Pending` | `kubectl describe pod ...`; inspect events, requests, node capacity/constraints, PVC binding, and quotas |
| `ImagePullBackOff` / `ErrImagePull` | Check image name/tag/digest, registry reachability, image-pull credentials, architecture, and recent Pod events |
| `CrashLoopBackOff` | `kubectl logs ... --previous`; inspect exit code and `kubectl describe pod ...`; check command, config, permissions, OOM status, and probe thresholds |
| Pod is `Running` but not `Ready` | Inspect readiness probe, startup path, logs, and dependencies; Running does not mean traffic-ready |
| Service has no ready endpoints | Compare Service selector with Pod labels; inspect readiness and EndpointSlices |
| Service cannot be reached | Check namespace/DNS, `port` vs `targetPort`, listening address, network policy, and the intended exposure controller |
| New image is not running | Verify the manifest changed, image reference is unique, rollout completed, pull policy/cache, and registry access |
| Pod repeatedly becomes `OOMKilled` | Check memory limit/request, actual usage, application memory behavior, and whether the node is under pressure |

Useful commands:

```sh
kubectl get pods,deployments,services -n learning
kubectl describe pod <pod-name> -n learning
kubectl logs <pod-name> -n learning
kubectl logs <pod-name> -n learning --previous
kubectl get events -n learning --sort-by=.metadata.creationTimestamp
kubectl get endpointslices -n learning
kubectl exec -it <pod-name> -n learning -- sh
kubectl port-forward service/sample-api 8080:80 -n learning
```

Some minimal images do not contain a shell, and production access policies may disallow `exec`; use logs, ephemeral debugging tools where permitted, and platform observability instead.

## Further reading

- [Cluster architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/) and [cluster networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) and [HPA](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/), [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), and [Secret good practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) and [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Troubleshooting applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Container runtimes and CRI](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
