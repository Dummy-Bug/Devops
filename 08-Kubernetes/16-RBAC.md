The last production gap in the Ingress note's table was that applications must not have unrestricted access to the cluster. The two previous notes made the Notes application scale; this one is about **who is allowed to do what** inside the cluster. The Ingress note named the terms in a line: **RBAC**, role-based access control, decides who may do what through the Kubernetes API, and a **service account** is the identity a pod uses when it calls that API. This note teaches both.

## Why permissions matter inside a cluster

Picture a Spring Boot pod and a MySQL pod in one cluster, and a ConfigMap (part of the same cluster) holding the Spring Boot application's configuration.

One distinction first, because the Notes API from the deployment note already reads a ConfigMap and needs no permission at all to do it. Its values arrive as environment variables through `configMapKeyRef`, and it is the kubelet that fetches them and hands them to the container when it starts; the application itself never talks to the Kubernetes API. The questions below are about the other kind of pod: one whose code calls the Kubernetes API directly, to read ConfigMaps, list pods or change Deployments while it runs.

- If such a pod can read the ConfigMap, should it also be able to **delete** it? No; nothing about its job requires that.
- Another pod runs some background task. Should it be able to store things in the database? Perhaps not. Delete the database? Certainly not.
- Should any of them be able to delete the Spring Boot pod? No.

A production cluster runs many more things than this: Spring Boot services, MySQL databases, payment systems, analytics jobs, monitoring tools. People, CI/CD pipelines and running applications all talk to the cluster, and none of them should have the same rights by default:

| Who | Needs | Should not be able to |
|---|---|---|
| a developer who reads logs | to inspect logs | delete the payment Deployment |
| an application that reads ConfigMaps | to read ConfigMaps | read Secrets, delete pods |
| a monitoring system | read-only access to specific resources | administer everything |

```mermaid
flowchart LR
    D["Developer"] -->|"read logs"| A["API server"]
    P["Application pod"] -->|"read ConfigMaps"| A
    M["Monitoring system"] -->|"read nodes, pods"| A
    A -->|"each request checked<br/>against its caller's rights"| C["Cluster objects"]
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef inert fill:#3a3a3a,stroke:#5a5a5a,stroke-width:1.5px,color:#fff
    class D,P,M actor
    class A mech
    class C inert
```

The reason is not only mistakes. **If an attacker breaks into one pod, whatever that pod is allowed to do, the attacker can now do.** A pod with every right lets them harm every other pod; a pod with narrow rights limits the damage to what those rights allow.

## Authentication and authorization

Two different questions get asked of every request:

| Concept | Question | Meaning |
|---|---|---|
| **Authentication** | who are you? | establishes the identity making the request (logging in, for example) |
| **Authorization** | what are you allowed to do? | decides whether that identity has permission for the requested action |

On Instagram, a logged-in user is authenticated, but is not authorized to do what an Instagram administrator does. Someone who only watches YouTube videos has an account, but no YouTube Studio (the part of YouTube where videos are uploaded and managed), so they are not authorized to upload or manage videos; they can only watch. **Authorization is about capabilities, not identity.**

Kubernetes uses both, and RBAC is how it does the second. Take a command that deletes a pod:

```bash
kubectl delete pod mysql-xyz
```

It sends a request to the API server. The API server establishes who sent it, then checks whether that identity is allowed to delete that pod, and only then allows or refuses the operation.

```mermaid
flowchart TB
    K["kubectl delete pod mysql-xyz"] -->|"request, with the<br/>caller's credentials"| N
    subgraph API["API server"]
        N["Check 1: Authentication<br/>who is asking?"] -->|"identity, e.g.<br/>minikube-user"| Z{"Check 2: Authorization (RBAC)<br/>does a rule allow<br/>this identity to delete pods?"}
    end
    Z -->|"yes"| D["Delete goes ahead"]
    Z -->|"no"| F["403 Forbidden<br/>stops here"]
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    classDef broken fill:#7a1f1f,stroke:#b03030,stroke-width:1.5px,color:#fff
    class K actor
    class N,Z mech
    class D good
    class F broken
```

