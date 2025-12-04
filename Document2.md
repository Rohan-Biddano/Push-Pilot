# `<!-- Image: HLD Document Header -->`

# Document 2 – HLD (High-Level Design)

**Thursday, 4 December 2025**
**11:43 AM**

**Project:** PushPilot
**Version:** 1.0

---

# 1. Architecture Overview

## 1.1 Logical Architecture

**Main components:**

- **Web Client (Frontend)** – React/Vue/etc. (out of scope for code, but consumes APIs).
- **API Server** – Django + DRF (REST APIs).
- **Worker Service** – Celery workers (scheduling & sending).
- **Database** – PostgreSQL.
- **Cache/Queue** – Redis.
- **External Services:**
  - Firebase Cloud Messaging (FCM)
  - AI Text Generation Provider

---

### Data flow (simplified):

1. User uses web app → auth via API → selects workspace.
2. User configures Firebase, creates campaign, and schedules it.
3. Backend saves campaign & creates scheduled_job.
4. Celery Beat / periodic task checks scheduled_jobs.
5. Celery Worker triggers send → uses Firebase client → FCM → user devices.
6. Analytics events collected back (v1 could be basic).

---

# 2. Module-Level Design

## 2.1 Django Apps

- **accounts** – Users, OAuth, JWT integration
- **workspaces** – Workspaces, members, roles
- **integrations.firebase** – Firebase configuration & messaging wrapper
- **campaigns** – Campaigns, campaign content, templates
- **notifications** – Notifications, scheduled jobs, Celery tasks
- **analytics** – Analytics events, dashboards
- **common** – shared utils, base models

---

# 3. Data Design (ER Overview)

**Core entities:**

- User (1) – (M) WorkspaceMember
- Workspace (1) – (M) WorkspaceMember
- Workspace (1) – (M) Campaign
- Campaign (1) – (M) CampaignContent
- Workspace (1) – (M) FirebaseConfig
- Campaign (1) – (M) Notifications
- Campaign (1) – (M) ScheduledJobs
- Campaign (1) – (M) AnalyticsEvents

_You already saw table fields earlier; this HLD just confirms relationships._

---

# 4. Component Interaction

## 4.1 Send Scheduled Campaign (Sequence)

1. User creates campaign & content → API saves to DB.
2. User schedules campaign → API creates entry in `scheduled_jobs`.
3. Celery Beat periodically runs `check_due_jobs`.
4. For each due job, Celery worker:
   - Loads campaign + content + Firebase config.
   - Builds notification payload.
   - Calls `FirebaseClient.send_message()` or `send_multicast()`.
   - Stores notifications records and status.

---

# 5. Deployment View

**Recommended GCP / AWS deployment:**

### Kubernetes or ECS:

- **Service A:** pushpilot-api (Django)
- **Service B:** pushpilot-worker (Celery workers)
- **Service C:** pushpilot-beat (Celery Beat / scheduler)

### External:

- PostgreSQL (Cloud SQL / RDS)
- Redis (Memorystore / ElastiCache)
- Object storage (GCS / S3)

---

# 6. Security Design (High-Level)

- **JWT Auth:** DRF + SimpleJWT
- **Encrypted fields** (Firebase server key) using a library like `django-encrypted-model-fields` or custom KMS encryption
- **Secrets stored** in cloud secret manager

---

# 7. Scalability & Fault Tolerance

- API layer horizontally scalable behind load balancer
- Celery workers scaled based on queue length
- If Firebase call fails (network or 5xx), Celery task retries with exponential backoff

---
