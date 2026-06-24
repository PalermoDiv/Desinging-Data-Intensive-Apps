# Kubernetes: A Practical Overview

Kubernetes (often called **K8s**) is an open-source platform for **automating the deployment, scaling, and management of containerized applications**.

If Docker packages your application into a container, Kubernetes decides where that container runs, keeps it healthy, scales it up or down, and exposes it to users.

---

## 1. The Problem Kubernetes Solves

Running one or two containers manually is easy. Running hundreds or thousands across many machines is not.

Without Kubernetes, you face questions like:

- Which machine should this container run on?
- What happens if a machine fails?
- How do I scale from 3 to 30 replicas?
- How do containers find and talk to each other?
- How do I roll out a new version without downtime?
- How do I manage configuration and secrets?

Kubernetes solves these by providing a **declarative control plane**: you describe the desired state, and Kubernetes makes it happen.

---

## 2. What Is a Cluster?

A **Kubernetes cluster** is a set of machines that run containerized applications.

It has two main parts:

| Part | Role |
|---|---|
| **Control Plane** | Makes global decisions, detects failures, schedules work. The brain. |
| **Worker Nodes** | Run the actual application containers. The muscle. |

---

## 3. Control Plane Components

These run on the master nodes and manage the cluster.

| Component | What It Does |
|---|---|
| **API Server** | The front door. All commands and queries go through it. |
| **etcd** | A distributed key-value store that holds the cluster state. |
| **Scheduler** | Decides which worker node should run each new pod. |
| **Controller Manager** | Runs controllers that keep the system in the desired state. |
| **Cloud Controller Manager** | Integrates with cloud providers for load balancers, storage, and nodes. |

---

## 4. Worker Node Components

These run on every machine that executes workloads.

| Component | What It Does |
|---|---|
| **Kubelet** | An agent that talks to the API server and runs containers on the node. |
| **Container Runtime** | The software that runs containers, such as containerd or CRI-O. |
| **Kube-proxy** | Manages networking rules so services can reach pods. |

