# README — Installing Keycloak 25+ (Quarkus) on Kubernetes with Helm

**Last updated:** October 2025

> This document provides a detailed technical guide (internal onboarding style) for installing and running **Keycloak 25+ (Quarkus)** in a local Kubernetes environment (Docker Desktop), using the **codecentric/keycloakx** Helm Chart. It includes step‑by‑step instructions, technical rationale for chart selection (Codecentric vs Bitnami), comparative `values.yaml` notes, and migration considerations for production environments.

---

## Table of Contents

1. Objective
2. Scope and Assumptions
3. Prerequisites
4. Why We Chose the Codecentric Chart (Comparative Summary)
5. Technical Comparison — Codecentric vs Bitnami (Key Parameters)
6. Step‑by‑Step Tutorial (Original Tutorial Fully Preserved)

    * Namespace
    * Add Helm Repository
    * Create `values.yaml`
    * Install with Helm
    * Monitoring and Troubleshooting
    * Access via Browser (Port‑forward)
    * Cleanup
7. Advanced Tips and Production Notes
8. Migration: WildFly → Quarkus — Key Points
9. References and Useful Links

---

## 1. Objective

Provide a comprehensive technical guide for installing and operating **Keycloak 25+ (Quarkus)** on Kubernetes (local development with Docker Desktop), specifically addressing common networking and clustering issues (e.g., JGroups `jdbc‑ping` errors) and explaining the technical reasoning behind the chosen chart.

## 2. Scope and Assumptions

* Target environment: **Docker Desktop** with Kubernetes enabled (single node).
* Intended use: **Local development**, manual testing, and local CI integration.
* This README keeps **all original tutorial content** while adding context, comparison, and rationale.

## 3. Prerequisites

Ensure the following tools are installed and configured:

* **Docker Desktop** (Kubernetes enabled).

    * `Settings → Kubernetes → Enable Kubernetes`
* **kubectl** (configured for Docker Desktop cluster).

    * `kubectl version --client`
* **helm** (installed).

    * `helm version`

---

## 4. Why We Chose the Codecentric Chart (Summary)

* Since Keycloak v17, the project migrated from WildFly to **Quarkus** runtime (Keycloak.X). This impacts configuration, clustering behavior, and startup commands (`start‑dev`, `kc.sh`).
* The repository `keycloak/keycloak-helm` that could serve as an “official chart” returns 404 as of October 2025 — meaning no official, stable Helm chart exists.
* **Codecentric/keycloakx**: designed specifically for Quarkus/Keycloak.X with values optimized for version 25+ — ideal for dev and prototyping.
* **Bitnami/keycloak**: robust, widely used in production, but verify it uses the new Quarkus image (older versions still reference legacy WildFly images).

**Summary Decision:** For local development and to avoid Quarkus compatibility issues, use **codecentric/keycloakx**. For production/HA, evaluate Bitnami (ensuring Quarkus image) or the Operator‑based deployment.

---

## 5. Technical Comparison — Codecentric vs Bitnami (Key Parameters)

| Topic                     | Codecentric (`keycloakx`)                                    | Bitnami (`keycloak`)                                             |
| ------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| Focus                     | Quarkus‑first; built for Keycloak.X                          | Historically general; may still reference legacy images          |
| Cluster                   | `cluster.enabled` (boolean) — disable for dev (`false`)      | Distributed parameters (JGroups args, `JAVA_OPTS_APPEND`)        |
| Admin credentials         | `extraEnv` with `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | `auth.adminUser` / `auth.adminPassword`                          |
| Startup command           | Supports `/opt/keycloak/bin/kc.sh start-dev`                 | May require custom `command`/`args`; check Quarkus compatibility |
| Persistence / DB          | Simple H2 default; `persistence.enabled`                     | Integrated PostgreSQL (can enable `postgresql.enabled`)          |
| Documentation for Quarkus | Excellent — built for Keycloak.X                             | Historically good, but Quarkus behavior varies per release       |

**Practical recommendation:** For dev setups with Keycloak 25+, use **Codecentric keycloakx**. For production, verify Bitnami or Operator compatibility.

---

## 6. Step‑by‑Step Tutorial (Original Content Preserved)

> The following tutorial retains the original content, now formatted and annotated for clarity. Follow these exact steps to create a working single‑node Keycloak environment on Docker Desktop.

### Create Namespace

```bash
kubectl create namespace keycloak-ns
```

> *Tip:* A dedicated namespace helps isolate and clean up resources easily.

### Step 1 — Add Helm Repository

```bash
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update
```

### Step 2 — Create the `values.yaml` File

**Objectives:**

* Use Keycloak 25+ (Quarkus runtime)
* Disable clustering (prevent JGroups `jdbc‑ping` error)
* Force `KC_CACHE_STACK=tcp` to avoid discovery issues in Docker Desktop
* Use `start‑dev` for a fast, lightweight startup

**Sample `values.yaml`:**

```yaml
# ============================================================================
# values.yaml - Configuration for Keycloak 25+ (Dev / Single Node Mode)
# ============================================================================

