# Kubernetes Deployment Plan: AI Native Task Manager

> **Course:** AI-400 | **Class:** 22 | **Date:** 2026-04-06
> **Author:** Irfan | **Version:** 1.0 | **Status:** Final

---

## System Overview

The AI Native Task Manager is a cloud-native application that allows users to create, assign, and track tasks with the help of an AI agent that plans and prioritizes work. It consists of four primary components served through a Kubernetes cluster.

```
           Internet
               │
        ┌──────▼──────┐
        │   Ingress   │ (nginx-ingress)
        └──┬──────┬───┘
           │      │
    /      │      │ /api/*
    ▼      │      ▼
┌──────┐   │   ┌─────────┐
│  UI  │   │   │ Backend │
│ nginx│   │   │   API   │
└──────┘   │   └──┬──┬───┘
           │      │  │
           │  ┌───┘  └─────────────┐
           │  ▼                    ▼
           │ ┌──────────┐    ┌──────────┐
           │ │Task Agent│    │ Notify   │
           │ │ (LLM)    │    │ Service  │
           │ └──────────┘    └──────────┘
           │        │
           │  ┌─────┴──────┐
           │  │  Postgres  │  Redis
           │  │ StatefulSet│  Cache
           └──└────────────┘
```

---

## Section 1: Pods, Deployments & StatefulSets

| Component | Controller | Replicas (Prod) | Container Image | Container Port | Notes |
|---|---|---|---|---|---|
| ui-interface | Deployment | 2 | `nginx:alpine` | 80 | Serves pre-built React SPA; stateless; HPA eligible |
| backend-api | Deployment | 3 | `ai-taskmanager/api:v1.0` | 8080 | Stateless REST API; horizontal scaling safe |
| task-agent | Deployment | 2 | `ai-taskmanager/agent:v1.0` | 9090 | LLM inference calls; stateless worker |
| notification-svc | Deployment | 1 | `ai-taskmanager/notify:v1.0` | 8081 | Low-traffic event handler; single replica sufficient |
| postgres-db | **StatefulSet** | 1 | `postgres:15` | 5432 | Needs stable network identity + PVC; cannot be Deployment |
| redis-cache | **StatefulSet** | 1 | `redis:7-alpine` | 6379 | In-memory store; stable hostname required by clients |

### Horizontal Pod Autoscaler (HPA) Rules

| Component | Min Replicas | Max Replicas | Scale Trigger | Metric |
|---|---|---|---|---|
| ui-interface | 2 | 6 | CPU > 70% | `cpu` |
| backend-api | 3 | 10 | CPU > 70% | `cpu` |
| task-agent | 2 | 8 | CPU > 60% | `cpu` |
| notification-svc | 1 | 3 | Queue depth > 100 | custom |

### PersistentVolumeClaims (StatefulSets only)

| Component | PVC Name | Storage Class | Capacity | Access Mode |
|---|---|---|---|---|
| postgres-db | `postgres-data` | `standard` | 20Gi | ReadWriteOnce |
| redis-cache | `redis-data` | `standard` | 5Gi | ReadWriteOnce |

---

## Section 2: Services and Service Types

| Component | Service Name | Service Type | Cluster Port | Target Port | External Port | Reason for Type |
|---|---|---|---|---|---|---|
| ui-interface | `ui-service` | LoadBalancer | 80 | 80 | 443 (TLS at LB) | Public-facing SPA needs external IP |
| backend-api | `api-service` | ClusterIP | 8080 | 8080 | — | Internal only; routed via Ingress at `/api/*` |
| task-agent | `agent-service` | ClusterIP | 9090 | 9090 | — | Called only by backend-api; no external exposure |
| notification-svc | `notify-service` | ClusterIP | 8081 | 8081 | — | Triggered by backend-api; internal only |
| postgres-db | `postgres-service` | ClusterIP | 5432 | 5432 | — | Database never exposed outside cluster |
| redis-cache | `redis-service` | ClusterIP | 6379 | 6379 | — | Cache never exposed outside cluster |

### Ingress Rules

| Host / Path | Backend Service | Port | TLS | Notes |
|---|---|---|---|---|
| `taskmanager.example.com/` | `ui-service` | 80 | Yes | SPA catch-all route |
| `taskmanager.example.com/api/*` | `api-service` | 8080 | Yes | REST API routing |
| `taskmanager.example.com/ws` | `api-service` | 8080 | Yes | WebSocket upgrade for real-time |

---

