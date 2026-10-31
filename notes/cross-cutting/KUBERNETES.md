## What Kubernetes is

> **In short**: Kubernetes (K8s) is a container orchestrator: it runs containers on a cluster of machines, restarts them when they fail, scales them, and routes the traffic to them. It's declarative: you describe the desired state in YAML ("3 instances of this image"), and controllers continuously compare it with the actual state and correct the gap (reconciliation loop).

- **Desired state**: `kubectl apply -f deployment.yaml` sends the description to the cluster; Kubernetes then makes it true, and keeps it true (a crashed container is restarted, a lost node's containers are recreated elsewhere).
- **What it brings**: self-healing, horizontal scaling, rolling updates without downtime, service discovery and load balancing, configuration and secrets injected at runtime.
- **What it doesn't do**: build images (Docker, Jib, Buildpacks), or replace the application's own resilience (retries, timeouts).

## Cluster architecture: control plane and nodes

> **In short**: The control plane decides: the API server is the single entry point, etcd stores the cluster state, the scheduler chooses a node for each pod, and the controller manager runs the reconciliation loops. The worker nodes execute: the kubelet starts the pod's containers through the container runtime, and kube-proxy implements the services' networking.

```text
            kubectl / CI
                 │
┌─ Control plane ┴───────────────────────────────────────┐
│ kube-apiserver ── etcd                                  │
│ kube-scheduler    kube-controller-manager               │
└────────────────────────┬────────────────────────────────┘
          ┌──────────────┴──────────────┐
┌─ Node 1 ┴──────────────┐   ┌─ Node 2 ─┴─────────────┐
│ kubelet  kube-proxy    │   │ kubelet  kube-proxy    │
│ containerd             │   │ containerd             │
│ [pod] [pod]            │   │ [pod]                  │
└────────────────────────┘   └────────────────────────┘
```

| Component                    | Role                                                                                                     |
| ---------------------------- | -------------------------------------------------------------------------------------------------------- |
| **kube-apiserver**           | the REST API of the cluster: every component and `kubectl` go through it, nothing talks to etcd directly |
| **etcd**                     | distributed key-value store holding the whole cluster state (the objects and their status)               |
| **kube-scheduler**           | assigns each new pod to a node, based on the requested resources, constraints and affinities             |
| **kube-controller-manager**  | runs the controllers (Deployment, ReplicaSet, Node, Job...), each reconciling one type of object         |
| **cloud-controller-manager** | the link with the cloud provider: load balancers, volumes, nodes (EKS, GKE, AKS)                         |
| **kubelet**                  | agent on each node: starts and watches the containers of the pods assigned to it, runs the probes        |
| **container runtime**        | runs the containers (containerd, CRI-O)                                                                  |
| **kube-proxy**               | network rules on each node, so that a Service's address reaches its pods                                 |

**A node** is one machine of the cluster: a cloud VM (an EC2 instance) or a physical server, running the kubelet, the container runtime and kube-proxy. A node hosts many pods; a pod always runs entirely on one node, chosen by the scheduler. In the cloud, nodes are grouped in node pools that can scale automatically.

Managed offerings (EKS, GKE, AKS) run the control plane for you; only the nodes and the workloads are yours.

## Workloads: Pod, Deployment, StatefulSet, Job

> **In short**: A pod is the smallest unit: one or more containers sharing an IP address and volumes, ephemeral and never created by hand. A Deployment manages identical, stateless pods (through a ReplicaSet), with rolling updates; a StatefulSet gives stable names and storage to stateful pods (databases, Kafka); a DaemonSet runs one pod per node; a Job runs to completion, a CronJob on a schedule.

| Object                      | For                                                             | Example                               |
| --------------------------- | --------------------------------------------------------------- | ------------------------------------- |
| **Pod**                     | one instance: containers sharing network and volumes            | the application container + a sidecar |
| **Deployment** → ReplicaSet | N interchangeable stateless pods, rolling updates and rollbacks | a Spring Boot API                     |
| **StatefulSet**             | stable pod names (`kafka-0`, `kafka-1`) and one volume per pod  | PostgreSQL, Kafka, Elasticsearch      |
| **DaemonSet**               | one pod on every node                                           | log collector, monitoring agent       |
| **Job** / **CronJob**       | a task that ends / a scheduled task                             | a migration, a nightly export         |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels: { app: order-service }
  template: # the pod template
    metadata:
      labels: { app: order-service } # the Service finds the pods by label
    spec:
      containers:
        - name: app
          image: registry.example.com/order-service:1.4.2
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 500m, memory: 768Mi } # reserved on the node, used by the scheduler
            limits: { memory: 768Mi } # above it, the container is killed (OOMKilled); no CPU limit (see below)
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
```

- **Pods are ephemeral**: their IP changes at each restart, so nothing should call a pod directly: a Service gives a stable address (see below).
- **Labels and selectors** link the objects: the Deployment and the Service find their pods by `app: order-service`.
- **Rolling update**: changing the image creates new pods progressively and removes the old ones once the new ones are ready; `kubectl rollout undo` goes back.

**StatefulSet vs DaemonSet**: a StatefulSet runs exactly `replicas` pods, wherever the scheduler puts them, each with a stable name and its own volume; a DaemonSet runs one pod per node, whatever the number of nodes, for node-level agents.

**One volume per pod**: a StatefulSet's `volumeClaimTemplates` creates one PersistentVolumeClaim per pod (`data-kafka-0`, `data-kafka-1`...), each bound to its own disk outside the container. `kafka-0` always gets `data-kafka-0` back: after a container restart, after the pod is recreated, even on another node (the cloud disk is detached and reattached, within the same zone). Scaling down or deleting the StatefulSet keeps the PVCs by default.

- The data isn't replicated by Kubernetes: a PostgreSQL StatefulSet with `replicas: 3` gives three independent databases. Software that clusters itself (Kafka, Elasticsearch) fits `replicas: 3`; PostgreSQL needs an operator for replication (see where to put a database below).
- For a single database pod, a StatefulSet with `replicas: 1` rather than a Deployment: a Deployment's rolling update starts the new pod before stopping the old one, so two PostgreSQL would want the same disk; a StatefulSet never runs two `postgres-0` at once.

## Networking, configuration and storage

> **In short**: A Service gives a stable name and IP address to a set of pods and load-balances between them (`ClusterIP` inside the cluster, `LoadBalancer` exposed by the cloud). An Ingress, or its successor the Gateway API, routes external HTTP traffic to services by host and path. ConfigMaps and Secrets inject configuration as environment variables or files; a PersistentVolumeClaim requests storage that outlives the pod. Namespaces separate teams or environments.

| Object                                           | Role                                                                                                                                                           |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Service**                                      | stable DNS name (`order-service.shop.svc.cluster.local`, or just `order-service` in the same namespace) and load balancing to the pods matching its selector   |
| — `ClusterIP` (default)                          | reachable only inside the cluster: service-to-service calls                                                                                                    |
| — `NodePort` / `LoadBalancer`                    | exposed on a port of each node / through a cloud load balancer                                                                                                 |
| **Ingress** / **Gateway API**                    | HTTP routing from outside: `api.shop.com/orders` → `order-service`, TLS termination; needs a controller (NGINX, Traefik, a cloud one)                          |
| **ConfigMap**                                    | non-sensitive configuration, as environment variables or mounted files (`application.yml`)                                                                     |
| **Secret**                                       | passwords, keys, certificates; only **base64-encoded**, not encrypted by default: restrict access, enable encryption at rest, or use an external store (Vault) |
| **PersistentVolume** / **PersistentVolumeClaim** | storage independent of the pod's life; the pod requests a PVC, the cluster provides a volume (cloud disk)                                                      |
| **Namespace**                                    | logical isolation: names, quotas and access rights per team or environment                                                                                     |
| **HorizontalPodAutoscaler**                      | adjusts the number of replicas from the CPU, memory or custom metrics                                                                                          |

**Volume access modes**: most cloud disks are `ReadWriteOnce` (RWO): attached in read-write to one node at a time, like a physical disk to one machine. Network file systems (NFS, EFS, Azure Files) are `ReadWriteMany` (RWX): mountable by several nodes. That's why a Deployment shouldn't share one RWO PVC between its replicas: the pods on other nodes stay stuck in `ContainerCreating` (`Multi-Attach error`), and those on the same node would write the same files concurrently. Deployments are for stateless pods; a StatefulSet gives each pod its own disk.

**Where to put a database**:

| Case | Usual choice |
|---|---|
| production, in the cloud | a managed database outside the cluster (RDS, Cloud SQL, Azure Database): backups, high availability and upgrades handled by the provider |
| production inside the cluster | an operator (CloudNativePG, Zalando, Crunchy): primary + replicas, each with its own volume, streaming replication, failover, backups |
| dev, test environments | a StatefulSet with `replicas: 1` and a RWO volume: no high availability |
| automated tests | Testcontainers, outside Kubernetes |

## Running a Spring Boot application on Kubernetes

> **In short**: The readiness probe tells when a pod can receive traffic, the liveness probe when it must be restarted: Spring Boot exposes both through Actuator, automatically on Kubernetes. Requests and limits must match the JVM's memory, and graceful shutdown lets in-flight requests finish during a rolling update. Configuration comes from ConfigMaps and Secrets, not from the image.

- **Readiness vs liveness**: a pod failing readiness is removed from the Service (no traffic) but kept alive; failing liveness, it's restarted. Liveness must not check external dependencies: a database outage would restart every pod in a loop, without fixing anything.
- **Spring Boot probes**: `/actuator/health/liveness` and `/actuator/health/readiness` are enabled automatically when the application detects Kubernetes. A slow start is covered by a `startupProbe`, so that liveness doesn't kill the pod while it starts.
- **Graceful shutdown**: Kubernetes sends `SIGTERM`, then `SIGKILL` after `terminationGracePeriodSeconds` (30 s by default). With `server.shutdown=graceful` (the default since Spring Boot 3.4), the server stops accepting requests and finishes the current ones. A short `preStop` sleep gives the Service time to stop sending traffic to the pod.
- **Configuration**: the same image in every environment, with `SPRING_PROFILES_ACTIVE`, environment variables and mounted files coming from ConfigMaps and Secrets.

**Resources and the JVM**: nothing to code, the JVM reads the container's limits (cgroups) since Java 10, but its defaults need tuning.

- **Units**: CPU in cores, `500m` = 500 millicores = half a core, as CPU time: with a `500m` limit, the container gets 50 ms of CPU per 100 ms period, across all its threads. Memory in bytes (`768Mi`).
- **Requests vs limits**: the request is reserved on the node (the scheduler places pods by their requests), the limit is the ceiling. Above the memory limit, the container is killed (`OOMKilled`, no Java stack trace); above the CPU limit, it's paused until the next period (throttling).
- **Memory**: by default the maximum heap is 25% of the limit. `-Xmx` equal to the limit ends in `OOMKilled`, since the JVM also uses metaspace, thread stacks, JIT code and buffers. Usual setting: `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`, and a memory request equal to the limit, so that the memory is really reserved (the scheduler doesn't overcommit it).
- **CPU**: a CPU limit slows down the startup (class loading, JIT, Spring), and the JVM sizes itself from it: with `500m` it sees 1 processor, creates few GC threads and picks the Serial GC instead of G1. Hence a CPU request without a CPU limit for JVM services, possibly with `-XX:ActiveProcessorCount=2`, since without a limit the JVM sees all the node's cores.
- Spring Boot images built with Buildpacks (`mvn spring-boot:build-image`) include a memory calculator that sets these options from the container's limit.

**QoS classes**, which decide which pods are evicted first when a node runs out of memory:

| Class | Condition | Under memory pressure |
|---|---|---|
| BestEffort | no requests or limits | evicted first |
| Burstable | requests lower than the limits, or some missing | evicted next, those exceeding their requests first |
| Guaranteed | request = limit for CPU and memory, in every container | evicted last |

A JVM service without a CPU limit is `Burstable`, but with a memory request equal to its limit it never exceeds its request, so it's among the last ones evicted.
