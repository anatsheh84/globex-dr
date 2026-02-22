# globex-dr — Globex Helm Chart for Disaster Recovery

A **Helm chart** packaging the full Globex retail microservices application, purpose-built for **Disaster Recovery (DR)** demonstrations on OpenShift. This chart is designed to be deployed and managed by **Red Hat Advanced Cluster Management (RHACM)** — the hub cluster orchestrates application placement across primary and DR clusters, monitors compliance, and drives the failover workflow without requiring manual `oc` or `helm` commands on individual spoke clusters.

This is the Helm-based companion to the plain-manifest [Globex](https://github.com/anatsheh84/Globex) repository, with key DR-specific differences: stateful databases use **persistent PVCs** (candidates for ODF volume replication), database pods require the `anyuid` SCC via ClusterRoleBindings, and certain services start with `replicas: 0` to represent a passive standby state on the DR cluster.

---

## Chart Information

| Field | Value |
|---|---|
| Chart Name | `globex` |
| Chart Version | `0.1.0` |
| App Version | `0.0.1` |
| Default Namespace | `globex` (configurable via `values.yaml`) |
| Type | `application` |

---

## Architecture — Globex Microservices

Globex is a fictional retail application composed of 7 microservices and 2 supporting databases:

```
                         ┌─────────────┐
                         │  globex-ui  │  (Node.js frontend)
                         └──────┬──────┘
           ┌──────────┬─────────┼─────────┬──────────────────┐
           ▼          ▼         ▼          ▼                  ▼
       ┌────────┐ ┌─────────┐ ┌──────────────────────┐ ┌────────────────┐
       │catalog │ │inventory│ │recommendation-engine  │ │order-placement │
       └───┬────┘ └────┬────┘ └──────────────────────┘ └────────────────┘
           │           │
    ┌──────┴───┐  ┌────┴───────────┐
    │ catalog- │  │ inventory-     │
    │ database │  │ database       │
    │(PostgreSQL) │(PostgreSQL)    │
    └──────────┘  └────────────────┘
           ▲
    ┌──────┴──────────────┐
    │ activity-tracking   │  (+ activity-tracking-simulator)
    └─────────────────────┘
```

---

## Microservices Summary

| Service | Image | Port | Replicas | Runtime |
|---|---|---|---|---|
| `globex-ui` | `quay.io/redhat-gpte/globex-recommendation-ui:app-mod-workshop` | 8080 | 1 | Node.js |
| `catalog` | `quay.io/redhat-gpte/globex-catalog:app-mod-workshop` | 8080 | 1 | Quarkus |
| `catalog-database` | `quay.io/redhat-gpte/globex-catalog-database:app-mod-workshop` | 5432 | 1 | PostgreSQL |
| `inventory` | `quay.io/redhat-gpte/globex-inventory:app-mod-workshop` | 8080 | 1 | Quarkus |
| `inventory-database` | `quay.io/redhat-gpte/globex-inventory-database:app-mod-workshop` | 5432 | 1 | PostgreSQL |
| `recommendation-engine` | `quay.io/redhat-gpte/globex-recommendation-engine:app-mod-workshop` | 8080 | **0** (standby) | Quarkus |
| `order-placement` | `quay.io/redhat-gpte/globex-order-placement-service:app-mod-workshop` | 8080 | **0** (standby) | Quarkus |
| `activity-tracking` | `quay.io/redhat-gpte/globex-activity-tracking:app-mod-workshop` | 8080 | 1 | Quarkus |
| `activity-tracking-simulator` | `quay.io/redhat-gpte/globex-activity-tracking-simulator:app-mod-workshop` | 8080 | 1 | Quarkus |

> **DR Note:** `recommendation-engine` and `order-placement` are deployed with `replicas: 0`. On the primary cluster these services are active; on the DR cluster they remain at zero replicas until ACM triggers a failover and the DR cluster is promoted.

---

## DR-Specific Design Decisions

This chart differs from the plain-manifest Globex repo in several important ways:

**1. Persistent Storage for Databases**
Both PostgreSQL databases use PVCs instead of ephemeral storage. In an ACM-orchestrated DR setup, these PVCs are backed by **ODF volume replication** (using a `VolumeReplicationGroup` on the primary cluster) so that data is continuously mirrored to the DR cluster's storage system.

| PVC | Size | Access Mode |
|---|---|---|
| `catalog-database` | 5Gi | ReadWriteOnce |
| `inventory-database` | 5Gi | ReadWriteOnce |

**2. `anyuid` SCC ClusterRoleBindings**
The database ServiceAccounts (`catalog-app`, `inventory-app`) are granted the `anyuid` Security Context Constraint via ClusterRoleBindings. This is required because the database containers include an `initContainer` (`alpine:3.16.0`) that changes filesystem permissions — an operation that requires running as arbitrary UIDs. ACM deploys these CRBs as part of the application subscription, so no manual cluster-admin intervention is needed on spoke clusters.

**3. Helm Templating with Namespace Injection**
The namespace is parameterised throughout via `{{ $.Values.globex.namespace }}`, making it straightforward for ACM to deploy to different namespaces on primary vs. DR clusters using different `values.yaml` overrides per placement.

**4. Health Probes on Quarkus Services**
`recommendation-engine` and `order-placement` include liveness (`/q/health/live`) and readiness (`/q/health/ready`) probes. When ACM scales these up during failover, OpenShift will only route traffic once the readiness probe passes — preventing premature traffic before the DB connections are fully established.

---

## Deploying with Red Hat ACM

This chart is intended to be deployed via **RHACM ApplicationSet** or a **Subscription + Channel** on the hub cluster. ACM handles placement decisions (which clusters receive the application) and can drive the DR failover by updating placement rules — no `helm install` is needed directly on spoke clusters.

### High-Level ACM DR Architecture

```
  ┌──────────────────────────────────────────────────────┐
  │                   ACM Hub Cluster                    │
  │                                                      │
  │  ┌────────────────┐     ┌──────────────────────────┐ │
  │  │  Application   │     │   Placement / DR Policy  │ │
  │  │  (globex-dr    │────▶│   (primary → DR cluster  │ │
  │  │   Helm chart)  │     │    on failover trigger)  │ │
  │  └────────────────┘     └──────────────────────────┘ │
  └──────────────┬────────────────────────┬──────────────┘
                 │                        │
   ┌─────────────▼───────┐   ┌────────────▼────────────┐
   │   Primary Cluster   │   │       DR Cluster        │
   │                     │   │                         │
   │  globex (all svcs   │   │  globex (DBs replicated │
   │  active, replicas≥1)│   │  via ODF; rec-engine &  │
   │                     │   │  order-placement at     │
   │  ODF: replicating   │──▶│  replicas: 0 until      │
   │  PVCs to DR cluster │   │  failover triggered)    │
   └─────────────────────┘   └─────────────────────────┘
```

### ACM Application Subscription (Example)

Create a `Channel` pointing to this repository and a `Subscription` that deploys the Helm chart to your primary cluster:

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: globex-dr-channel
  namespace: globex-dr-channel
spec:
  type: HelmRepo
  pathname: https://github.com/anatsheh84/globex-dr
---
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  name: globex-dr-subscription
  namespace: globex
  annotations:
    apps.open-cluster-management.io/git-path: "."
    apps.open-cluster-management.io/git-branch: main
spec:
  channel: globex-dr-channel/globex-dr-channel
  placement:
    placementRef:
      name: globex-placement
      kind: PlacementRule
  packageOverrides:
  - packageName: globex
    packageOverrides:
    - path: spec.values
      value: |
        globex:
          namespace: globex
```

### Placement Rules

Define separate `PlacementRule` resources to control which clusters receive the application. On normal operation, only the primary cluster is selected. During failover, the hub operator updates (or ACM automatically updates via a DR policy) the placement to point to the DR cluster:

```yaml
# Normal operation — primary cluster only
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: globex-placement
  namespace: globex
spec:
  clusterSelector:
    matchLabels:
      site: primary

---
# After failover — DR cluster promoted
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: globex-placement
  namespace: globex
spec:
  clusterSelector:
    matchLabels:
      site: dr
```

---

## Templates Reference

| Prefix | Kind | Count | Description |
|---|---|---|---|
| `dep-*` | Deployment | 9 | All application and database deployments |
| `svc-*` | Service | 9 | ClusterIP services for all components |
| `secret-*` | Secret | 6 | Database credentials and service configs |
| `sa-*` | ServiceAccount | 6 | Per-service accounts with least-privilege scope |
| `pvc-*` | PersistentVolumeClaim | 2 | Persistent storage for both databases (ODF-replicated in DR) |
| `crb-*` | ClusterRoleBinding | 2 | `anyuid` SCC grants for database pods (deployed by ACM) |
| `rt-*` | Route | 3 | OpenShift Routes for `globex-ui`, `catalog`, `activity-tracking-simulator` |

---

## Manual Installation (Without ACM)

For testing or standalone deployments without ACM:

### Prerequisites

- OpenShift cluster with cluster-admin access
- Helm 3 installed
- For DR scenarios: ODF installed with volume replication configured between primary and DR clusters

### Deploy

```bash
# Deploy to the default 'globex' namespace
helm install globex . -n globex --create-namespace

# Deploy to a custom namespace
helm install globex . -n my-namespace --create-namespace \
  --set globex.namespace=my-namespace
```

### Verify

```bash
oc get pods -n globex
oc get routes -n globex
oc get pvc -n globex
```

### Access the UI

```bash
oc get route globex-ui -n globex -o jsonpath='{.spec.host}'
```

---

## Configuration

| Parameter | Default | Description |
|---|---|---|
| `globex.namespace` | `globex` | Target namespace for all resources |

---

## DR Failover Workflow

### With ACM (Recommended)

1. A failover is triggered — either manually by an operator on the ACM hub, or automatically via an ACM DR policy reacting to a cluster health event
2. ACM updates the `PlacementRule` (or `Placement`) to target the DR cluster
3. ACM reconciles the Helm chart on the DR cluster, which already has the database PVCs populated via ODF replication
4. The DR cluster's `recommendation-engine` and `order-placement` deployments are patched to `replicas: 1` as part of the failover runbook (via an ACM policy or automation template)
5. OpenShift Routes on the DR cluster become active; DNS is updated to point traffic to the DR cluster's ingress

### Without ACM (Manual)

```bash
# On the DR cluster, after ODF replication has been fenced/promoted:
helm install globex . -n globex --create-namespace

# Then scale up the standby services
oc scale deployment recommendation-engine --replicas=1 -n globex
oc scale deployment order-placement --replicas=1 -n globex
```

---

## Uninstall

```bash
helm uninstall globex -n globex
oc delete namespace globex
```

---

## Relationship to `anatsheh84/Globex`

| Feature | [Globex](https://github.com/anatsheh84/Globex) | [globex-dr](https://github.com/anatsheh84/globex-dr) |
|---|---|---|
| Format | Plain Kubernetes manifests | Helm chart |
| Deployment method | `oc apply -k .` | `helm install` or **ACM Subscription** |
| Database storage | No PVCs (ephemeral) | PVCs (5Gi each, ODF-replicated) |
| Namespace | Hardcoded | Configurable via `values.yaml` |
| SCC handling | Not included | `anyuid` CRBs included |
| DR standby mode | All replicas active | `recommendation-engine` and `order-placement` start at `replicas: 0` |
| ACM integration | Not applicable | Designed for ACM Subscription + PlacementRule |