## Section 3: Resource Requests and Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Notes |
|---|---|---|---|---|---|
| ui-interface | `100m` | `200m` | `128Mi` | `256Mi` | Static file serving; minimal compute |
| backend-api | `250m` | `500m` | `256Mi` | `512Mi` | Business logic + DB queries |
| task-agent | `500m` | `2000m` | `512Mi` | `2Gi` | LLM context buffering; high burst |
| notification-svc | `100m` | `200m` | `128Mi` | `256Mi` | Lightweight SMTP/webhook sender |
| postgres-db | `500m` | `1000m` | `512Mi` | `1Gi` | DB engine; stable allocation avoids OOM |
| redis-cache | `100m` | `200m` | `256Mi` | `512Mi` | In-memory; cap strictly to prevent bloat |

### LimitRange (Namespace Default)

| Scope | Default CPU Request | Default CPU Limit | Default Mem Request | Default Mem Limit |
|---|---|---|---|---|
| Container | `100m` | `500m` | `128Mi` | `512Mi` |

---

## Section 4: ConfigMaps

| ConfigMap Name | Consumed By | Key Data Fields | Mount Method |
|---|---|---|---|
| `api-config` | backend-api | `DB_HOST`, `DB_PORT`, `REDIS_HOST`, `REDIS_PORT`, `LOG_LEVEL`, `AGENT_URL`, `NOTIFY_URL` | `envFrom` |
| `agent-config` | task-agent | `LLM_PROVIDER`, `MODEL_NAME`, `MAX_TOKENS`, `TEMPERATURE`, `API_TIMEOUT_SECONDS` | `envFrom` |
| `notify-config` | notification-svc | `SMTP_HOST`, `SMTP_PORT`, `FROM_EMAIL`, `RETRY_COUNT`, `RETRY_DELAY_MS` | `envFrom` |
| `ui-runtime-config` | ui-interface | `API_BASE_URL`, `WS_URL`, `APP_VERSION`, `FEATURE_FLAGS` | Volume mount as `/app/config.js` |
| `postgres-config` | postgres-db | `POSTGRES_DB`, `POSTGRES_USER` (non-secret fields only) | `envFrom` |

---

## Section 5: Secrets Management and Expiry Handling

| Secret Name | Consumed By | Keys Stored | Storage Backend | Rotation Policy | Expiry Handling |
|---|---|---|---|---|---|
| `db-credentials` | backend-api, postgres-db | `POSTGRES_PASSWORD`, `DATABASE_URL` | Kubernetes Secret (base64) | 90 days | CronJob triggers rolling restart post-rotation |
| `llm-api-key` | task-agent | `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` | External Secrets Operator → Vault | 30 days | ESO auto-syncs; pod restarts automatically |
| `jwt-secret` | backend-api | `JWT_SECRET`, `JWT_REFRESH_SECRET` | Kubernetes Secret | 180 days | Dual-key: old key kept 24h for token drain |
| `smtp-credentials` | notification-svc | `SMTP_USERNAME`, `SMTP_PASSWORD` | External Secrets Operator → Vault | 90 days | CronJob alerts on Slack 7 days before expiry |
| `redis-auth` | backend-api, redis-cache | `REDIS_PASSWORD` | Kubernetes Secret | 90 days | Rolling restart of all consumers after rotation |

### Secret Rotation Architecture

| Tool | Role | Trigger |
|---|---|---|
| External Secrets Operator (ESO) | Syncs secrets from Vault / AWS Secrets Manager into K8s | Polls every 1 hour |
| HashiCorp Vault | Source of truth for all sensitive credentials | Manual or scheduled rotation |
| `secret-expiry-checker` CronJob | Checks TTL of all tracked secrets | Runs nightly at 02:00 UTC |
| Reloader (Stakater) | Watches Secrets for changes and triggers rolling restarts | Event-driven |

---

## Section 6: Namespaces

| Namespace | Components Deployed | Purpose / Isolation |
|---|---|---|
| `ai-taskmanager-prod` | All production components | Full production isolation; strict NetworkPolicy |
| `ai-taskmanager-dev` | All components (1 replica each) | Dev and testing; relaxed policies |
| `ai-taskmanager-staging` | All components (2 replicas) | Pre-production validation |
| `ai-taskmanager-monitoring` | Prometheus, Grafana, Alertmanager | Observability stack isolated from app |
| `ai-taskmanager-secrets` | ESO SecretStore, Vault Agent | Secrets management isolated |

### ResourceQuota Per Namespace

| Namespace | CPU Limit | Memory Limit | Max Pods | Max PVCs |
|---|---|---|---|---|
| `prod` | 16 cores | 32Gi | 50 | 10 |
| `dev` | 4 cores | 8Gi | 20 | 5 |
| `staging` | 8 cores | 16Gi | 30 | 8 |
| `monitoring` | 4 cores | 8Gi | 15 | 5 |
| `secrets` | 2 cores | 4Gi | 10 | 2 |

---

