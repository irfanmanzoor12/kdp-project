# Kubernetes Deployment Planning — AI-400 Class 22

> **Course:** AI-400 | **Class:** 22 | **Author:** Irfan Manzoor | **Date:** 2026-04-06

Kubernetes deployment plans for two AI application scenarios — no code, planning only. Includes a reusable Agent Skill that can generate K8s deployment plans for any future project.

---

## Repository Contents

| File | Description |
|---|---|
| [`plan1-ai-task-manager.md`](./plan1-ai-task-manager.md) | K8s deployment plan for AI Native Task Manager |
| [`plan2-ai-employee-openclaw.md`](./plan2-ai-employee-openclaw.md) | K8s deployment plan for AI Employee (OpenClaw) |
| [`k8-planning-skill.md`](./k8-planning-skill.md) | Reusable Agent Skill to generate K8s plans for any app |

---

## Plan 1: AI Native Task Manager

A cloud-native task management application with an AI agent that plans and prioritizes work.

### Components

| Component | Controller | Replicas | Public |
|---|---|---|---|
| UI Interface (nginx) | Deployment | 2–6 | Yes (LoadBalancer) |
| Backend API | Deployment | 3–10 | Via Ingress |
| Task Agent (LLM) | Deployment | 2–8 | No |
| Notification Service | Deployment | 1–3 | No |
| PostgreSQL | **StatefulSet** | 1 | No |
| Redis Cache | **StatefulSet** | 1 | No |

### Covers
- Pods, Deployments & StatefulSets (+ HPA rules)
- Services: ClusterIP, LoadBalancer, Ingress routing
- Resource Requests & Limits per component
- ConfigMaps (one per component)
- Secrets management with rotation policies (ESO + Vault)
- 5 Namespaces with ResourceQuota
- RBAC Roles, RoleBindings & ServiceAccounts
- Inter-service communication + NetworkPolicy (default-deny)
- Deployment order (12 steps)

---

## Plan 2: AI Employee — OpenClaw

A personal AI employee that autonomously executes tasks on behalf of users. Security is the primary design constraint.

### Components

| Component | Controller | Namespace | Security Note |
|---|---|---|---|
| API Gateway | Deployment | prod | Only public entry; rate limiting + JWT |
| Identity Service | Deployment | prod | Issues 1hr TTL tokens |
| OpenClaw Core | Deployment | prod | Cannot read its own secrets |
| Tool Executor | Deployment | **executor** (isolated) | gVisor sandbox runtime |
| Memory Store (Weaviate) | **StatefulSet** | prod | KMS-encrypted vector DB |
| Audit Logger | Deployment | prod | Append-only; write-only RBAC |
| PostgreSQL | **StatefulSet** | prod | SSL required |
| Redis Cache | **StatefulSet** | prod | Auth password required |

### Covers everything in Plan 1, plus
- Security Threat Model (8 threats: prompt injection, token theft, sandbox escape, etc.)
- Security Hardening Checklist (16 controls with checkboxes)
- gVisor sandboxed tool execution namespace
- Immutable append-only audit log
- AI self-escalation prevention (core SA cannot read its own secrets)
- mTLS between all internal services
- KMS-managed memory encryption (key never stored in etcd)

---

## K8 Planning Skill

A reusable Agent Skill (`k8-planner`) that generates complete Kubernetes deployment plans in table format for **any** application.

### How to Use

| Invocation | Mode |
|---|---|
| `/k8-planner` | Interactive — asks 10 questions, then generates |
| `/k8-planner my-app frontend,api,postgres` | Quick — generates immediately |
| `/k8-planner --security-level=high` | Adds Threat Model + Hardening Checklist |
| `/k8-planner --env=dev` | Scales down for dev environment |
| `/k8-planner --export=yaml` | Also outputs starter YAML manifests |

### What It Generates

| # | Section | Always? |
|---|---|---|
| 1 | Pods / Deployments / StatefulSets + HPA | Always |
| 2 | Services + Ingress rules | Always |
| 3 | Resource Requests & Limits | Always |
| 4 | ConfigMaps | Always |
| 5 | Secrets + Rotation Policy | Always |
| 6 | Namespaces + ResourceQuota | Always |
| 7 | RBAC Roles & ServiceAccounts | Always |
| 8 | Inter-Service Communication + NetworkPolicy | Always |
| 9 | Security Threat Model | `--security-level=high` only |
| 10 | Security Hardening Checklist | `--security-level=high` only |
| 11 | Deployment Order | Always |

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| StatefulSet only for storage-dependent pods | Needs stable hostname + PersistentVolumeClaim |
| Default-deny NetworkPolicy in all plans | Zero-trust baseline; explicit allow-list only |
| One ServiceAccount per component | Principle of least privilege |
| Secrets as volume mounts (not env vars) | Env vars leak in `kubectl describe pod` and crash dumps |
| Tool executor in isolated namespace + gVisor | Sandboxed AI code execution; kernel-level isolation |
| 1hr session token TTL in OpenClaw | Short blast radius if token is stolen |
| Append-only audit log | Tamper-evident record of all AI actions |

---

## Project Info

| Property | Value |
|---|---|
| Course | AI-400 |
| Class | 22 |
| Due Date | 2026-04-06 |
| Submission | Public GitHub repo with 3 markdown files |
| Optional | Eval for K8 Planning Skill using Skill Creator (Anthropic) |
