# `<!-- Image: High-Level Architecture Diagram -->`

# 1. High-Level Architecture

**Core idea:**PushPilot is a multi-tenant SaaS with:

- Web app (frontend) – talks to backend via REST APIs.
- Backend (Django + DRF) – business logic, APIs, auth, scheduling.
- Background workers (Celery) – send notifications, run schedules.
- Firebase – actual push delivery.
- Database (PostgreSQL) – users, workspaces, campaigns, logs.
- Redis – queues & caching.
- Cloud (AWS/GCP) – hosting, object storage, monitoring.

You can imagine it as:

---

# 2. Backend Logical Modules (Django Apps)

You can split Django project into these apps:

- **accounts** – users, auth, sessions, OAuth (Google/Apple)
- **workspaces** – workspaces, members, roles, billing later
- **integrations**
  - integrations.firebase – Firebase credentials & messaging
- **campaigns** – campaigns, content, templates
- **notifications** – notification instances, scheduling, logs
- **analytics** – metrics, stats, charts
- **common** – shared models, utils, base classes

---

# 3. Authentication & Authorization Model

## 3.1 Auth Type

Use JWT-based auth via Django REST Framework + SimpleJWT (or similar):

- Access token – short-lived
- Refresh token – longer-lived

## 3.2 User Authentication Flows

### **Email/Password**

- `POST /auth/register` → create user, send verification email (optional)
- `POST /auth/login` → returns access + refresh JWTs

### **Google/Apple Sign-In**

- `POST /auth/google` – send Google token → verify → create or login user
- `POST /auth/apple` – similar

## 3.3 Authorization

- **Multi-tenant via workspace**

  - Every request has current workspace context (e.g. `X-Workspace-ID` header or selected workspace stored in user profile).
  - All queries filtered by `workspace_id`.
- **Role-based:**

  - **Owner** – full control, billing, Firebase connect
  - **Admin** – manage campaigns, templates, members (except delete workspace)
  - **Member** – create/edit campaigns, view analytics
  - **Viewer (optional)** – read-only

---

# 4. Database Schema (Detailed, First Version)

A strong starting schema design.

---

## 4.1 `users`

| Field         | Type          | Notes                     |
| ------------- | ------------- | ------------------------- |
| id (PK)       | UUID/BigInt   |                           |
| email         | String (uniq) | login identity            |
| password_hash | String        | null if social login only |
| full_name     | String        | optional                  |
| auth_provider | Enum          | email, google, apple      |
| is_active     | Bool          |                           |
| created_at    | DateTime      |                           |
| updated_at    | DateTime      |                           |

---

## 4.2 `workspaces`

| Field         | Type          | Notes                 |
| ------------- | ------------- | --------------------- |
| id (PK)       | UUID/BigInt   |                       |
| name          | String        | e.g. “Biddano Tech” |
| slug          | String (uniq) | for URLs              |
| owner_id (FK) | users         | workspace creator     |
| plan_type     | Enum          | free, pro, enterprise |
| created_at    | DateTime      |                       |
| updated_at    | DateTime      |                       |

---

## 4.3 `workspace_members`

| Field               | Type        | Notes                        |
| ------------------- | ----------- | ---------------------------- |
| id (PK)             | UUID/BigInt |                              |
| user_id (FK)        | users       |                              |
| workspace_id   (FK) | workspaces  |                              |
| role                | Enum        | owner, admin, member, viewer |
| status              | Enum        | invited, active              |
| invited_at          | DateTime    |                              |
| joined_at           | DateTime    |                              |

---

## 4.4 `firebase_configs`

| Field                | Type        | Notes                         |
| -------------------- | ----------- | ----------------------------- |
| id (PK)              | UUID/BigInt |                               |
| workspace_id (FK)    | workspaces  |                               |
| project_id           | String      | Firebase project ID           |
| server_key_encrypted | String      | encrypted form                |
| is_active            | Bool        | only one active per workspace |
| created_at           | DateTime    |                               |
| updated_at           | DateTime    |                               |