> [!important] **RBAC governs the Kubernetes API, not your application's endpoints.**
> It decides whether an identity may get, create or delete Kubernetes objects. It does not decide whether a customer may call an endpoint of the Spring Boot application; that is the application's own security, such as Spring Security, and it is entirely separate.

## Four objects

| Object | Purpose | Scope |
|---|---|---|
| **Role** | defines permissions on resources within a namespace | one namespace |
| **RoleBinding** | grants a Role's permissions (or a ClusterRole's) to an identity, in a namespace | one namespace |
| **ClusterRole** | defines permissions that can include cluster-wide resources, or be reused across namespaces | the whole cluster |
| **ClusterRoleBinding** | grants a ClusterRole's permissions across the whole cluster | the whole cluster |

**A Role or ClusterRole defines permissions. A binding gives them to someone.** Two separate manifests, and neither does anything without the other. In SQL terms, a Role is a row in a `roles` table listing what the role may do, and a RoleBinding is a row in a `user_roles` join table connecting that role to a user. Spring Security has the same split: a role is a set of authorities, and granting it to a user is a separate step.

```mermaid
flowchart LR
    R["Role<br/>configmap-reader"] -->|"named in roleRef"| RB["RoleBinding<br/>one namespace"]
    CR["ClusterRole<br/>node-reader"] -->|"named in roleRef"| CRB["ClusterRoleBinding<br/>whole cluster"]
    CR -.->|"can also be named by"| RB
    RB -->|"subjects"| SA["Identity<br/>user, group or<br/>service account"]
    CRB -->|"subjects"| SA
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    classDef step fill:#7a5a1f,stroke:#b08a2e,stroke-width:1.5px,color:#fff
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    class R,CR mech
    class RB,CRB step
    class SA actor
```

### A Role

This Role allows reading ConfigMaps in the namespace `devops-demo`, the namespace the previous note's in-place resize example used (`kubectl create namespace devops-demo` creates it). Recall from the local cluster note that a **namespace** is a named partition of one cluster, keeping different sets of objects apart; most objects belong to one.

```yaml
# k8s/rbac/configmap-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: devops-demo
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

| Field | Whose value | Means |
|---|---|---|
| `apiVersion: rbac.authorization.k8s.io/v1` | Kubernetes' fixed name | the RBAC part of the API; the request goes to `/apis/rbac.authorization.k8s.io/v1/...` the way an order goes to `POST /api/v1/orders` |
| `kind: Role` | fixed name | a set of permissions in one namespace |
| `name: configmap-reader` | yours, free text | what bindings will call this Role |
| `namespace: devops-demo` | yours | the Role applies in this namespace only |
| `rules` | fixed name | the list of permissions; each entry adds some |
| `apiGroups: [""]` | Kubernetes' group names | the API group the resources belong to (see below) |
| `resources: ["configmaps"]` | Kubernetes' resource names, plural and lower case | the permissions are about ConfigMaps |
| `verbs: ["get", "list"]` | Kubernetes' fixed verbs, such as `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` | reading one ConfigMap by name, and listing them |

What the Role does not say is denied. It grants no `create`, `update` or `delete` on ConfigMaps, and it grants nothing at all on Secrets, Deployments or pods. **RBAC permissions are purely additive: there are no deny rules**, so whatever no rule grants is refused.

> [!bug] **`apiGroups: [""]` is the core API group only, not every group.**
> It is tempting to read the empty string as a wildcard. It is not. Kubernetes resources belong to API groups: ConfigMaps, pods, Secrets, Services, nodes and namespaces belong to the original **core** group, whose `apiVersion` is plain `v1` (served under `/api/v1`), and the core group's name is the empty string. Deployments belong to the `apps` group, which is why their `apiVersion` is `apps/v1` (served under `/apis/apps/v1`).
> A Role granting `get` and `list` on `deployments` with `apiGroups: [""]` lets its holder list no Deployments at all; the same rule with `apiGroups: ["apps"]` works. Nothing reports the mistake: a rule that names the wrong group silently grants nothing. `kubectl api-resources` shows each resource's group.

### A RoleBinding

Defining a Role does not give it to anybody. A RoleBinding connects the permission definition to an identity, here a service account named `notes-reader` (service accounts are the next section):

```yaml
# k8s/rbac/configmap-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: notes-configmap-binding
  namespace: devops-demo