```
┌──────────────────────────────────────────────────────────────┐
│                     Control Plane                             │
│  ┌─────────────┐ ┌──────────┐ ┌─────────────┐ ┌────────────┐ │
│  │ API Server  │ │ Scheduler│ │ Controllers │ │    etcd    │ │
│  └──────┬──────┘ └──────────┘ └─────────────┘ └────────────┘ │
└───────┬──────────────────────────────────────────────────────┘
        │
        │ manages
        ▼
┌──────────────────────────────────────────────────────────────┐
│                      Worker Nodes                             │
│  ┌─────────────────────┐    ┌─────────────────────┐           │
│  │      Node 1         │    │      Node 2         │           │
│  │  ┌───────────────┐  │    │  ┌───────────────┐  │           │
│  │  │   Kubelet     │  │    │  │   Kubelet     │  │           │
│  │  │  ┌─────────┐  │  │    │  │  ┌─────────┐  │  │           │
│  │  │  │  Pod A  │  │  │    │  │  │  Pod B  │  │  │           │
│  │  │  └─────────┘  │  │    │  │  └─────────┘  │  │           │
│  │  └───────────────┘  │    │  └───────────────┘  │           │
│  └─────────────────────┘    └─────────────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Core Kubernetes Objects

### Pod

A **Pod** is the smallest deployable unit in Kubernetes. It usually wraps one container, but can hold multiple tightly coupled containers that share storage and network.

- Pods are ephemeral: they can be created, destroyed, and recreated.
- You usually do not create pods directly. Controllers like Deployments create them for you.

### Deployment

A **Deployment** manages a set of identical pods and ensures the right number are running.

- It supports rolling updates.
- It supports rollback to a previous version.
- It uses a ReplicaSet under the hood.

Use it for stateless applications like APIs, web servers, and background workers.

### ReplicaSet

A **ReplicaSet** ensures that a specified number of pod replicas are running at all times.

Most users work with Deployments instead of ReplicaSets directly, because Deployments add update and rollback behavior.

### Service

A **Service** provides a stable network endpoint for a group of pods.

Because pods come and go, their IP addresses change. A Service gives you a stable IP and DNS name.

| Service Type | Use Case |
|---|---|
| **ClusterIP** | Internal communication only. Default. |
| **NodePort** | Exposes the service on each node's IP at a static port. |
| **LoadBalancer** | Uses a cloud provider's load balancer to expose the service externally. |
| **ExternalName** | Maps the service to an external DNS name. |

### Ingress

An **Ingress** routes external HTTP and HTTPS traffic to services inside the cluster.

Instead of exposing many LoadBalancer services, you expose one Ingress and define routing rules like:

```
/api   → service-a
/blog  → service-b
/      → service-c
```

Use it for web applications that need path-based or host-based routing.

### ConfigMap

A **ConfigMap** stores non-sensitive configuration data, such as environment variables or configuration files.

Use it to separate configuration from container images.

### Secret

A **Secret** stores sensitive data such as passwords, tokens, or TLS certificates.

Secrets are base64-encoded by default. For production, use additional encryption or a secret management tool.

### Namespace

A **Namespace** partitions a cluster into virtual sub-clusters.

Use namespaces to separate environments (dev, staging, prod) or teams.

### StatefulSet

A **StatefulSet** is like a Deployment but for stateful applications.

It provides:

- Stable, unique network identities.
- Stable persistent storage.
- Ordered deployment and scaling.

Use it for databases like PostgreSQL, MongoDB, or distributed systems like Kafka and ZooKeeper.

### DaemonSet

A **DaemonSet** ensures that a copy of a pod runs on every node (or selected nodes).

Use it for log collectors, monitoring agents, or node-level proxies.

### Job / CronJob

A **Job** runs a pod to completion, then stops.

A **CronJob** runs Jobs on a schedule.

Use Jobs for one-off tasks like database migrations. Use CronJobs for scheduled tasks like backups or reports.

### PersistentVolume and PersistentVolumeClaim

A **PersistentVolume (PV)** is a piece of storage in the cluster, provisioned by an administrator or dynamically by a storage class.

A **PersistentVolumeClaim (PVC)** is a request for storage by a pod.

Use them when containers need durable storage that survives pod restarts.

---

## 6. Kubernetes Architecture Summary

| Object | What It Manages | When to Use It |
|---|---|---|
| **Pod** | One or more containers | Basic unit; usually managed by controllers |
| **Deployment** | Stateless pods with replicas | Web apps, APIs, workers |
| **StatefulSet** | Stateful pods with stable identity | Databases, distributed stateful systems |
| **DaemonSet** | One pod per node | Log collectors, monitors, node agents |
| **Job / CronJob** | Pods that run to completion | Migrations, backups, scheduled tasks |
| **Service** | Stable networking for pods | Any workload that needs to be reached |
| **Ingress** | HTTP/HTTPS routing | Web apps with multiple paths or domains |
| **ConfigMap** | Non-sensitive config | Environment variables, config files |
| **Secret** | Sensitive data | Passwords, tokens, certificates |
| **Namespace** | Cluster partitioning | Teams, environments, isolation |
| **PVC / PV** | Persistent storage | Databases, file storage, caches |

---

## 7. Declarative vs Imperative

Kubernetes is designed around a **declarative** model.

| Approach | Meaning | Example |
|---|---|---|
| **Imperative** | You give step-by-step commands | "Run this container now" |
| **Declarative** | You describe the desired state | "I want 3 replicas of this app" |

You declare the desired state in YAML files and apply them:

```bash
kubectl apply -f deployment.yaml
```

Kubernetes then continuously compares the actual state with the desired state and corrects differences.

This makes systems self-healing: if a pod dies, Kubernetes creates a new one. If a node fails, pods are rescheduled elsewhere.

---

## 8. Example Deployment

Here is a simple Kubernetes Deployment and Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: myapp:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

This deploys three replicas of `myapp:1.0` and exposes them through a LoadBalancer service.

---

## 9. Common kubectl Commands

| Command | What It Does |
|---|---|
| `kubectl get pods` | List pods |
| `kubectl get nodes` | List nodes |
| `kubectl apply -f file.yaml` | Apply a configuration file |
| `kubectl delete -f file.yaml` | Delete resources from a file |
| `kubectl describe pod <name>` | Show detailed pod information |
| `kubectl logs <pod>` | View pod logs |
| `kubectl exec -it <pod> -- sh` | Open a shell inside a pod |
| `kubectl port-forward <pod> 8080:8080` | Forward a local port to a pod |
| `kubectl scale deployment <name> --replicas=5` | Scale a deployment |
| `kubectl rollout status deployment/<name>` | Check rollout progress |
| `kubectl rollout undo deployment/<name>` | Roll back a deployment |

---

## 10. Kubernetes vs Docker Compose

| Aspect | Docker Compose | Kubernetes |
|---|---|---|
| Scope | Single machine, local or small setups | Multi-machine clusters, production |
| Scaling | Manual or basic | Automatic, fine-grained |
| Self-healing | Limited | Built-in |
| Rolling updates | Basic | Advanced with health checks |
| Service discovery | DNS on local network | Cluster DNS, Services, Ingress |
| Storage | Local volumes | PersistentVolumes, storage classes |
| Complexity | Low | High |
| Best for | Development, small apps, prototypes | Production, large systems |

Use Docker Compose for local development and simple deployments. Use Kubernetes when you need resilience, scale, and production-grade orchestration.

---

## 11. Kubernetes vs Docker Swarm

| Aspect | Docker Swarm | Kubernetes |
|---|---|---|
| Ease of use | Simpler | Steeper learning curve |
| Features | Basic orchestration | Rich ecosystem and extensibility |
| Scalability | Good | Excellent |
| Community | Smaller | Very large |
| Tooling | Limited | Vast: Helm, operators, service meshes |
| Cloud support | Less integrated | Native integration with all major clouds |

Docker Swarm is a valid choice for simple clusters. Kubernetes is the standard for complex, cloud-native systems.

---

## 12. Kubernetes vs Nomad

| Aspect | HashiCorp Nomad | Kubernetes |
|---|---|---|
| Workload types | Containers, binaries, JVM, batch | Primarily containers |
| Complexity | Simpler | More complex |
| Scheduler | Flexible, multi-region | Container-focused |
| Ecosystem | Smaller | Larger |
| Integration | Works well with Consul and Vault | Cloud-native ecosystem |

Nomad is a good alternative if you need to schedule more than containers or want a simpler operational model.

---

## 13. When to Use Kubernetes

Use Kubernetes when:

- You run many containers across multiple machines.
- You need high availability and automatic failover.
- You want automated scaling based on load.
- You deploy frequently and need rolling updates and rollbacks.
- You need service discovery, load balancing, and observability at scale.
- You are building cloud-native or microservices architectures.

Avoid Kubernetes when:

- You have a single small application or a few containers.
- Your team lacks the operational expertise to run it.
- A simpler platform-as-a-service (PaaS) like Heroku, Fly.io, or AWS ECS meets your needs.
- The added complexity is not justified by the scale.

---

## 14. Common Use Cases

| Use Case | Why Kubernetes Helps |
|---|---|
| **Microservices** | Each service runs independently, scales independently, and communicates through Services. |
| **CI/CD** | Automate deployments, canary releases, and rollbacks. |
| **High availability** | Failed pods and nodes are replaced automatically. |
| **Auto-scaling** | Scale pods and nodes based on CPU, memory, or custom metrics. |
| **Multi-cloud or hybrid cloud** | Run the same workloads across different cloud providers or on-premises. |
| **Machine learning workloads** | Schedule training jobs, manage GPUs, and serve models. |

---

## 15. Typical Kubernetes Workflow

```
Containerize app  ---->  Write Dockerfile
        |
        v
Build and push image  ---->  docker build && docker push
        |
        v
Write manifests  ---->  deployment.yaml, service.yaml, ingress.yaml
        |
        v
Apply to cluster  ---->  kubectl apply -f ./manifests
        |
        v
Monitor and scale  ---->  kubectl logs, kubectl top, HPA
```

---

## 16. Summary

- Kubernetes automates the deployment, scaling, and operation of containerized applications.
- A cluster has a **Control Plane** that makes decisions and **Worker Nodes** that run workloads.
- A **Pod** is the basic unit; controllers like **Deployments** and **StatefulSets** manage pods.
- **Services** provide stable networking; **Ingress** routes external HTTP traffic.
- **ConfigMaps** and **Secrets** separate configuration from code.
- Kubernetes is declarative: you describe the desired state, and the system maintains it.

Kubernetes adds complexity, but it also provides the foundation for resilient, scalable, and portable production systems.
