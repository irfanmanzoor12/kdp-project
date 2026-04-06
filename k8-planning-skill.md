---
name: k8-planner
description: Generate a comprehensive Kubernetes deployment plan in table format for any application. Covers Pods/Deployments/StatefulSets, Services, Resource Requests/Limits, ConfigMaps, Secrets, Namespaces, RBAC, and Inter-Service Communication.
version: 1.0.0
author: AI-400 Class 22 — Irfan
created: 2026-04-06
triggers:
  - "plan k8s deployment"
  - "kubernetes plan for"
  - "k8 deploy plan"
  - "generate k8 plan"
  - "k8s architecture plan"
  - "/k8-planner"
---

# K8 Planning Skill

## Skill Overview

This skill generates a complete, structured Kubernetes deployment plan for any application. It produces table-format documentation covering all critical K8s concerns — from pod controllers and service exposure to RBAC, secrets rotation, and inter-service networking.

| Property | Value |
|---|---|
| **Output Format** | Markdown with tables for every section |
| **Sections Produced** | 8 required (+ optional Security sections) |
| **Security Depth** | Adjustable via `--security-level` flag |
| **Use Cases** | Project planning, architecture review, onboarding docs, course assignments |
| **What It Does NOT Do** | Generate YAML manifests (use `--export=yaml` flag for starter manifests) |

---

## Trigger Conditions

| Trigger Pattern | Invocation Mode | Behavior |
|---|---|---|
| `/k8-planner` | Interactive | Asks information-gathering questions, then generates full plan |
| `/k8-planner [app-name]` | Quick (partial) | Uses app name, asks remaining questions |
| `/k8-planner [app-name] [components...]` | Quick (full) | Generates immediately with smart defaults |
| `/k8-planner --security-level=high` | Security-enhanced | Expands RBAC, Secrets, and adds Threat Model + Hardening Checklist sections |
| `/k8-planner --env=dev` | Environment-scoped | Adjusts replicas and resource sizes for dev environment |
| `/k8-planner --export=yaml` | With YAML stubs | Also generates starter YAML manifests for each section |
| User types: "plan k8s for my app" | Natural language | Treats as interactive mode |

---

## Information Gathering

When invoked interactively, the skill asks the following questions before generating the plan:

| # | Question | Purpose | Example Answer |
|---|---|---|---|
| 1 | What is the application name? | Names all K8s resources (prefixed consistently) | `"ai-task-manager"` |
| 2 | List the components / services (comma-separated) | Drives all 8 sections | `"frontend, backend-api, task-agent, notification-svc"` |
| 3 | Which components need persistent storage? | Determines StatefulSet vs Deployment | `"postgres, redis"` |
| 4 | What is the deployment environment? | Adjusts replicas and resource sizes | `"production"` / `"dev"` / `"staging"` |
| 5 | What external APIs does the app call? | Drives Secrets section (API keys) | `"OpenAI API, Stripe, SendGrid"` |
| 6 | Which components should be publicly accessible? | Determines Service types (LoadBalancer vs ClusterIP) | `"frontend"` |
| 7 | What databases or caches are needed? | StatefulSet planning, ConfigMap DB fields | `"postgres:15, redis:7-alpine"` |
| 8 | Security sensitivity level? (low / medium / high) | Expands RBAC + Secrets depth; adds threat model if high | `"high"` |
| 9 | Expected traffic level? (low / medium / high) | Resource requests/limits sizing and HPA rules | `"medium (100 req/s)"` |
| 10 | Any compliance requirements? (SOC2, HIPAA, PCI, etc.) | Adds compliance notes to Secrets and RBAC sections | `"SOC2"` |

---

## Generation Prompt Template

When invoked, the skill sends the following prompt to Claude (substituting gathered values):

