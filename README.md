# globex-dr — Globex Helm Chart for Disaster Recovery

A **Helm chart** packaging the full Globex retail microservices application, purpose-built for **Disaster Recovery (DR)** demonstrations on OpenShift. This is the Helm-based companion to the plain-manifest [Globex](https://github.com/anatsheh84/Globex) repository, with key DR-specific differences: stateful databases use **persistent PVCs**, database pods require the `anyuid` SCC via ClusterRoleBindings, and certain services start with `replicas: 0` to represent a standby DR state.

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
       ┌────────┐ ┌────────┐ ┌────────────────────┐ ┌────────────────┐
       │catalog │ │inventory│ │recommendation-engine│ │order-placement │
       └───┬────┘ └────┬───┘ └────────────────────┘ └────────────────┘
           │           │
    ┌──────┴──┐  ┌─────┴──────┐
    │catalog- │  │inventory-  │
    │database │  │database    │
    │(PostgreSQL) (PostgreSQL)│
    └─────────┘  └────────────┘
           ▲
    ┌──────┴─────────────┐
    │activity-tracking   │  (+ activity-tracking-simulator)
    └────────────────────┘
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

> **DR Note:** `recommendation-engine` and `order-placement` are deployed with `replicas: 0`. This represents services that are in a **passive standby state** on the DR cluster and are scaled up only after a failover event.

---

## DR-Specific Design Decisions

This chart differs from the plain-manifest Globex repo in several important ways:

**1. Persistent Storage for Databases**
Both PostgreSQL databases use PVCs instead of ephemeral storage, making them candidates for **ODF (Ceph RBD/CephFS) volume replication** in a DR setup:

| PVC | Size | Access Mode |
|---|---|---|
| `catalog-database` | 5Gi | ReadWriteOnce |
| `inventory-database` | 5Gi | ReadWriteOnce |

**2. `anyuid` SCC ClusterRoleBindings**
The database ServiceAccounts (`catalog-app`, `inventory-app`) are granted the `anyuid` Security Context Constraint via ClusterRoleBindings. This is required because the database containers include an `initContainer` (`alpine:3.16.0`) that changes filesystem permissions — an operation that requires running as arbitrary UIDs.

**3. Helm Templating with Namespace Injection**
The namespace is parameterised throughout via `{{ $.Values.globex.namespace }}`, making it straightforward to deploy to different namespaces on primary vs. DR clusters.

**4. Health Probes on Quarkus Services**
`recommendation-engine` and `order-placement` include both liveness (`/q/health/live`) and readiness (`/q/health/ready`) probes with appropriate delays, ensuring proper readiness signalling on DR cluster startup.

---

## Templates Reference

| Prefix | Kind | Count | Description |
|---|---|---|---|
| `dep-*` | Deployment | 9 | All application and database deployments |
| `svc-*` | Service | 9 | ClusterIP services for all components |
| `secret-*` | Secret | 6 | Database credentials and service configs |
| `sa-*` | ServiceAccount | 6 | Per-service accounts with least-privilege scope |
| `pvc-*` | PersistentVolumeClaim | 2 | Persistent storage for both databases |
| `crb-*` | ClusterRoleBinding | 2 | `anyuid` SCC grants for database pods |
| `rt-*` | Route | 3 | OpenShift Routes for `globex-ui`, `catalog`, `activity-tracking-simulator` |

---

## Installation

### Prerequisites

- OpenShift cluster with cluster-admin access
- Helm 3 installed
- For DR scenarios: ODF (OpenShift Data Foundation) installed with volume replication configured between primary and DR clusters

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

After the DR cluster receives replicated data and you are ready to activate the standby services:

```bash
# Scale up standby services on the DR cluster
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
| Database storage | No PVCs (ephemeral) | PVCs (5Gi each) |
| Namespace | Hardcoded | Configurable via `values.yaml` |
| SCC handling | Not included | `anyuid` CRBs included |
| DR standby mode | All replicas active | `recommendation-engine` and `order-placement` start at `replicas: 0` |
| Deployment method | `oc apply -k .` | `helm install` |