---

## 4.5 `campaigns`

| Field                 | Type        | Notes                                           |
| --------------------- | ----------- | ----------------------------------------------- |
| id (PK)               | UUID/BigInt |                                                 |
| workspace_id (FK)     | workspaces  |                                                 |
| name                  | String      | “Diwali Delight Campaign”                     |
| description           | Text        | optional                                        |
| status                | Enum        | draft, scheduled, running, completed, cancelled |
| channel_type          | Enum        | app_push, web_push, sms, email, whatsapp        |
| tone                  | String/Enum | cheerful, festive, formal, etc.                 |
| schedule_type         | Enum        | send_now, scheduled, recurring                  |
| scheduled_at          | DateTime    | null if send_now                                |
| recurrence_rule       | Text        | RRULE syntax (optional)                         |
| created_by (FK users) |             |                                                 |
| updated_by (FK users) |             |                                                 |
| created_at            | DateTime    |                                                 |
| updated_at            | DateTime    |                                                 |

---

## 4.6 `campaign_contents`

| Field               | Type        | Notes                                   |
| ------------------- | ----------- | --------------------------------------- |
| id (PK)             | UUID/BigInt |                                         |
| campaign_id (FK)    | campaigns   |                                         |
| title               | String      | e.g. “Weekend Success Starts Tonight” |
| subtitle            | Text        | detailed message                        |
| language            | String      | e.g. en-IN                              |
| image_url           | String      | optional                                |
| tone                | String      | for analytics / reuse                   |
| segment_filter_json | JSON        | optional segmentation filter            |

---

## 4.7 `templates`

| Field             | Type        | Notes                               |
| ----------------- | ----------- | ----------------------------------- |
| id (PK)           | UUID/BigInt |                                     |
| workspace_id (FK) | Nullable    | null = global template              |
| name              | String      | “Self-Care Saturday”              |
| category          | String      | festive, weekend, retention, pharma |
| title             | String      |                                     |
| subtitle          | Text        |                                     |
| tone              | String      |                                     |
| is_global         | Bool        |                                     |
| created_at        | DateTime    |                                     |
| updated_at        | DateTime    |                                     |

---

## 4.8 `notifications`

Each row = notification sending instance.

| Field               | Type        | Notes                                     |
| ------------------- | ----------- | ----------------------------------------- |
| id (PK)             | UUID/BigInt |                                           |
| workspace_id (FK)   |             |                                           |
| campaign_id (FK)    |             |                                           |
| target_segment_json | JSON        | optional                                  |
| firebase_message_id | String      | returned by FCM                           |
| status              | Enum        | pending, scheduled, sending, sent, failed |
| scheduled_for       | DateTime    |                                           |
| sent_at             | DateTime    |                                           |
| failure_reason      | Text        |                                           |
| created_at          | DateTime    |                                           |

---

## 4.9 `scheduled_jobs`

| Field             | Type        | Notes                               |
| ----------------- | ----------- | ----------------------------------- |
| id (PK)           | UUID/BigInt |                                     |
| workspace_id (FK) |             |                                     |
| campaign_id (FK)  |             |                                     |
| run_at            | DateTime    | when Celery executes                |
| status            | Enum        | pending, running, completed, failed |
| attempts          | Integer     | retry count                         |
| last_error        | Text        |                                     |
| created_at        | DateTime    |                                     |

---

## 4.10 `analytics_events`

| Field                          | Type        | Notes                                            |
| ------------------------------ | ----------- | ------------------------------------------------ |
| id (PK)                        | UUID/BigInt |                                                  |
| workspace_id (FK)              |             |                                                  |
| campaign_id (FK)               |             |                                                  |
| notification_id (FK, nullable) |             |                                                  |
| event_type                     | Enum        | delivered, opened, clicked, failed, unsubscribed |
| external_user_id               | String      | optional                                         |
| occurred_at                    | DateTime    |                                                  |
| metadata_json                  | JSON        | device, OS, app version                          |

