The architecture note described the scheduler choosing a node for a container and the kubelet getting that container started, and set one level of the hierarchy aside: the pod, sitting between the node and the container. This note picks up the question it skipped — does Kubernetes place and manage individual containers directly?

It is the obvious assumption, and it looks like it would work. A worker node already runs the `user-service`, `payment-service` and `order-service` containers. A fourth container needs placing, and the node has a core and some RAM free. The scheduler could pick this node for the new container, that node's kubelet could tell its container runtime to start it, and the job is done. Nothing in that chain is broken.

**Normally, Kubernetes does not do this.** It wraps containers in a unit called a **Pod**, and to see why, look at an application that is not just one container.

## The problem: containers that belong together

The order service runs in one container. Next to it is a second container — a **helper**, the small companion program the first note mentioned. A helper is not a microservice. Microservices are separate applications that talk to each other over HTTP, so two of them can live on different servers without any issue. A helper exists only to support one main application:

- collecting its logs, which the application writes and the helper ships somewhere;
- holding configuration, or heavy extra libraries, that the application uses;
- acting as a proxy for its network traffic, or writing proxy-related information;
- preparing or creating files for it;
- running scheduled jobs — the periodic tasks Linux calls cron jobs — that nobody wants inside the main application.

Some of that could go inside the application's own container — and sometimes it does. When it is heavy, you keep it in a separate container so the main one stays lean. Some teams do this, some never need to.

Now let the scheduler place containers individually. Nothing stops it from doing this:

```mermaid
flowchart LR
    S["Scheduler,<br/>placing containers one by one"] -->|"places"| N1["Node 1<br/>order-service"]
    S -->|"places"| N8["Node 8<br/>its helper"]
    N1 -.->|"the helper is meant to be<br/>beside it, and is not"| N8
    classDef broken fill:#7a1f1f,stroke:#b03030,stroke-width:1.5px,color:#fff
    classDef step fill:#7a5a1f,stroke:#b08a2e,stroke-width:1.5px,color:#fff
    class S step
    class N1,N8 broken
```

The relationship breaks. The helper cannot, or should not, exist on its own; it is useful only next to the application it serves. What you want is simple: these two always land on the same node, together, every time.

## The answer: the Pod

Kubernetes' answer is to say: put both of them **in one Pod**.

> **A Pod is a group of related containers that are scheduled and managed together, on one node.**

The scheduler then places pods, not containers: it decides which node a pod goes to, and whatever is inside the pod — one container, two, or more — goes there together. Everything the architecture note said about placing and running containers holds with the pod as the unit: the scheduler places pods, the kubelet runs the pods assigned to its node, and the runtime creates the containers inside them. Kubernetes still manages your containers, but indirectly: what it looks after directly is the pod, and the containers come with it.

```mermaid
flowchart LR
    subgraph P["Pod — scheduled as one unit"]
        A["order-service<br/>container"]
        H["helper<br/>container"]
    end
    S["Scheduler"] -->|"places"| P
    P -->|"runs on"| N["Node 2"]
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    class S actor
    class A,H good
    class N mech
```

**Checked against the documentation:** Kubernetes defines Pods as the smallest deployable units of computing that you can create and manage. There is no smaller thing it deploys. The hierarchy from the architecture note is complete now:

> **Cluster → Node → Pod → Container**

## Most pods hold one container

Having said all that, **the most common pod by far contains exactly one container** — the application. Most applications have no helper at all, so the pod wraps just the order service and nothing else.

That can look strange — a pod for a single container? — but **a pod is still necessary**, because Kubernetes manages pods, not containers. Whether a pod holds one container or several, the pod is there. The documentation says the same: the one-container-per-Pod model is the most common Kubernetes use case.

> [!danger] **Never put separate microservices into one pod.**
> Pods group a main application with its own helpers. They are not a way of grouping an architecture. Taking `user-service`, `payment-service` and `order-service` — three microservices — and putting them in one pod is a very wrong approach, and makes no sense. Each gets its own pod: a pod for the user service with its container, one for payment, one for orders. If one of them needs a helper, the helper goes in that service's pod.
>
> The reason is everything from the first note. These services need to **scale independently, fail independently and deploy independently**. Bundle them, and the payment service cannot get five replicas without the other two getting five as well; a crash in one takes the pod down for all three; a new version of one means redeploying all three.

