# retail-store-gitops

The **deployment repository** for [`retail-store`](https://github.com/Muhamed404/retail-store), a five-service shop running on **on-premises servers only** (no cloud services).

This repo describes everything that runs on the servers: the Helm charts and per-environment values ArgoCD deploys, the cluster add-ons bare metal needs, monitoring and logging, and the Ansible playbooks that prepare the servers in the first place. Application code, Dockerfiles and Jenkinsfiles live in the app repo.

The whole delivery is driven and visualised by **Liveflow**, a separate dashboard application with its own repo and CI/CD (see [How Liveflow uses this repo](#how-liveflow-uses-this-repo)).

> **Status:** every file below exists but is an **empty skeleton**. This README explains what each one is for, so they can be filled in one at a time. `.gitignore` is the only file with content, because this repo is public.

---

## Contents

1. [How the pieces fit](#how-the-pieces-fit)
2. [Environments](#environments)
3. [Repository structure](#repository-structure)
4. [`ansible/`: preparing the servers](#ansible-preparing-the-servers)
5. [`platform/`: cluster add-ons](#platform-cluster-add-ons)
6. [`k8s/`: plain manifests](#k8s-plain-manifests)
7. [`helm/`: the charts ArgoCD deploys](#helm-the-charts-argocd-deploys)
8. [`argocd/`: what gets deployed where](#argocd-what-gets-deployed-where)
9. [`monitoring/`: metrics, dashboards and logs](#monitoring-metrics-dashboards-and-logs)
10. [Secrets: this repo is public](#secrets-this-repo-is-public)
11. [How Liveflow uses this repo](#how-liveflow-uses-this-repo)
12. [Build order](#build-order)

---

## How the pieces fit

```
                     retail-store (app repo)                     retail-store-gitops (this repo)
developer push ──► Jenkins: test, build, scan ──► Harbor ──┐
                                                           │ Jenkins commits the new image tag
                                                           ▼
                                        helm/environments/values-stg.yaml / values-prod.yaml
                                                           │
                                              ArgoCD watches this repo
                                                           ▼
                                     cluster-stg (auto-sync)    cluster-prod (manual sync)

        Liveflow sits on top: reads Jenkins, ArgoCD, Kubernetes, Prometheus, Loki; opens the prod gate
```

Two repos keep code and deployment apart: Jenkins' tag commits land here, so they never re-trigger a build; this repo's history is exactly what ran where; and developers don't need write access to production settings.

---

## Environments

| Environment | Runs on | Deployed by | Sync | Values file |
|---|---|---|---|---|
| dev | Docker Compose on a dev server | Jenkins over SSH (app repo) | — | `.env` on the server |
| staging | Kubernetes `cluster-stg` | ArgoCD | **automatic** | `helm/environments/values-stg.yaml` |
| prod | Kubernetes `cluster-prod` (separate cluster) | ArgoCD | **manual**: someone clicks Sync in Liveflow | `helm/environments/values-prod.yaml` |

---

## Repository structure

```
retail-store-gitops/
├── README.md
├── .gitignore                         keeps keys, kubeconfigs and unsealed secrets out of Git
│
├── ansible/                           prepares bare servers (runs from a laptop or a jump host)
│   ├── ansible.cfg
│   ├── inventory/hosts.ini
│   ├── group_vars/all.yml
│   ├── playbooks/
│   │   ├── common.yml
│   │   ├── k8s-cluster.yml
│   │   ├── dev-server.yml
│   │   ├── jenkins.yml
│   │   └── harbor.yml
│   └── templates/harbor.yml.j2
│
├── platform/                          add-ons every on-prem cluster needs, installed once per cluster
│   ├── metallb/        values.yaml, ip-pool-stg.yaml, ip-pool-prod.yaml
│   ├── ingress-nginx/  values.yaml
│   ├── cert-manager/   values.yaml, cluster-issuer.yaml
│   ├── longhorn/       values.yaml
│   ├── sealed-secrets/ values.yaml
│   └── argocd/         values.yaml
│
├── k8s/                               plain YAML version of the app (learning step, not deployed by ArgoCD)
│   ├── namespace.yaml
│   ├── ingress.yaml
│   ├── <service>/      deployment.yaml, service.yaml, configmap.yaml, hpa.yaml, secret.example.yaml   (×5)
│   └── dependencies/   mysql-catalog, postgres-orders, redis-checkout, rabbitmq, dynamodb-cart
│
├── helm/                              what ArgoCD actually deploys
│   ├── charts/
│   │   ├── <service>/  Chart.yaml, values.yaml, .helmignore, templates/   (×5: ui catalog cart orders checkout)
│   │   │     templates/: _helpers.tpl, deployment.yaml, service.yaml, configmap.yaml,
│   │   │                 sealedsecret.yaml, hpa.yaml, servicemonitor.yaml
│   │   └── dynamodb-local/  Chart.yaml, values.yaml, templates/: _helpers.tpl, statefulset.yaml, service.yaml
│   ├── retail-store/   Chart.yaml, values.yaml, templates/ingress.yaml   (parent chart: the whole shop)
│   └── environments/   values-stg.yaml, values-prod.yaml
│
├── argocd/
│   ├── root-app.yaml                  "app of apps": the one app you apply by hand
│   ├── projects/retail-store.yaml
│   └── applications/
│       ├── platform.yaml
│       ├── monitoring.yaml
│       ├── retail-store-stg.yaml
│       └── retail-store-prod.yaml
│
└── monitoring/
    ├── prometheus/
    │   ├── values.yaml                kube-prometheus-stack settings (Prometheus, Grafana, Alertmanager)
    │   └── rules/      alerts.yaml, recording-rules.yaml
    ├── grafana/
    │   ├── datasources.yaml
    │   └── dashboards/ overview.json, services.json, kubernetes.json
    ├── loki/           values.yaml, promtail-values.yaml
    └── compose/                       monitoring for the dev server (Docker, not Kubernetes)
        ├── docker-compose.monitoring.yml
        ├── prometheus.yml
        └── promtail-config.yml
```

---

## `ansible/`: preparing the servers

On-prem there is no cloud console that hands you a ready cluster. Ansible installs everything on bare Linux servers in a repeatable way, so a server can be rebuilt from this repo instead of from memory.

| File | Purpose |
|---|---|
| `ansible.cfg` | Ansible settings for this repo: inventory path, SSH user, privilege escalation, no host-key prompts in the lab. |
| `inventory/hosts.ini` | The list of servers, grouped by role: `cluster_stg_control`, `cluster_stg_workers`, `cluster_prod_control`, `cluster_prod_workers`, `dev_server`, `jenkins`, `harbor`. Playbooks target groups, never single IPs. |
| `group_vars/all.yml` | Values shared by every playbook: Kubernetes version, container runtime version, pod and service CIDRs, NTP servers, the admin user. Passwords do **not** go here; use an Ansible Vault file (ignored by Git). |
| `playbooks/common.yml` | Runs on **every** server first: OS updates, time sync, disable swap, kernel modules and sysctls Kubernetes needs (`overlay`, `br_netfilter`, IP forwarding), install containerd, firewall basics. |
| `playbooks/k8s-cluster.yml` | Installs Kubernetes (kubeadm or k3s) on one cluster's group: control plane first, then joins the workers, then installs the CNI (e.g. Calico or Flannel). Run it twice: once for `cluster-stg`, once for `cluster-prod`. Saves the kubeconfig locally (ignored by Git). |
| `playbooks/dev-server.yml` | Prepares the dev server: Docker Engine + Compose plugin, a `deploy` user Jenkins connects to over SSH, and the folder Jenkins copies `docker-compose.yml` into. |
| `playbooks/jenkins.yml` | Installs Jenkins with the plugins Liveflow depends on: Pipeline, Git, GitHub, Credentials, **Pipeline Stage View** (Liveflow reads stages from it). Creates the API user Liveflow uses (Overall/Read, Job/Read, Job/Build, Job/Cancel). |
| `playbooks/harbor.yml` | Installs **Harbor**, the private image registry Jenkins pushes to and the clusters pull from. Replaces Docker Hub / ECR. |
| `templates/harbor.yml.j2` | Harbor's own configuration file, filled from Ansible variables: hostname, HTTPS certificate, data volume, admin password (from Vault). |

---

## `platform/`: cluster add-ons

A cloud Kubernetes service comes with load balancers, disks and certificates built in. A bare-metal cluster has none of them. These add-ons fill the gaps and are installed **once per cluster**, before the app. Each folder holds the Helm values for that add-on's official chart; ArgoCD installs them through `argocd/applications/platform.yaml`.

| Folder / file | What it solves |
|---|---|
| `metallb/values.yaml` | **MetalLB** gives `LoadBalancer` services a real IP on your network. Without it, they stay `<pending>` forever on bare metal and nothing outside the cluster can reach the shop. |
| `metallb/ip-pool-stg.yaml` | The free IP range MetalLB may hand out on **cluster-stg** (e.g. `192.168.10.240-250`). An `IPAddressPool` + `L2Advertisement`. |
| `metallb/ip-pool-prod.yaml` | The same for **cluster-prod**, with a different range: each cluster has its own pool. |
| `ingress-nginx/values.yaml` | **ingress-nginx** is the single front door. It takes one IP from MetalLB and routes requests by host name (e.g. `shop.stg.lab`) to the right service. |
| `cert-manager/values.yaml` | **cert-manager** issues and renews HTTPS certificates automatically. |
| `cert-manager/cluster-issuer.yaml` | Tells cert-manager **where** certificates come from: your **internal CA**. Public Let's Encrypt can't reach on-prem servers, so an internal CA is used and trusted once on the machines that browse the shop. |
| `longhorn/values.yaml` | **Longhorn** turns the servers' local disks into persistent volumes, replicated across nodes. Without it a database pod that restarts or moves loses all its data. A simpler option for a lab is `local-path` (no replication). |
| `sealed-secrets/values.yaml` | **Sealed Secrets** lets encrypted secrets live in this public repo. Only the cluster's controller can decrypt them (see [Secrets](#secrets-this-repo-is-public)). |
| `argocd/values.yaml` | Settings for **ArgoCD itself** (repo access, RBAC for the Liveflow API account, UI options), so ArgoCD's own configuration is also in Git. ArgoCD is the one tool installed by hand first; after that it can manage its own upgrade from these values. |

Install order on a new cluster: MetalLB → Longhorn → ingress-nginx → cert-manager → Sealed Secrets → ArgoCD → (ArgoCD takes over) monitoring → app.

---

## `k8s/`: plain manifests

The app written as **plain Kubernetes YAML**, one file per object, applied with `kubectl apply -f`. This is the learning step: it shows exactly what Kubernetes needs before Helm templates hide it. **ArgoCD does not deploy this folder**; once the Helm charts work, `k8s/` stays as a readable reference.

| File | Purpose |
|---|---|
| `namespace.yaml` | The `retail-store` namespace everything lives in. |
| `ingress.yaml` | Routes the shop's host name to the `ui` service through ingress-nginx. Only `ui` is exposed; the other four are internal. |
| `<service>/deployment.yaml` | How to run the service: image from Harbor, port 8080, liveness/readiness probes on its health path, CPU/memory requests and limits (the HPA needs requests to work). |
| `<service>/service.yaml` | A stable internal name (`catalog`, `cart` ...) other services call on port 8080. |
| `<service>/configmap.yaml` | Non-secret settings: endpoints of the other services, persistence provider, database host and name. |
| `<service>/hpa.yaml` | HorizontalPodAutoscaler: min/max replicas and the CPU target. The stress test exists to make this react, and Liveflow reads its numbers live; replicas are **never** set by hand. |
| `<service>/secret.example.yaml` | Shows which secret keys the service needs (database password ...) with **fake values**. Copy to `secret.yaml` locally to try it; real `secret.yaml` files are ignored by Git. |
| `dependencies/mysql-catalog.yaml` | StatefulSet + Service + PersistentVolumeClaim for catalog's MySQL. |
| `dependencies/postgres-orders.yaml` | The same for orders' PostgreSQL. |
| `dependencies/redis-checkout.yaml` | Redis for checkout. |
| `dependencies/rabbitmq.yaml` | RabbitMQ that orders publishes events to. |
| `dependencies/dynamodb-cart.yaml` | **DynamoDB Local** for cart, the on-prem stand-in for Amazon DynamoDB (same API, no AWS account). |

---

## `helm/`: the charts ArgoCD deploys

### `helm/charts/<service>/` (×5)

One chart per service, so each can be versioned and tested on its own.

| File | Purpose |
|---|---|
| `Chart.yaml` | Chart name and version. |
| `values.yaml` | Defaults that are the same in every environment: port, probes, resource requests, HPA min/max/target, image repository. The **image tag** is not set here; it comes from the environment file. |
| `.helmignore` | Files Helm leaves out of the packaged chart. |
| `templates/_helpers.tpl` | Shared snippets: chart name, labels and selectors, so every object is labelled the same way (Liveflow and Prometheus select pods by these labels). |
| `templates/deployment.yaml` | The Deployment, built from values. |
| `templates/service.yaml` | The internal Service on port 8080. |
| `templates/configmap.yaml` | Non-secret settings from values. |
| `templates/sealedsecret.yaml` | The service's secrets as a **SealedSecret**: encrypted values from the environment file, decrypted only inside the cluster. |
| `templates/hpa.yaml` | The HorizontalPodAutoscaler. |
| `templates/servicemonitor.yaml` | Tells Prometheus to scrape the service (`/actuator/prometheus` for ui, cart, orders; `/metrics` for catalog and checkout). This is what feeds Liveflow's CPU, request and error charts. |

### `helm/charts/dynamodb-local/`

DynamoDB Local has no official chart, so it gets a small one: `Chart.yaml`, `values.yaml`, and templates for a single-replica `statefulset.yaml` with a Longhorn volume, a `service.yaml` on port 8000, and `_helpers.tpl`.

### `helm/retail-store/`: the parent chart

The whole shop as **one release**, so one ArgoCD app deploys everything together.

| File | Purpose |
|---|---|
| `Chart.yaml` | Lists the dependencies: the five service charts and `dynamodb-local` (from `../charts/`), plus community charts for MySQL, PostgreSQL, Redis and RabbitMQ (check each chart's image licence before choosing one). |
| `values.yaml` | Wires them together: which database each service uses, shared labels, the ingress host. |
| `templates/ingress.yaml` | The single Ingress for the shop, pointing at `ui`. |

### `helm/environments/`

| File | Purpose |
|---|---|
| `values-stg.yaml` | Staging overrides: image tags (`stg-<sha>`, **written by Jenkins** after each staging build), host name `shop.stg.lab`, smaller HPA limits, sealed secrets encrypted for cluster-stg. |
| `values-prod.yaml` | Production overrides: image tags (`vX.Y.Z`, **written by Jenkins** on a release or rollback), production host name, larger HPA limits, sealed secrets encrypted for cluster-prod. |

The image tag is the **only** thing Jenkins ever changes in this repo. Everything else is changed by people, through pull requests.

---

## `argocd/`: what gets deployed where

ArgoCD watches this repo and makes each cluster match it.

| File | Purpose |
|---|---|
| `root-app.yaml` | The **app of apps**: the only manifest applied by hand (`kubectl apply -f argocd/root-app.yaml`). It points ArgoCD at `argocd/applications/`, and ArgoCD creates everything else from there. |
| `projects/retail-store.yaml` | An ArgoCD **AppProject**: limits which repos, clusters and namespaces these apps may touch, so a mistake in one file can't deploy somewhere unexpected. |
| `applications/platform.yaml` | Installs and keeps the add-ons in `platform/` in sync on each cluster. |
| `applications/monitoring.yaml` | Installs `monitoring/` (kube-prometheus-stack, Loki, Promtail) on each cluster. |
| `applications/retail-store-stg.yaml` | The shop on **cluster-stg**: `helm/retail-store` + `values-stg.yaml`, **auto-sync** with self-heal. A Jenkins commit to `values-stg.yaml` deploys within moments. |
| `applications/retail-store-prod.yaml` | The shop on **cluster-prod**: `helm/retail-store` + `values-prod.yaml`, **no auto-sync**. After a release ArgoCD shows **OutOfSync** and waits; the production gate is someone clicking **Sync** in Liveflow. |

---

## `monitoring/`: metrics, dashboards and logs

| Path | Purpose |
|---|---|
| `prometheus/values.yaml` | Settings for **kube-prometheus-stack**, which installs Prometheus, Grafana, Alertmanager, node-exporter and kube-state-metrics together. Retention, storage on Longhorn, and picking up every `ServiceMonitor` the service charts create. |
| `prometheus/rules/alerts.yaml` | Alert rules: pod crash-looping, error rate above a threshold, HPA at max replicas, database down, disk almost full. |
| `prometheus/rules/recording-rules.yaml` | Pre-computed queries (requests per second and error rate per service) so dashboards and Liveflow stay fast. |
| `grafana/datasources.yaml` | Connects Grafana to Prometheus and Loki. |
| `grafana/dashboards/overview.json` | The whole shop on one screen: traffic, errors, latency, versions running. |
| `grafana/dashboards/services.json` | One row per service: requests, errors, latency, CPU, memory, replicas. |
| `grafana/dashboards/kubernetes.json` | Cluster health: nodes, pods, restarts, disk. |
| `loki/values.yaml` | **Loki** stores the logs of every pod (Prometheus only stores numbers, and a dead pod's logs disappear with it). |
| `loki/promtail-values.yaml` | **Promtail** runs on every node, reads pod logs and ships them to Loki with labels such as namespace and pod. |
| `compose/docker-compose.monitoring.yml` | For the **dev server**, which runs Docker Compose, not Kubernetes: Prometheus, **cAdvisor** (per-container CPU, memory, running or stopped), node-exporter (the server itself) and Promtail. Started next to the app's `docker-compose.yml`. |
| `compose/prometheus.yml` | That Prometheus's scrape config: cAdvisor, node-exporter and the five services' metrics paths. |
| `compose/promtail-config.yml` | Promtail on the dev server: reads Docker container logs and ships them to Loki, labelled with the Compose project and service. |

Grafana is reachable on its own, and Liveflow also embeds a Grafana dashboard and reads Prometheus and Loki directly.

---

## Secrets: this repo is public

Anyone on the internet can read this repo. A normal Kubernetes `Secret` is only **base64-encoded**, not encrypted: anyone can decode it in one command. So:

| Rule | How |
|---|---|
| Real secrets are committed **only as SealedSecrets** | Create a normal Secret locally, run `kubeseal` against the target cluster, commit only the sealed output. Each cluster has its own key, so staging and prod values are sealed separately. |
| Plain secrets never reach Git | `.gitignore` blocks `k8s/**/secret.yaml`, `*-unsealed.yaml`, `*.secret.yaml`. |
| Cluster keys never reach Git | `kubeconfig*`, `*.key`, `*.pem` and the Sealed Secrets controller backup key are ignored. Keep that backup key offline: without it, sealed secrets can't be recovered after a cluster rebuild. |
| Server passwords | Ansible Vault files, ignored by Git. |
| Fake values only | `k8s/<service>/secret.example.yaml` shows key names with placeholder values. |

---

## How Liveflow uses this repo

Liveflow never edits this repo. It reads the tools that act on it:

| Liveflow shows or does | Reads | Comes from this repo |
|---|---|---|
| ArgoCD station, Synced / OutOfSync | ArgoCD API | `applications/retail-store-stg.yaml`, `retail-store-prod.yaml` |
| the **production gate** (Sync button) | ArgoCD API | prod app is manual-sync |
| pods, rolling updates, HPA | Kubernetes API (read-only service account) | Deployments and HPAs from the charts |
| CPU, requests, errors charts | Prometheus | `servicemonitor.yaml` + kube-prometheus-stack |
| logs | Loki | Loki + Promtail |
| dev containers | Prometheus + cAdvisor on the dev server | `monitoring/compose/` |

In Liveflow, each environment lists **all five Deployments** of the namespace: `ui, catalog, cart, orders, checkout`. List `ui` first: it is the main service, whose version leads the page and whose HPA the stress test watches. Liveflow then shows every service with its version, pods, HPA and state, totals across all of them, and which service is rolling out or scaling.

---

## Build order

1. **Ansible**: `common.yml` on all servers, then `harbor.yml`, `jenkins.yml`, `dev-server.yml`.
2. **Clusters**: `k8s-cluster.yml` for cluster-stg, then cluster-prod.
3. **Platform** on cluster-stg: MetalLB, Longhorn, ingress-nginx, cert-manager, Sealed Secrets, ArgoCD.
4. **`k8s/`**: deploy the shop by hand with plain YAML to learn what it needs.
5. **`helm/`**: turn those manifests into the charts; test with `helm template` and `helm install` on staging.
6. **`argocd/`**: apply `root-app.yaml` and let ArgoCD take over staging.
7. **`monitoring/`**: kube-prometheus-stack, dashboards, Loki; then the dev server's `compose/` stack.
8. **Production**: repeat 3, 6 and 7 for cluster-prod, with `retail-store-prod.yaml` on manual sync.
