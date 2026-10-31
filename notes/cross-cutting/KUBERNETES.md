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
            requests: { cpu: 500m, memory: 512Mi } # used by the scheduler to place the pod
            limits: { memory: 768Mi } # above it, the container is killed (OOMKilled)
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
```

- **Pods are ephemeral**: their IP changes at each restart, so nothing should call a pod directly: a Service gives a stable address (see below).
- **Labels and selectors** link the objects: the Deployment and the Service find their pods by `app: order-service`.
- **Rolling update**: changing the image creates new pods progressively and removes the old ones once the new ones are ready; `kubectl rollout undo` goes back.

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

## Running a Spring Boot application on Kubernetes

> **In short**: The readiness probe tells when a pod can receive traffic, the liveness probe when it must be restarted: Spring Boot exposes both through Actuator, automatically on Kubernetes. Requests and limits must match the JVM's memory, and graceful shutdown lets in-flight requests finish during a rolling update. Configuration comes from ConfigMaps and Secrets, not from the image.

- **Readiness vs liveness**: a pod failing readiness is removed from the Service (no traffic) but kept alive; failing liveness, it's restarted. Liveness must not check external dependencies: a database outage would restart every pod in a loop, without fixing anything.
- **Spring Boot probes**: `/actuator/health/liveness` and `/actuator/health/readiness` are enabled automatically when the application detects Kubernetes. A slow start is covered by a `startupProbe`, so that liveness doesn't kill the pod while it starts.
- **Memory**: the JVM is container-aware and sizes its heap from the container's limit (`-XX:MaxRAMPercentage=75`); the heap isn't the only memory used (metaspace, threads, buffers), so a limit equal to `-Xmx` ends in `OOMKilled`.
- **CPU**: a CPU limit throttles the process; many teams set a CPU request without a CPU limit for JVM services.
- **Graceful shutdown**: Kubernetes sends `SIGTERM`, then `SIGKILL` after `terminationGracePeriodSeconds` (30 s by default). With `server.shutdown=graceful` (the default since Spring Boot 3.4), the server stops accepting requests and finishes the current ones. A short `preStop` sleep gives the Service time to stop sending traffic to the pod.
- **Configuration**: the same image in every environment, with `SPRING_PROFILES_ACTIVE`, environment variables and mounted files coming from ConfigMaps and Secrets.