> [!danger] **Never put several replicas of one service into one pod either.**
> Can one service's multiple replicas share a pod? No. The whole value of replicas comes from keeping them apart, so that if one server goes down by mistake, the other replicas are still alive. Put them in one pod and they land on one node, and one node failing takes every replica with it — you have not replicated anything. **One pod means one application**; occasionally that application's helper rides along with it.

> [!question] **Do microservice architectures use helpers at all?**
> Yes. Even in a microservice architecture, a service often keeps separate helpers — for logs, for proxies, for libraries — and they are seen in real applications. They go in the same pod as the service they help.

## What containers in a pod share

Putting containers in one pod is not just a scheduling convenience. **Containers in the same pod share certain resources**, and the two that matter here are networking and storage.

### One network identity

**A pod gets its own IP address.** Say Kubernetes gives a pod the address `10.244.1.7`. That is the pod's address; to talk to the pod, you talk to that IP. **The containers inside it do not have IP addresses of their own** — they live inside the pod's single network identity.

That has a direct, useful consequence. If the order service listens on port 8080, its helper in the same pod reaches it at:

> `localhost:8080`

To each of them, the other is on localhost — exactly the way you run MySQL and Spring Boot side by side on your own laptop and connect with `localhost`. This is the biggest practical benefit of the pod: one network identity for the whole group, so its members talk to each other through localhost.

```mermaid
flowchart LR
    subgraph POD["Pod · IP 10.244.1.7"]
        APP["order-service<br/>listening on 8080"]
        HLP["helper"]
    end
    HLP -->|"localhost:8080"| APP
    OUT["Anything outside<br/>the pod"] -->|"10.244.1.7"| POD
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    class APP good
    class HLP mech
    class OUT actor
```

> [!info] **This was run, not just read.**
> A pod was started with two containers: one serving a page on port 8080, and a second doing nothing but waiting. Asked for their own addresses, **both containers answered with the same IP** — the pod's, which on that cluster was `10.42.0.10` rather than the example address above — and the second container fetched the first one's page from `http://localhost:8080/` and got it back. **Checked against the documentation:** every container in a pod shares the network namespace, including the IP address and network ports, and inside a pod — and only there — containers can reach each other on localhost.

> [!bug] **Going further: shared ports cut both ways.**
> Sharing one network namespace means sharing one set of ports. Two containers in the same pod cannot both listen on 8080, any more than two programs on your laptop can. Tried on the check cluster with two web servers both told to use 8080, the first started and the second died on startup, over and over, leaving the pod at one of two containers ready:
> ```text
> portclash   1/2   Error   2 (19s ago)   20s
> httpd: bind: Address already in use
> ```
> Containers in different pods have no such conflict, because each pod has its own IP.

### Shared storage

Containers in a pod can also share **volumes** — the same idea as the volume in the first note, storage that is not inside any one container, here mounted into several containers of the same pod at once. The order service writes its log to `/app/logs/app.log`; the log helper in the same pod mounts the same volume and reads the same file, and ships it wherever logs go.

```mermaid
flowchart LR
    APP["order-service<br/>writes /app/logs/app.log"] -->|"writes"| V["Shared volume"]
    V -->|"read by"| HLP["log helper<br/>reads the same file"]
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    class APP,HLP good
    class V mech
```

The kinds of volume Kubernetes offers are a subject of their own, outside this folder. For now: **same pod → can share networking → can share volumes.** On the check cluster, an application container appended a line to a log file on a shared volume and the helper container, mounting that volume at a different path, read the line straight back.

> [!question] **Is a helper container in the same pod called a sidecar?**
> **Yes, and the documentation confirms it.** Kubernetes calls them **sidecar containers** — secondary containers that run alongside the main application container within the same Pod. **Going further:** Kubernetes now has built-in support for sidecars that start before the main container and keep running beside it; it has been on by default since v1.29 and stable since v1.33.

> [!question] **Can a pod be brought up and down like starting and stopping a service, to change the application's properties?**
> That belongs with interacting with pods, which is where the next note starts: creating a pod by hand, inspecting it and deleting it — and why pods are not managed by hand at all. Changing an application's settings without rebuilding anything is the subject of the ConfigMaps note.

> [!note] **Going further: you rarely create pods yourself.**
> The documentation is direct about it: usually you do not create pods directly, even for a single container, but through a workload resource such as a **Deployment** — the object from the architecture note that holds the three-replicas description, keeps a ReplicaSet, and has the pods created from it. That is how the deletion test in the first note was set up, and why a deleted pod came back.