```
You are a Kubernetes architect. Generate a complete Kubernetes deployment plan for the following application.
Output ALL sections as Markdown tables. Do not use bullet lists — use tables for everything.

## Application Details
- App Name: {APP_NAME}
- Components: {COMPONENTS}
- Persistent Storage Needed: {STATEFUL_COMPONENTS}
- Environment: {ENVIRONMENT}
- External APIs: {EXTERNAL_APIS}
- Public Components: {PUBLIC_COMPONENTS}
- Databases/Caches: {DATABASES}
- Security Level: {SECURITY_LEVEL}
- Traffic Level: {TRAFFIC_LEVEL}

## Required Sections (produce all 8, in order)

### Section 1: Pods, Deployments & StatefulSets
Columns: Component | Controller (Deployment/StatefulSet) | Replicas | Container Image | Port | Notes
Rules:
- Use StatefulSet for any component in {STATEFUL_COMPONENTS}
- Use Deployment for all others (stateless assumed)
- Include HPA rules sub-table: Component | Min | Max | Trigger | Metric
- Include PVC sub-table for StatefulSets: Component | PVC Name | Storage Class | Capacity | Access Mode

### Section 2: Services and Service Types
Columns: Component | Service Name | Service Type | Cluster Port | Target Port | External Port | Reason for Type
Rules:
- Public components ({PUBLIC_COMPONENTS}) → LoadBalancer
- All others → ClusterIP
- Add Ingress rules sub-table if applicable: Path | Backend Service | Port | TLS

### Section 3: Resource Requests and Limits
Columns: Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Notes
Rules:
- Low traffic: halve the medium values
- Medium: frontend 100m/200m 128Mi/256Mi; APIs 250m/500m 256Mi/512Mi; AI workers 500m/2000m 512Mi/2Gi; DBs 500m/1000m 512Mi/1Gi
- High traffic: double the medium values
- Include LimitRange default sub-table

### Section 4: ConfigMaps
Columns: ConfigMap Name | Consumed By | Key Data Fields | Mount Method
Rules:
- Never put passwords, tokens, or keys in ConfigMaps
- Use envFrom for env vars; use volume mounts for files (nginx.conf, config.js, etc.)
- One ConfigMap per component is the standard pattern

### Section 5: Secrets Management and Expiry Handling
Columns: Secret Name | Consumed By | Keys Stored | Storage Backend | Rotation Policy | Expiry Handling
Rules:
- If security level is "high": use External Secrets Operator + HashiCorp Vault as backend
- If security level is "medium": use Kubernetes Secrets with Reloader
- If security level is "low": use Kubernetes Secrets
- Always include a Secret Rotation Architecture sub-table: Tool | Role | Trigger
- NEVER suggest storing secrets as environment variables if security level is medium or high

### Section 6: Namespaces
Columns: Namespace | Components Deployed | Purpose / Isolation
Rules:
- Always create at minimum: prod, dev, monitoring namespaces
- If security level is "high": add a secrets namespace and isolate high-risk components
- Include ResourceQuota sub-table: Namespace | CPU Limit | Memory Limit | Max Pods | Pod Security Standard

### Section 7: RBAC Roles and RoleBindings
Columns: Role Name | Role Kind | Namespace | Allowed Verbs | Resources | Bound To | Subject Kind
Rules:
- Principle of least privilege: each ServiceAccount gets only what it needs
- No ServiceAccount should have wildcard (*) verbs in production
- Create one ServiceAccount per component (never use the default SA)
- Add ServiceAccounts summary sub-table: SA Name | Namespace | automountServiceAccountToken
- If security level is "high": add a "What It CANNOT Do" column

### Section 8: Inter-Service Communication
Columns: Source | Destination | Protocol | Port | Authentication | Security Notes
Rules:
- Add NetworkPolicy sub-table: Policy Name | Namespace | Applied To | Ingress Allowed From | Egress Allowed To
- Always include a "default-deny-all" NetworkPolicy as first row
- If security level is "high": add mTLS requirement for all internal service calls

## Conditional Sections (include only if security level is "high")

### Section 9: Security Threat Model
Columns: Threat | Attack Vector | Mitigation | Kubernetes Control

### Section 10: Security Hardening Checklist
Columns: # | Control | Implementation | K8s Mechanism | Status (checkbox [ ])
Always include: PodSecurityStandard, no-root, readOnlyRootFilesystem, drop capabilities, seccomp, no env var secrets, default-deny NetworkPolicy, image signing, no automounted SA tokens.

## Deployment Order Section
Always end with a deployment order table:
Columns: Step | Component | Namespace | Depends On | Notes
Rules:
- Namespaces and RBAC always first
- NetworkPolicy deny-all before any pods
- Databases before APIs
- APIs before frontends
- Ingress/Gateway always last

## Formatting Rules
- Every section starts with the section number and title as H3
- Every section has a one-sentence rationale before the table
- Use backtick formatting for all resource names, image names, and k8s values
- Use bold for StatefulSet in the controller column to visually distinguish them
```

---

## Output Sections Reference