subjects:
- kind: ServiceAccount
  name: notes-reader
  namespace: devops-demo
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

| Field | Whose value | Means |
|---|---|---|
| `kind: RoleBinding` | fixed name | a grant, valid in this binding's namespace |
| `name: notes-configmap-binding` | yours, free text | the binding's own name |
| `subjects` | fixed name | who receives the permissions; a list, so one binding can name several |
| `subjects[].kind` | one of `User`, `Group`, `ServiceAccount` | what sort of identity it is |
| `subjects[].name`, `namespace` | yours | which one: the service account `notes-reader` in `devops-demo` |
| `roleRef` | fixed name | which permissions are granted |
| `roleRef.kind` | `Role` or `ClusterRole` | where to look for them |
| `roleRef.name` | yours | the Role `configmap-reader`, matched by name |
| `roleRef.apiGroup` | fixed, always `rbac.authorization.k8s.io` | the group the Role belongs to |

The binding is in `devops-demo`, so the permissions apply there and nowhere else.

## Service accounts

Anything that calls the Kubernetes API falls into one of two groups:

| Identity | Example |
|---|---|
| **a human** | a DevOps engineer running `kubectl get nodes`, `kubectl get pods`, and the rest |
| **a workload** | an application inside a pod that needs to query the Kubernetes API: to read a ConfigMap, list pods, or update a Deployment |

Kubernetes has no objects that represent human users. A person proves who they are with a credential that kubectl sends: on Minikube, a client certificate that `minikube start` created, whose location it recorded in the kubeconfig file. The certificate's common name becomes the username and its organization fields become the groups; that is the whole identity. `kubectl auth whoami` shows it:

```text
Username   minikube-user
Groups     [system:masters system:authenticated]
```

That is why a RoleBinding's `subjects` can name three kinds of identity: a `User` or a `Group` for people, and a `ServiceAccount` for workloads. The Minikube user is in the group `system:masters`, which is bound to the built-in ClusterRole **`cluster-admin`**: a role whose single rule grants every verb on every resource in every API group. That is why everything in this folder so far has simply worked from kubectl.

A **ServiceAccount** is a Kubernetes identity for a **workload**. It is to the Kubernetes API what `spring.datasource.username` is to MySQL: the account a program, not a person, uses. The rules are attached to the service account, and every pod that runs as it gets them.

```yaml
# k8s/rbac/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notes-reader
  namespace: devops-demo
```

`apiVersion: v1` because service accounts are in the core group; `name` and `namespace` are yours. A pod runs as a service account when its Deployment's pod template (`spec.template.spec`) names it:

```yaml
serviceAccountName: notes-reader
containers:
  - name: notes-app
    image: notes-app:v1
```

| Field | Whose value | Means |
|---|---|---|
| `serviceAccountName` | Kubernetes' fixed name | which identity the pod's API calls use |
| `notes-reader` | yours | must match a ServiceAccount's name in the same namespace |

The Deployment, its pods and the service account must be in the same namespace.

### How the pod proves who it is

The pod does not log in by hand. In a modern cluster:

1. The pod is assigned a service account.
2. Kubernetes places a short-lived token for that service account inside the pod's filesystem, at `/var/run/secrets/kubernetes.io/serviceaccount/token`, alongside `ca.crt` (the certificate needed to trust the API server) and `namespace`.
3. The application, or the Kubernetes client library it uses, sends that token with each request to the API server.
4. The API server authenticates the token as the service account, and RBAC decides what it may do.