---

# 5. API Design Conventions (Path, Query, Body)

## 5.1 General Rules

- **Path params** → identify a specific resourceExample: `/campaigns/{id}/`, `/workspaces/{id}/members/`
- **Query params** → filtering, searching, sorting, paginationExample: `/campaigns?status=active&page=1&page_size=20`
- **Request body (JSON)** → create or update resources
  Example: POST/PUT/PATCH for campaigns, Firebase config, content

---

## 5.2 Examples

### List campaigns (with filters)

```md
GET /campaigns?status=active&page=1
```

### Create campaign

{
  "name": "Weekend Blast",
  "channel_type": "app_push",
  "schedule_type": "send_now"
}


### Generate AI content

POST /campaigns/{id}/generate-content
{
  "prompt": "Write a festive message"
}




### Connect Firebase

POST /workspaces/{id}/firebase
{
  "project_id": "demo-app",
  "server_key": "XYZ..."
}


# 6. Key API Groups & Sample Endpoints

## 6.1 Auth

* POST `/auth/register`
* POST `/auth/login`
* POST `/auth/refresh`
* POST `/auth/google`
* POST `/auth/apple`

## 6.2 Workspaces

* GET `/workspaces/`
* POST `/workspaces/`
* GET `/workspaces/{id}/`
* POST `/workspaces/{id}/members`
* GET `/workspaces/{id}/members`

## 6.3 Firebase Integration

* POST `/workspaces/{id}/firebase` – connect
* GET `/workspaces/{id}/firebase` – view (masked server key)
* PATCH `/workspaces/{id}/firebase` – update
* POST `/workspaces/{id}/firebase/test` – send test notification

## 6.4 Campaigns

* GET `/campaigns/`
* POST `/campaigns/`
* GET `/campaigns/{id}/`
* PATCH `/campaigns/{id}/`
* DELETE `/campaigns/{id}/`
* POST `/campaigns/{id}/generate-content`
* POST `/campaigns/{id}/schedule`
* POST `/campaigns/{id}/cancel`

## 6.5 Notifications

* GET `/notifications/`
* GET `/notifications/{id}/`
* POST `/notifications/send-now`

## 6.6 Templates

* GET `/templates/`
* POST `/templates/`
* GET `/templates/{id}/`
* PATCH `/templates/{id}/`
* DELETE `/templates/{id}/`

## 6.7 Analytics

### GET `/analytics/summary` returns:

* total_notifications_sent
* today_notifications
* next_notification_time
* active_campaign_count
* connected_applications_count

### GET `/analytics/campaign/{id}` returns:

* sent
* delivered
* opened
* clicked
* failed %

---

# 7. Scheduling & Background Processing

Using  **Celery + Redis** :

1. When a campaign is scheduled:

   → Create a `scheduled_jobs` row with `run_at = scheduled_at`
2. **Celery Beat** checks every minute:

   → Find jobs where `run_at <= now` & status = pending
3. For each due job:

   * Mark as running
   * Call Firebase sending logic
   * Create notification rows
   * Update status (sent/failed)
4. Recurring campaigns:

   * Compute next `run_at` using `recurrence_rule`
   * Create new scheduled job

---

# 8. Deployment Architecture (AWS/GCP)

**Components:**

* **Django API** – Docker container
  * AWS: ECS/EKS/Elastic Beanstalk
  * GCP: Cloud Run/GKE
* **Celery Worker** – separate container
* **Celery Beat** – scheduler container
* **PostgreSQL** – RDS / Cloud SQL
* **Redis** – ElastiCache / Memorystore
* **Static/Media** – S3 / GCS
* **Secrets** – AWS or GCP Secret Manager
* **Monitoring** – CloudWatch / Stackdriver