image:
  tag: "25.0"

cluster:
  enabled: false

extraEnv: |
  - name: KEYCLOAK_ADMIN
    value: "admin"
  - name: KEYCLOAK_ADMIN_PASSWORD
    value: "ChangeThisPassword123" # ⚠️ CHANGE THIS PASSWORD!

  - name: KC_CACHE_STACK
    value: "tcp"

  - name: KC_HOSTNAME_STRICT
    value: "false"
  - name: KC_HTTP_ENABLED
    value: "true"

command:
  - "/opt/keycloak/bin/kc.sh"
  - "start-dev"
  - "--http-port=8080"

service:
  type: NodePort
  port: 8080
  targetPort: 8080

persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 5Gi

ingress:
  enabled: false
```

### Step 3 — Install / Upgrade Keycloak with Helm

```bash
helm upgrade --install keycloak-dev codecentric/keycloakx -f values.yaml --namespace keycloak-ns
```

### Step 4 — Monitor and Troubleshoot

Check pod status:

```bash
kubectl get pods -n keycloak-ns
```

Check logs:

```bash
kubectl logs -n keycloak-ns keycloak-dev-keycloakx-0
```

Expected message:

```
...Keycloak 25.0.x on JVM (...) started in X.Xs...
```

If startup fails:

* Use `kubectl describe pod <pod>` for event details.
* Check resource limits and volume mounts.
* Confirm Quarkus image compatibility.

### Step 5 — Access via Browser (Port‑forward)

```bash
kubectl port-forward pod/keycloak-dev-keycloakx-0 8888:8080 -n keycloak-ns
```

Open `http://localhost:8888/auth/admin/` and log in with:

* **Username:** `admin`
* **Password:** `ChangeThisPassword123`

> Keep the port‑forward terminal open while accessing the UI.

### Step 6 — Cleanup (Optional)

```bash
helm uninstall keycloak-dev -n keycloak-ns
kubectl delete namespace keycloak-ns
```

---

## 7. Advanced Tips and Production Notes

* **Database:** For production, **do not** use H2. Use PostgreSQL (external or chart‑based). Configure credentials and backups.
* **Clustering / HA:** Set `cluster.enabled=true` and configure appropriate cache stack (e.g., `jdbc-ping`) with shared DB.
* **Operator:** Consider using the **Keycloak Operator** for production‑grade deployments.
* **Resources:** Adjust CPU/memory requests and limits for stability.
* **Probes:** Configure readiness and liveness probes properly to ensure uptime.

---

## 8. Migration WildFly → Quarkus — Key Points

* **Custom providers and themes** may need recompilation or adaptation for Quarkus.
* **Paths and endpoints:** some paths changed (e.g., `/auth` prefix removed by default).
* **Startup scripts:** WildFly’s `standalone.sh` replaced by `kc.sh start-dev`.
* **Testing:** Always validate tokens, clients, and SPI scripts in staging before production migration.

---

## 9. References and Useful Links (October 2025)

* Codecentric Helm Charts: [https://codecentric.github.io/helm-charts](https://codecentric.github.io/helm-charts)
* Bitnami Keycloak Chart (Artifact Hub): [https://artifacthub.io/packages/helm/bitnami/keycloak](https://artifacthub.io/packages/helm/bitnami/keycloak)
* Keycloak Migration to Quarkus: [https://www.keycloak.org/migration/migrating-to-quarkus](https://www.keycloak.org/migration/migrating-to-quarkus)
* GitHub Issues and Discussions: search for `keycloakx helm` or `bitnami keycloak quarkus`

---

### Final Notes

This README is designed for internal onboarding and DevOps use. Next steps you may request:

* Generate a `values-bitnami.yaml` aligned with the Bitnami chart.
* Export this README as a downloadable `.md` file for repository documentation.