```mermaid
---
config:
  sequence:
    mirrorActors: false
---
sequenceDiagram
    autonumber
    box rgba(176,138,46,0.45) Node
        participant KL as kubelet
        participant App as Application<br/>in the pod
    end
    box rgba(47,110,166,0.45) API server
        participant API as Authentication
        participant RBAC as RBAC
    end
    KL->>App: mount token, ca.crt, namespace
    App->>API: request + Bearer token
    API->>RBAC: this is notes-reader
    RBAC-->>App: 200 or 403
```

**A service account is an identity. It grants nothing by itself;** the RoleBindings and ClusterRoleBindings that name it decide what it can do.

### Asking what an identity may do

With the ServiceAccount, Role and RoleBinding above applied in `devops-demo`, a ConfigMap and a Secret give them something to be tried against:

```bash
kubectl create configmap app-settings -n devops-demo --from-literal=MODE=demo
kubectl create secret generic db-pass -n devops-demo --from-literal=password=x
```

`kubectl auth can-i` answers what an identity may do. A service account's identity is written `system:serviceaccount:<namespace>:<name>`:

```bash
kubectl auth can-i get configmaps -n devops-demo --as=system:serviceaccount:devops-demo:notes-reader
```

For `notes-reader`, it answers:

| Question | Answer |
|---|---|
| get, list configmaps in devops-demo | yes |
| create, delete configmaps in devops-demo | no |
| get secrets in devops-demo | no |
| delete pods, delete deployments in devops-demo | no |
| get configmaps in default | no |
| list nodes | no |

### From inside a pod

The `kubectl auth can-i` table above only asks a question: would `notes-reader` be allowed? This subsection makes the requests for real, from a pod running as `notes-reader`, the way an application would send them:

```mermaid
flowchart TB
    P["Pod api-client<br/>runs as notes-reader"] -->|"GET the ConfigMap<br/>app-settings"| A1["API server:<br/>200, here it is"]
    P -->|"GET the Secret<br/>db-pass"| A2["API server:<br/>403 Forbidden"]
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    classDef broken fill:#7a1f1f,stroke:#b03030,stroke-width:1.5px,color:#fff
    class P actor
    class A1 good
    class A2 broken
```

**Step 1: a pod that plays the application.** A real Spring Boot application would call the API through a Kubernetes Java client library. Here the stand-in is the simplest client there is, `curl`:

```yaml
# k8s/rbac/api-client-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-client
  namespace: devops-demo
spec:
  serviceAccountName: notes-reader
  containers:
  - name: curl
    image: curlimages/curl:8.11.1
    command: ["sleep", "3600"]
```

| Field | Whose value | Means |
|---|---|---|
| `name: api-client` | yours, free text | the pod's name |
| `namespace: devops-demo` | yours | must be the service account's namespace |
| `serviceAccountName: notes-reader` | the name is yours, it must match the ServiceAccount | **the line that matters**: every API call from this pod carries the identity `notes-reader` |
| `image: curlimages/curl:8.11.1` | a published image | a small image that contains `curl` |
| `command: ["sleep", "3600"]` | yours | the container does nothing for an hour, which keeps the pod alive so that commands can be run inside it |

Two more pods differ only in name: `default-client` leaves out the `serviceAccountName` line, and `no-token` adds `automountServiceAccountToken: false` (used further down).

**Step 2: finding the API server from inside the pod.** The same way the Spring Boot pod found MySQL: through the name of a ClusterIP Service.

| The pod wants | It uses the name | Which is the ClusterIP Service |
|---|---|---|
| MySQL | `mysql-service:3306` | `mysql-service`, whose pod list is the MySQL pod |
| the API server | `https://kubernetes.default.svc` (port 443) | `kubernetes`, which every cluster has in the `default` namespace, and whose endpoint is the API server itself |

On Minikube, the ClusterIP Service `kubernetes` is at `10.96.0.1:443` and forwards to the API server at `192.168.49.2:8443`. The name `kubernetes.default.svc` follows the Services note's DNS rule: Service name, then namespace, then `svc`.