| # | Section | Required Columns | Sub-Tables |
|---|---|---|---|
| 1 | Pods, Deployments & StatefulSets | Component, Controller, Replicas, Image, Port, Notes | HPA Rules; PVCs |
| 2 | Services and Types | Component, Service Name, Type, Cluster Port, Target Port, External, Reason | Ingress Rules |
| 3 | Resource Requests & Limits | Component, CPU Req, CPU Limit, Mem Req, Mem Limit, Notes | LimitRange Defaults |
| 4 | ConfigMaps | Name, Consumed By, Key Fields, Mount Method | — |
| 5 | Secrets & Expiry | Name, Consumed By, Keys, Backend, Rotation, Expiry | Rotation Architecture |
| 6 | Namespaces | Namespace, Components, Purpose | ResourceQuota |
| 7 | RBAC | Role Name, Kind, Namespace, Verbs, Resources, Bound To | ServiceAccounts |
| 8 | Inter-Service Communication | Source, Dest, Protocol, Port, Auth, Notes | NetworkPolicy Rules |
| 9* | Security Threat Model | Threat, Vector, Mitigation, K8s Control | — |
| 10* | Security Hardening Checklist | #, Control, Implementation, Mechanism, Status | — |
| 11 | Deployment Order | Step, Component, Namespace, Depends On, Notes | — |

*Sections 9 and 10 are generated only when `--security-level=high`

---

## Example Invocations

### Example 1: Simple Web App (Quick Mode)

**Input:**
```
/k8-planner my-blog frontend,backend-api,postgres
```

**What the skill does:**
1. Uses `my-blog` as app name
2. Sets frontend as Deployment (LoadBalancer), backend-api as Deployment (ClusterIP), postgres as StatefulSet (ClusterIP)
3. Infers medium security level, medium traffic
4. Generates all 8 sections + Deployment Order

---

### Example 2: AI Employee System (Security-Enhanced)

**Input:**
```
/k8-planner openclaw api-gateway,ai-core,tool-executor,memory-store,audit-logger,postgres,redis --security-level=high
```

**What the skill does:**
1. Generates all 8 sections with expanded RBAC (adds "What It CANNOT Do" column)
2. Expands Secrets section: ESO + Vault backend, volume-mount-only rule
3. Adds Section 9: Security Threat Model
4. Adds Section 10: Security Hardening Checklist (20+ controls)
5. Isolates tool-executor in its own namespace

---

### Example 3: Dev Environment Plan

**Input:**
```
/k8-planner ecommerce frontend,catalog-api,cart-api,order-api,postgres,redis,elasticsearch --env=dev --security-level=low
```

**What the skill does:**
1. Sets all replicas to 1
2. Halves all resource requests/limits
3. Uses `baseline` Pod Security Standard (not `restricted`)
4. Uses Kubernetes Secrets (no Vault/ESO)
5. Relaxes NetworkPolicy (no deny-all in dev)
6. Skips Sections 9 and 10

---

## Skill Flags Reference

| Flag | Values | Default | Effect |
|---|---|---|---|
| `--security-level` | `low`, `medium`, `high` | `medium` | Adjusts Secrets backend, RBAC depth, adds Threat Model + Checklist if `high` |
| `--env` | `dev`, `staging`, `production` | `production` | Scales replicas and resource sizes accordingly |
| `--export=yaml` | `yaml` | (off) | Also generates starter YAML manifest stubs for each section |
| `--no-hpa` | (flag) | (off) | Omits HPA sub-table from Section 1 |
| `--service-mesh` | `istio`, `linkerd` | (off) | Adds mTLS notes to Section 8 and service mesh config hints |
| `--monitoring` | `prometheus`, `datadog` | `prometheus` | Adds monitoring namespace and ServiceMonitor notes |
| `--compliance` | `soc2`, `hipaa`, `pci` | (off) | Adds compliance notes to Sections 5, 6, and 7 |

---

## Skill Design Notes

| Decision | Rationale |
|---|---|
| Tables for every section | Scannable at a glance; easy to diff between plans; renders well on GitHub |
| StatefulSet decision based on storage needs | The only reliable heuristic: if a pod needs stable identity + PVC, it's a StatefulSet |
| Default-deny NetworkPolicy always first | Zero-trust baseline; adding explicit allows is safer than removing broad denies |
| One ServiceAccount per component | Principle of least privilege; prevents cross-component secret access |
| Secrets via volume mounts (medium+) | Environment variables leak into `kubectl describe pod` output and crash dumps |
| Deployment order as final section | Dependencies are the #1 source of failed first-time cluster deployments |
| Security level controls depth, not accuracy | All plans are architecturally correct; security level controls how much detail and how many controls are shown |
