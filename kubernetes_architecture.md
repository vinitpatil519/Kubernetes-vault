# Kubernetes Architecture Explained

This document explains the key components that make up the architecture of a Kubernetes cluster, in simple terms.

## Table of Contents

- [Control Plane (Master Node Components)](#control-plane-master-node-components)
- [Worker Node Components](#worker-node-components)
- [Other Components](#other-components)

---

![Kubernetes Architecture Diagram](https://miro.medium.com/v2/resize:fit:1400/1*0Sudxeu5mQyN3ahi1FV49A.png)

```mermaid
flowchart LR
    subgraph CP["Control Plane"]
        API["API Server"]
        ETCD["ETCD"]
        SCHED["Scheduler"]
        CM["Controller Manager"]
        CCM["Cloud Controller Manager"]
    end

    subgraph WN1["Worker Node 1"]
        KUBELET1["kubelet"]
        PROXY1["kube-proxy"]
        RT1["Container Runtime"]
        POD1["Pod"]
    end

    subgraph WN2["Worker Node 2"]
        KUBELET2["kubelet"]
        PROXY2["kube-proxy"]
        RT2["Container Runtime"]
        POD2["Pod"]
    end

    KCTL["kubectl"] --> API
    API --- ETCD
    API --- SCHED
    API --- CM
    API --- CCM
    API --> KUBELET1
    API --> KUBELET2
    KUBELET1 --> RT1 --> POD1
    KUBELET2 --> RT2 --> POD2
    PROXY1 -.-> POD1
    PROXY2 -.-> POD2
```

> **Figure:** Control plane components on the left, worker node components on the right — every instruction flows through the API Server.

### What happens when you run `kubectl apply`

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant API as API Server
    participant ETCD as ETCD
    participant Sched as Scheduler
    participant Kubelet as kubelet
    participant CRI as Container Runtime

    Dev->>API: kubectl apply -f deployment.yaml
    API->>API: Authenticate, authorize, validate
    API->>ETCD: Persist desired state
    ETCD-->>API: Write acknowledged
    API-->>Dev: Resource created
    Sched->>API: Watch for unscheduled Pods
    Sched->>API: Bind Pod to a Node
    API->>ETCD: Persist Pod binding
    Kubelet->>API: Watch Pods assigned to this Node
    Kubelet->>CRI: Pull image and start container
    CRI-->>Kubelet: Container running
    Kubelet->>API: Report Pod status Running
```

> **Figure:** The control plane workflow triggered by a single `kubectl apply`.



## Control Plane (Master Node Components)

### API Server

This is the "front desk" of Kubernetes. Whenever you want to interact with your cluster, your request goes through the API Server. It validates and processes these requests to the backend components.

### etcd

Think of this as the "database" of Kubernetes. It stores all the information about your cluster—what nodes are part of the cluster, what pods are running, what their statuses are, and more.

```mermaid
flowchart LR
    KCTL["kubectl"] --> API["API Server"]
    API -->|"Write desired state"| ETCD["ETCD<br/>key-value store"]
    ETCD -->|"Read current state"| API
    API -->|"Watch events"| CM["Controller Manager"]
    API -->|"Watch events"| SCHED["Scheduler"]
    CM -->|"Reconcile toward desired state"| API
```

> **Figure:** ETCD is the single source of truth — only the API Server talks to it, and controllers reconcile the cluster toward the state stored there.


### Scheduler

The "event planner" for your containers. When you ask for a container to be run, the Scheduler decides which machine (Node) in your cluster should run it. It considers resource availability and other constraints while making this decision.

```mermaid
flowchart TD
    POD["Pod created<br/>nodeName empty"] --> API["API Server"]
    API --> SCHED["Scheduler"]
    SCHED --> FILTER["Filtering<br/>list feasible Nodes"]
    FILTER --> RES["Resource check<br/>CPU · memory · ports"]
    RES --> CONSTR["Node selector · affinity<br/>taints and tolerations"]
    CONSTR --> SCORE["Scoring<br/>rank feasible Nodes"]
    SCORE --> PICK["Node selected"]
    PICK --> BIND["Pod bound to Node"]
    BIND --> KUBELET["kubelet on that Node starts the Pod"]
    CONSTR -->|"No feasible Node"| PENDING["Pod stays Pending"]
```

> **Figure:** How the Scheduler filters, scores and finally binds a Pod to a worker node.


### Controller Manager

Imagine a bunch of small robots that continuously monitor the cluster to make sure everything is running smoothly. If something goes wrong (e.g., a Pod crashes), they work to fix it, ensuring the cluster state matches your desired state.

### Cloud Controller Manager

This is a specialized component that allows Kubernetes to interact with the underlying cloud provider, like AWS or Azure. It helps in tasks like setting up load balancers and persistent storage.

---

## Worker Node Components

### kubelet

This is the "manager" for each worker node. It ensures all containers on the node are healthy and running as they should be.

### kube-proxy

Think of this as the "traffic cop" for network communication either between Pods or from external clients to Pods. It helps in routing the network traffic appropriately.

### Container Runtime

This is the software used to run containers. Docker is commonly used, but other runtimes like containerd can also be used.

---

## Other Components

### Pod

The smallest unit in Kubernetes, a Pod is a group of one or more containers. Think of it like an apartment in an apartment building.

```mermaid
stateDiagram-v2
    [*] --> Pending: Pod object created in ETCD
    Pending --> ContainerCreating: Scheduler binds Pod to a Node
    ContainerCreating --> Running: kubelet pulls image and starts containers
    Running --> Succeeded: All containers exit with code 0
    Running --> Failed: A container exits non-zero
    ContainerCreating --> Failed: Image pull or mount error
    Running --> Pending: Pod evicted or rescheduled
    Succeeded --> [*]
    Failed --> [*]
```

> **Figure:** Pod lifecycle states — the kubelet drives every transition from `ContainerCreating` onward and reports each state back to the API Server.


### Service

This is like a phone directory for Pods. Since Pods can come and go, a Service provides a stable "address" so that other parts of your application can find them.

```mermaid
flowchart LR
    CLIENT["Client Pod"] --> SVC["Service<br/>ClusterIP 10.96.0.20:80"]
    SVC --> PROXY["kube-proxy<br/>iptables / IPVS rules"]
    PROXY --> POD1["Pod 1<br/>10.244.1.5:8080"]
    PROXY --> POD2["Pod 2<br/>10.244.2.7:8080"]
    PROXY --> POD3["Pod 3<br/>10.244.3.9:8080"]
    EP["Endpoints / EndpointSlice<br/>maintained by Controller Manager"] -.-> SVC
```

> **Figure:** A Service gives a stable virtual IP; kube-proxy load balances each connection across the healthy Pods behind it.

```mermaid
flowchart LR
    PODA["Pod A"] -->|"http://backend.default.svc.cluster.local"| DNS["CoreDNS"]
    DNS -->|"Returns ClusterIP"| PODA
    PODA -->|"Connect to ClusterIP"| SVC["Service: backend"]
    SVC --> PODB["Pod B"]
```

> **Figure:** Service discovery — CoreDNS resolves the Service name to its ClusterIP, which is why Pods can reach each other by name instead of by IP.


### Volume

This is like an external hard-drive that can be attached to a Pod to store data.

### Namespace

A way to divide cluster resources among multiple users or teams. Think of it as having different folders on a shared computer, where each team can only see their own folder.

### Ingress

Think of this as the "front door" for external access to your applications, controlling how HTTP and HTTPS traffic should be routed to your services.

```mermaid
flowchart TD
    USER["User opens the website"] --> NET["Internet"]
    NET --> LB["Cloud Load Balancer"]
    LB --> IC["Ingress Controller<br/>NGINX"]
    IC --> RULES["Ingress rules<br/>host and path matching"]
    RULES --> SVC["Service<br/>ClusterIP"]
    SVC --> PROXY["kube-proxy"]
    PROXY --> POD["Pod"]
    POD --> CTR["Container"]
    CTR --> APP["Application"]
    APP --> RESP["Response returned to the user"]
```

> **Figure:** The complete request lifecycle — from a browser on the internet all the way to the application process inside a container, and back.


---

And there you have it! That's a simplified breakdown of Kubernetes architecture components.