**Step 3: the request, piece by piece.** Inside `api-client`, using its own token:

```bash
D=/var/run/secrets/kubernetes.io/serviceaccount
curl --cacert $D/ca.crt -H "Authorization: Bearer $(cat $D/token)" https://kubernetes.default.svc/api/v1/namespaces/devops-demo/configmaps/app-settings
curl --cacert $D/ca.crt -H "Authorization: Bearer $(cat $D/token)" https://kubernetes.default.svc/api/v1/namespaces/devops-demo/secrets/db-pass
```

| Piece | Means | The Spring equivalent |
|---|---|---|
| `D=/var/run/secrets/kubernetes.io/serviceaccount` | a shortcut for the folder where the kubelet placed the three files, as the previous subsection described | |
| `--cacert $D/ca.crt` | trust the API server's HTTPS certificate | adding a certificate to the JVM's truststore |
| `-H "Authorization: Bearer $(cat $D/token)"` | read the token file and send it as the caller's identity | sending a JWT in the `Authorization` header |
| `/api/v1/namespaces/devops-demo/configmaps/app-settings` | a REST path: the ConfigMap `app-settings` in the namespace `devops-demo` | `GET /notes/42` |

**This is the same request `kubectl` sends.** `kubectl get configmap app-settings -n devops-demo`, run with request logging turned on (`-v=6`), shows it calling the same path:

```text
verb="GET" url="https://192.168.49.2:8443/api/v1/namespaces/devops-demo/configmaps/app-settings" status="200 OK"
```

**Step 4: the results match the `can-i` table.** Same identity, same rules, but this time the requests were real:

| Request | `can-i` said | The real answer |
|---|---|---|
| get the ConfigMap `app-settings` | yes | `200` and the ConfigMap itself |
| get the Secret `db-pass` | no | `403` |

```text
403 secrets "db-pass" is forbidden: User "system:serviceaccount:devops-demo:notes-reader" cannot get resource "secrets" in API group "" in the namespace "devops-demo"
```

> [!note] **The token expires, so the application must re-read the file.**
> The token works like a JWT with an expiry time. Its mount declares an expiry of 3,607 seconds, about an hour. Once the token is older than 80% of that (roughly 48 minutes), the kubelet fetches a fresh one and writes it into the same file. An application that read the file once at startup and kept the token in a variable would go on sending the old one after it expired, and the API server rejects a token it cannot accept with `401 Unauthorized`. So reloading is the application's job: re-reading the file on a schedule, say every five minutes, is usually enough.

> [!failure] **A pod that names no service account.**
> A pod without `serviceAccountName` runs as its namespace's `default` service account. The same calls from `default-client` are all refused, even the ConfigMap:
> ```text
> 403 configmaps "app-settings" is forbidden: User "system:serviceaccount:devops-demo:default" cannot get resource "configmaps" in API group "" in the namespace "devops-demo"
> ```
> The default service account has an identity and no permissions. That is the safe starting point.

### A ClusterRole for cluster-wide resources

Some resources belong to no namespace. Nodes are one: `kubectl get nodes` lists the nodes of the whole cluster. A monitoring application that needs to list them needs a ClusterRole:

```yaml
# k8s/rbac/node-reader-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

The fields are the Role's, with `kind: ClusterRole` and no `namespace`, because a ClusterRole belongs to the whole cluster. A ClusterRoleBinding gives it to the monitoring identity across the cluster. Before any binding, `notes-reader` gets `403` listing nodes, with the message `cannot list resource "nodes" in API group "" at the cluster scope`. After

```bash
kubectl create clusterrolebinding node-reader-binding --clusterrole=node-reader --serviceaccount=devops-demo:notes-reader
```

it can list nodes (the same request returns `200`) and still cannot delete them.

## Does every Spring Boot application need a service account?

No. A typical Spring Boot application handles HTTP requests and talks to MySQL:

```text
HTTP request → Spring Boot controller → MySQL
```

**If it never calls the Kubernetes API** (never lists pods, reads ConfigMaps through the API or updates Deployments), **it needs no Kubernetes API permissions at all.** It still runs as the namespace's `default` service account, and there is no reason to give that account anything.

**A service account token is for the Kubernetes API only.** It does not let a pod connect to MySQL, and it is not how one pod talks to another. A database connection is ordinary networking to the MySQL Service, with the database's own username and password, as the ConfigMaps and Secrets note set up.

```mermaid
flowchart LR
    S["Spring Boot pod"] -->|"username and password<br/>from a Secret"| DB["MySQL Service"]
    S -->|"service account token<br/>only if it calls the API"| A["Kubernetes API server"]
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    classDef mech fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
    class S actor
    class DB good
    class A mech
```

For an application that never needs the API, the token need not be placed in the pod at all. In the pod template (`spec.template.spec`):

```yaml
automountServiceAccountToken: false
```

| Field | Whose value | Means |
|---|---|---|
| `automountServiceAccountToken` | Kubernetes' fixed name | whether the kubelet mounts the token files into the pod |
| `false` | `true` or `false`, default `true` | no token, `ca.crt` or `namespace` file is placed in the pod |

In the `no-token` pod, the token directory does not exist:

```text
ls: /var/run/secrets/kubernetes.io/serviceaccount: No such file or directory
```

If that container is ever compromised, there is no Kubernetes credential in it to steal.

## The principle of least privilege

**Give every identity only the permissions it needs to do its job — the minimum, and nothing more.** If `notes-reader` only reads ConfigMap values, it gets read access and no write access. If the Spring Boot application genuinely needs to create ConfigMaps, give it create; otherwise do not. This is the **principle of least privilege**, and it applies to developers, CI/CD pipelines, monitoring tools, Kubernetes controllers and application pods alike.

| A compromised Spring Boot container whose workload has | What an attacker gets |
|---|---|
| excessive permissions, such as a binding to `cluster-admin`, the everything role | with its Kubernetes credentials, the ability to read or change much of the cluster |
| least privilege: read ConfigMaps in its own namespace | through that identity, only that; it cannot delete Deployments, read Secrets or touch cluster-wide resources |

```mermaid
flowchart LR
    subgraph WIDE["Bound to cluster-admin"]
        X1["Attacker in<br/>the pod"] -->|"uses its token"| W["read Secrets,<br/>delete Deployments,<br/>change the cluster"]
    end
    subgraph NARROW["Bound to configmap-reader"]
        X2["Attacker in<br/>the pod"] -->|"uses its token"| N["read ConfigMaps<br/>in devops-demo only"]
    end
    classDef actor fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
    classDef broken fill:#7a1f1f,stroke:#b03030,stroke-width:1.5px,color:#fff
    classDef good fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
    class X1,X2 actor
    class W broken
    class N good
```

The goal is to limit what an application or a person can do if their credentials are misused. **Least privilege does not stop a pod being broken into; it limits what the break-in is worth.**

> [!question] **Is it mandatory to use RBAC?**
> On a modern cluster it is already on, and it is not something to bolt on later. Minikube's API server runs with `--authorization-mode=Node,RBAC`: two authorizers, consulted in turn — `Node`, a special-purpose authorizer for the requests kubelets make, and `RBAC` for everyone else. As the `default-client` pod showed, a pod's default service account can do nothing at all until a binding gives it something. So the choice is not whether to have RBAC but what to grant: a cluster where every workload is bound to `cluster-admin` has thrown the protection away. Writing narrow Roles is what puts least privilege into practice.

> [!question] **What is the difference between DevOps and DevSecOps?**
> In practice, very little: you need to know all of it. The separation into titles matters less than the responsibility, which should be end to end: from thinking of an idea, to writing it, deploying it, managing it, and making it sustainable and scalable. A developer should know DevOps, and security is part of the same job. Rather than playing with terms, call yourself an engineer and learn what the work requires.