## Section 7: RBAC Roles and RoleBindings

| Role Name | Role Kind | Namespace | Allowed Verbs | Resources | Bound To (Subject) | Subject Kind |
|---|---|---|---|---|---|---|
| `backend-api-role` | Role | prod | get, list, watch | pods, configmaps, secrets | `backend-api-sa` | ServiceAccount |
| `task-agent-role` | Role | prod | get, list | configmaps, secrets | `task-agent-sa` | ServiceAccount |
| `notify-role` | Role | prod | get | configmaps, secrets | `notify-sa` | ServiceAccount |
| `dev-full-role` | Role | dev | * (all) | * (all) | `ai400-class22-devs` | Group |
| `cicd-deployer-role` | Role | prod | get, list, update, patch | deployments, replicasets, pods | `cicd-sa` | ServiceAccount |
| `monitoring-reader` | ClusterRole | cluster-wide | get, list, watch | pods, nodes, endpoints, services | `prometheus-sa` | ServiceAccount |

### ServiceAccounts

| ServiceAccount | Namespace | Used By | Automount Token |
|---|---|---|---|
| `backend-api-sa` | prod | backend-api pods | false (manual mount) |
| `task-agent-sa` | prod | task-agent pods | false |
| `notify-sa` | prod | notification-svc pods | false |
| `cicd-sa` | prod | CI/CD pipeline | false |
| `prometheus-sa` | monitoring | Prometheus | false |

---

## Section 8: Inter-Service Communication

| Source | Destination | Protocol | Method | Port | Authentication | Notes |
|---|---|---|---|---|---|---|
| User Browser | ui-interface | HTTPS | HTTP/1.1 | 443 | None (public) | TLS terminated at LoadBalancer |
| ui-interface | backend-api | HTTPS | REST (via Ingress) | 443 → 8080 | JWT Bearer token | Token issued at login |
| backend-api | task-agent | HTTP | REST POST | 9090 | Internal service token | Synchronous task planning calls |
| backend-api | postgres-db | TCP | PostgreSQL wire protocol | 5432 | DB credentials (Secret) | Connection pool via PgBouncer |
| backend-api | redis-cache | TCP | Redis protocol | 6379 | Redis AUTH password | Session store + task queue |
| backend-api | notification-svc | HTTP | REST (async) | 8081 | Internal service token | Fire-and-forget; non-blocking |
| notification-svc | External SMTP | TCP | SMTP/TLS | 587 | SMTP credentials (Secret) | Email notifications |
| task-agent | External LLM API | HTTPS | REST | 443 | API key (Secret) | OpenAI / Anthropic API |

### NetworkPolicy Rules

| Policy Name | Applied To (Pod Selector) | Ingress Allowed From | Egress Allowed To |
|---|---|---|---|
| `default-deny-all` | all pods in namespace | none | none (baseline block) |
| `allow-ingress-to-ui` | `app: ui-interface` | Ingress controller namespace | api-service only |
| `allow-ui-to-api` | `app: backend-api` | ui-interface pods, Ingress | postgres, redis, task-agent, notify |
| `allow-api-to-agent` | `app: task-agent` | backend-api pods | External (LLM API) via egress |
| `allow-api-to-db` | `app: postgres-db` | backend-api pods | none |
| `allow-api-to-redis` | `app: redis-cache` | backend-api pods | none |
| `allow-api-to-notify` | `app: notification-svc` | backend-api pods | External SMTP via egress |

---

## Deployment Order

| Step | Component | Depends On | Controller | Notes |
|---|---|---|---|---|
| 1 | Namespaces | — | kubectl apply | Create all namespaces first |
| 2 | RBAC | Namespaces | kubectl apply | ServiceAccounts, Roles, Bindings |
| 3 | ConfigMaps | Namespaces | kubectl apply | All config before any pods |
| 4 | Secrets / ESO | Namespaces, Vault | kubectl apply | Secrets must exist before pods reference them |
| 5 | postgres-db | Secrets, ConfigMaps | StatefulSet | Wait for `Running` + readiness probe |
| 6 | redis-cache | Secrets | StatefulSet | Wait for `Running` |
| 7 | backend-api | postgres, redis, ConfigMaps | Deployment | Health check: `GET /healthz` |
| 8 | task-agent | backend-api, Secrets | Deployment | Health check: `GET /healthz` |
| 9 | notification-svc | backend-api, Secrets | Deployment | Health check: `GET /healthz` |
| 10 | ui-interface | backend-api (Ingress route) | Deployment | Serve only after API is healthy |
| 11 | Ingress | ui-service, api-service | Ingress resource | Apply after all services are Ready |
| 12 | HPA | All Deployments | HPA resources | Apply last; requires metrics-server |
