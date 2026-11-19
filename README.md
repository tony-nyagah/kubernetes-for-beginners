# Kubernetes for Beginners (KodeKloud) - Notes

Personal notes following KodeKloud's *Kubernetes for the Absolute Beginners* course.

## Contents

1. [Containers Overview](#1-containers-overview)
2. [Container Orchestration](#2-container-orchestration)
3. [Kubernetes Architecture](#3-kubernetes-architecture)
4. [Docker vs ContainerD](#4-docker-vs-containerd)
5. [Kubernetes Setup (Minikube)](#5-kubernetes-setup-minikube)
6. [Pods](#6-pods)
7. [YAML Introduction](#7-yaml-introduction)
8. [Pods with YAML](#8-pods-with-yaml)
9. [Replication Controllers and ReplicaSets](#9-replication-controllers-and-replicasets)
10. [Deployments](#10-deployments)

---

## 1. Containers Overview

- A **container** packages an application with its libraries and dependencies so it runs the same everywhere.
- Containers share the host OS kernel, so they are lighter and faster to start than VMs (which each carry a full guest OS).
- **Docker** is the most popular tool for building and running containers.
- An **image** is the package/template; a **container** is a running instance of an image.

## 2. Container Orchestration

- Running one container is easy. Running hundreds across many hosts needs **orchestration**.
- An orchestrator handles:
  - Deploying containers across multiple hosts
  - Scaling up/down with demand
  - Networking between containers
  - Self-healing (replacing failed containers/nodes)
- Options: Docker Swarm, Mesos, **Kubernetes** (the most widely adopted).

## 3. Kubernetes Architecture

**Node**: a machine (physical or virtual) where containers run. Also called a worker/minion.

**Cluster**: a set of nodes, so the app stays available if one node fails.

**Master / control plane components**
| Component | Role |
|---|---|
| kube-apiserver | Front door of the cluster; all `kubectl` and internal communication goes through it |
| etcd | Distributed key-value store holding all cluster data |
| kube-scheduler | Decides which node a new pod is placed on |
| Controllers | Notice and respond when nodes/containers go down; keep desired state |
| Container runtime | Software that runs containers (Docker, containerd, ...) |

**Worker node components**
| Component | Role |
|---|---|
| kubelet | Agent on each node; makes sure containers run as expected and reports back to the API server |
| Container runtime | Runs the containers |
| kube-proxy | Handles networking rules so services/pods can talk to each other |

## 4. Docker vs ContainerD

- Kubernetes originally only supported Docker.
- Kubernetes introduced the **CRI (Container Runtime Interface)** so any OCI-compliant runtime can be used.
- Docker itself isn't CRI compliant, so Kubernetes used a shim (`dockershim`), which was **removed in v1.24**.
- **containerd** (a component of Docker) is CRI compatible and now a standalone runtime used by Kubernetes.
- CLI tools:
  - `ctr`: low-level, debugging containerd only (not user friendly)
  - `nerdctl`: Docker-like CLI for containerd
  - `crictl`: CLI for any CRI runtime, used for debugging (made by the Kubernetes community)

## 5. Kubernetes Setup (Minikube)

- **Minikube** runs a single-node cluster locally (needs a hypervisor or a container driver such as Docker).
- **kubectl** is the CLI used to talk to the cluster.

```bash
minikube start        # start a local cluster
minikube status
kubectl get nodes     # verify the node is Ready
minikube stop
```

## 6. Pods

- A **Pod** is the smallest deployable object in Kubernetes.
- A pod wraps one (usually) or more containers.
- Scale by adding **more pods**, not more containers inside one pod.
- Containers in the same pod share network (localhost) and storage. Multi-container pods are for helpers/sidecars.

```bash
kubectl run nginx --image=nginx      # create a pod
kubectl get pods
kubectl get pods -o wide             # shows node and IP
kubectl describe pod nginx
kubectl delete pod nginx
```

## 7. YAML Introduction

YAML represents data. Three basic types:

```yaml
# Key-value pairs
Fruit: Apple
Vegetable: Carrot

# Arrays / lists (ordered, items start with "-")
Fruits:
  - Orange
  - Apple

# Dictionaries / maps (indentation matters, use spaces not tabs)
Banana:
  Calories: 105
  Fat: 0.4
```

- Lists are ordered, dictionaries are not.
- Indentation defines structure. Wrong spacing = wrong or broken data.
- `#` starts a comment.

## 8. Pods with YAML

Every Kubernetes definition file has four top-level required fields:

| Field | Description | Pod value |
|---|---|---|
| `apiVersion` | Version of the Kubernetes API used for the object | `v1` |
| `kind` | Type of object | `Pod` |
| `metadata` | Data about the object (name, labels) | dict |
| `spec` | Desired specification (containers, etc.) | dict |

Example: `nginx.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-2
  labels:
    env: production
spec:
  containers:
    - name: nginx
      image: nginx
```

- `metadata.name` is a string; `labels` is a free-form dictionary used to group/filter objects.
- `spec.containers` is a **list** (note the `-`), because a pod can have multiple containers.

```bash
kubectl create -f nginx.yaml      # create from file
kubectl apply -f nginx.yaml       # create or update from file
kubectl get pods
kubectl describe pod nginx-2
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml   # generate YAML template
kubectl edit pod nginx-2          # edit a running pod
```

**YAML tips**
- Generate a template with `--dry-run=client -o yaml` instead of writing from scratch.
- Use a editor with YAML support (indentation guides, spaces for tabs).

---

## 9. Replication Controllers and ReplicaSets

> Covers Replication Controllers, ReplicaSets, labels/selectors and scaling.

### Why replication?

- **High availability**: run multiple copies of a pod so the app survives if one crashes. Even with a single pod, a replication controller will bring it back automatically.
- **Load balancing & scaling**: spread load across pods, and add more pods (even across nodes) as demand grows.

### Replication Controller (old) vs ReplicaSet (new)

| | Replication Controller | ReplicaSet |
|---|---|---|
| `apiVersion` | `v1` | `apps/v1` |
| Status | Older, being replaced | Recommended |
| `selector` | Not required (inferred from template labels) | **Required** |

### Replication Controller definition

Example: `rc-definition.yaml`

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  template:
    # this part is the pod definition
    metadata:
      name: myapp-prod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx
  replicas: 3
```

Structure:
- The outer `metadata` describes the **ReplicationController** itself.
- `spec.template` holds the **pod definition** (its own `metadata` and `spec`), copied from a pod file without `apiVersion` and `kind`.
- `spec.replicas` is the number of pods to keep running.

```bash
kubectl create -f rc-definition.yaml
kubectl get replicationcontroller      # or: kubectl get rc
kubectl get pods                       # pods are named myapp-rc-xxxxx
```

### ReplicaSet definition

Example: [`replicasets/replicaset.yaml`](replicasets/replicaset.yaml)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp-prod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: front-end
```

Differences from the Replication Controller:
- `apiVersion` is `apps/v1`. Using `v1` gives an error because the kind isn't found in that API group.
- `spec.selector` is **required**. `matchLabels` matches pods whose labels equal the given key/values.
- The selector must match the labels in the pod template, otherwise the ReplicaSet is rejected.

### Labels and selectors

- A ReplicaSet can manage pods that were **not** created by it. It uses the selector to find existing pods with matching labels.
- This is why the selector exists: the RS watches all pods in the cluster and keeps the number of matching ones equal to `replicas`.
- The `template` is still required, because the RS needs it to create new pods when the count drops below the desired number.
- Example: 3 pods already labeled `type: front-end` exist. A ReplicaSet with that selector adopts them and won't create extra ones. It only creates new pods if one dies.

### Scaling

Three ways to change the number of replicas:

```bash
# 1. Edit replicas in the file, then replace
kubectl replace -f replicaset-definition.yaml

# 2. Scale by file (does NOT change the number in the file)
kubectl scale --replicas=6 -f replicaset-definition.yaml

# 3. Scale by type and name (file also unchanged)
kubectl scale replicaset myapp-replicaset --replicas=6

# Or edit the live object directly
kubectl edit replicaset myapp-replicaset
```

### Useful commands

```bash
kubectl create -f replicaset-definition.yaml
kubectl get replicaset              # or: kubectl get rs
kubectl get pods                    # pods are named myapp-replicaset-xxxxx
kubectl describe replicaset myapp-replicaset
kubectl delete replicaset myapp-replicaset   # also deletes its pods
kubectl explain replicaset          # look up fields/apiVersion
```

Tip: delete one of the pods and run `kubectl get pods` again. The RS creates a replacement right away to restore the desired count.

### Summary

- Both keep a set number of pods running (self-healing, scaling, load balancing).
- **ReplicaSet** is the modern replacement and adds a required label `selector`, which allows it to manage pods created outside it.
- In practice you rarely create ReplicaSets directly. **Deployments** (a later topic) manage ReplicaSets for you.

---

## 10. Deployments

> Covers Deployments and the start of update/rollback. Rolling update demo and lab are still to do.

### Why Deployments?

A real app in production needs more than just running pods:
- Many instances running for availability and load.
- **Upgrade** the app smoothly (new image versions) without taking it down. Replacing all instances at once would cause downtime and a bad version would hit all users.
- **Roll back** quickly if a new version has problems.
- Make several changes (image, replicas, resources) and apply them together: **pause**, then **resume**.

A **Deployment** sits above the ReplicaSet and provides all of this.

```
Deployment
  └── ReplicaSet
        └── Pods
```

### Deployment definition

Example: [`deployments/deployment.yaml`](deployments/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    tier: frontend
    app: caddy
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 3
  template:
    metadata:
      name: caddy
      labels:
        app: myapp
    spec:
      containers:
        - name: caddy
          image: caddy:2.11.8
```

- Identical to a ReplicaSet definition except `kind: Deployment`.
- `selector.matchLabels` must match the pod `template.metadata.labels`.
- Pin an image **tag** (e.g. `caddy:2.11.8`) instead of relying on `latest`, so updates and rollbacks are predictable.

### Commands

```bash
kubectl create -f deployments/deployment.yaml
kubectl get deployments            # or: kubectl get deploy
kubectl get replicaset             # the Deployment created this automatically
kubectl get pods                   # named <deployment>-<rs-hash>-<pod-hash>
kubectl get all                    # deployment, replicaset and pods together
kubectl describe deployment myapp-deployment

# Generate a YAML template quickly
kubectl create deployment caddy --image=caddy:2.11.8 --replicas=3 --dry-run=client -o yaml > deployment.yaml
```

### Updates and rollouts

- Creating a deployment triggers a **rollout**, which creates a new **revision** (revision 1).
- Changing the deployment (e.g. a new image version) triggers a new rollout and revision (revision 2). This lets you roll back.

```bash
kubectl rollout status deployment/myapp-deployment      # watch progress
kubectl rollout history deployment/myapp-deployment     # list revisions
```

### Deployment strategies

| Strategy | How it works |
|---|---|
| **Recreate** | Destroy all old pods, then create all new ones. Causes **downtime**. |
| **RollingUpdate** (default) | Take down old pods and bring up new ones **one at a time**. App stays available. |

```yaml
spec:
  strategy:
    type: RollingUpdate     # or Recreate
```

You can see this in `kubectl describe deployment`: the Deployment scales the **new ReplicaSet up** and the **old one down** gradually. With Recreate, the old ReplicaSet is scaled to 0 first.

### Updating a deployment

```bash
# Option 1: edit the file, then apply (preferred, keeps the file as source of truth)
kubectl apply -f deployments/deployment.yaml

# Option 2: change the image directly (file becomes out of date!)
kubectl set image deployment/myapp-deployment caddy=caddy:2.11.9

# Option 3: edit the live object
kubectl edit deployment myapp-deployment
```

- In `set image`, the part before `=` is the **container name**, not the image name.
- During an update a **second ReplicaSet** is created. The old one is kept with 0 pods, which is what makes rollback possible.

### Recording the change cause

`kubectl rollout history` has a `CHANGE-CAUSE` column that shows `<none>` unless the `kubernetes.io/change-cause` annotation is set. The old `--record` flag did this automatically but is **deprecated**.

```bash
# set it right after a change (it's stamped on the current revision)
kubectl annotate deployment/myapp-deployment kubernetes.io/change-cause="update caddy to 2.11.9"
```

Or in the YAML, so it's recorded on every `apply`:

```yaml
metadata:
  name: myapp-deployment
  annotations:
    kubernetes.io/change-cause: "caddy 2.11.9"
```

More detail on revisions:

```bash
kubectl rollout history deployment/myapp-deployment --revision=2   # image, labels for that revision
kubectl get rs                                                     # one ReplicaSet per revision
kubectl describe deployment myapp-deployment                       # events show old/new RS scaling
```

**Revision numbers can jump or disappear.** If a new pod template matches an older ReplicaSet (e.g. after a rollback or re-applying an old image), Kubernetes reuses that ReplicaSet and gives it the **next** revision number, so the old number vanishes (history like `1, 4, 5` becomes `1, 5, 6`).

### Rollback

```bash
kubectl rollout undo deployment/myapp-deployment                 # previous revision
kubectl rollout undo deployment/myapp-deployment --to-revision=1 # specific revision
```

After a rollback the old ReplicaSet scales back up and the new one goes to 0.

### Pause and resume

```bash
kubectl rollout pause deployment/myapp-deployment
# make several changes (image, resources, ...) without triggering rollouts
kubectl rollout resume deployment/myapp-deployment
```

### Summary of commands

| Task | Command |
|---|---|
| Create | `kubectl create -f file.yaml` |
| Get | `kubectl get deployments` |
| Update | `kubectl apply -f file.yaml` / `kubectl set image ...` |
| Status | `kubectl rollout status deployment/<name>` |
| History | `kubectl rollout history deployment/<name>` |
| Rollback | `kubectl rollout undo deployment/<name>` |

### Gotchas

- In `kubectl set image deployment/<name> <container>=<image>`, `<container>` is the **container name** from the YAML (`caddy`), not the image or deployment name. A wrong name gives `unable to find container named "..."`.
- A typo in the deployment name gives `NotFound`. Check with `kubectl get deploy`.
- `connection refused ... 127.0.0.1:6443` means the cluster isn't running, not a YAML problem. On k3s: `sudo systemctl start k3s`, and set `KUBECONFIG=/etc/rancher/k3s/k3s.yaml` if kubectl can't find the config.

### TODO

- [ ] Demo: Deployments - Update and Rollback (hands-on notes)
- [ ] Lab: Practice Test Rolling Updates and Rollbacks
