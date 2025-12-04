# `<!-- Image: SRS Document Header -->`

# Document 1 – SRS (Software Requirements Specification)

**Thursday, 4 December 2025**
**11:39 AM**

**Project:** PushPilot – AI-Powered Notification Campaign Platform
**Version:** 1.0
**Owner:** Biddano Tech

---

# 1. Introduction

## 1.1 Purpose

This SRS defines the functional and non-functional requirements of PushPilot, an AI-powered SaaS platform that enables businesses and developers to create, manage, and automate multi-channel notification campaigns.

It is intended for:

- Product managers
- Backend and frontend developers
- QA engineers
- DevOps engineers
- Stakeholders and management

---

## 1.2 Scope

PushPilot will provide:

- A web-based dashboard where users can:

  - Sign up, sign in (email, Google, Apple)
  - Create and manage workspaces
  - Connect their app via Firebase (Project ID, Server Key)
  - Use AI to generate notification content (title + subtitle, tone-aware)
  - Create, schedule, and run campaigns
  - View analytics (notifications sent, campaign performance, schedules)
- A backend service that:

  - Exposes REST APIs
  - Handles authentication & authorization
  - Stores configuration, campaigns, notifications, analytics logs
  - Schedules and sends notifications via Firebase

---

## 1.3 Definitions, Acronyms, Abbreviations

- **FCM** – Firebase Cloud Messaging
- **AI** – Artificial Intelligence (LLM-based generation)
- **SaaS** – Software as a Service
- **JWT** – JSON Web Token
- **DRF** – Django REST Framework

---

## 1.4 References

- Firebase Cloud Messaging Documentation
- Django, Django REST Framework documentation
- Celery documentation

---

# 2. Overall Description

## 2.1 Product Perspective

PushPilot is a multi-tenant SaaS platform. Each company/team has its own workspace.

The product sits between:

- The company’s apps (mobile/web)
- Firebase (for push delivery)
- AI provider (for generated content)

PushPilot does **not** send low-level device messages directly; it uses Firebase as the delivery layer.

---

## 2.2 Product Functions (High-Level)

1. User registration & authentication
2. Workspace creation & membership management
3. Firebase configuration (Project ID, Server Key) per workspace
4. AI text generation for notifications
5. Campaign creation, editing, duplication
6. Notification scheduling and sending
7. Viewing analytics and campaign performance
8. Template library for reusable messages

---

## 2.3 User Classes and Characteristics

### **Owner**

Creates workspace, manages billing (future), controls Firebase integration and members.

### **Admin**

Manages campaigns, templates, scheduling, members (except deleting workspace).

### **Member**

Creates and manages campaigns & templates, views analytics.

### **Viewer (optional)**

Read-only access to campaigns and analytics.

All users are assumed to be reasonably tech-savvy (developers, product, marketing).

---

## 2.4 Operating Environment

- Backend: Django + DRF, Python 3.x
- Database: PostgreSQL
- Cache/Queue: Redis
- Message scheduler: Celery
- Cloud: AWS or GCP (container-based deployment)

**External services:**

- Firebase Cloud Messaging
- AI text generation provider

---

## 2.5 Design and Implementation Constraints

- Must support JWT-based stateless authentication.
- Firebase Server Key must be stored encrypted.
- Must use HTTPS in production.
- Backend must be deployable on AWS or GCP.

---

## 2.6 Assumptions and Dependencies

- Client applications already use FCM SDK.
- Users can access their Firebase Console and retrieve credentials.
- AI provider is reliable and reachable from backend.

---

# 3. Specific Functional Requirements

## 3.1 Authentication & User Management

**FR-1:** The system shall allow users to register using email and password.
**FR-2:** Passwords must meet minimum length (>= 8 chars).
**FR-3:** The system shall allow login via email/password.
**FR-4:** The system shall issue JWT access and refresh tokens on login.
**FR-5:** The system shall allow login/register via Google OAuth.
**FR-6:** The system shall allow login/register via Apple OAuth (if configured).
**FR-7:** The system shall allow users to update their profile (name).
**FR-8:** The system shall allow users to reset password via email (optional v1.1).

---

## 3.2 Workspace Management

**FR-9:** A user shall be able to create a workspace with name and slug.
**FR-10:** A user shall be able to invite other users to a workspace.
**FR-11:** The system shall support roles: owner, admin, member, viewer.
**FR-12:** Users shall be able to switch between workspaces they belong to.

---

## 3.3 Firebase Integration

**FR-13:** A workspace owner/admin shall configure Firebase by providing project_id and server_key.
**FR-14:** The system shall store the Firebase server key in encrypted form.
**FR-15:** The system shall validate connectivity by sending a test notification (optional).

---

## 3.4 Campaign Management

**FR-16:** A user shall be able to create a campaign with:

- Name
- Description
- Channel type (app_push, web_push, etc.)
- Tone
- Schedule type (send_now, scheduled, recurring)
- Scheduled time (if scheduled)

**FR-17:** A user shall view a list of campaigns filtered by status.
**FR-18:** A user shall edit a campaign in draft or scheduled state.
**FR-19:** A user shall duplicate a campaign.
**FR-20:** A user shall cancel a scheduled campaign.

---

## 3.5 AI Content Generation

**FR-21:** The system shall provide an endpoint to generate content from a prompt.
**FR-22:** The system shall generate title (4–6 words) and subtitle (10–15 words).
**FR-23:** The system shall allow specifying tone (cheerful, festive, etc.).
**FR-24:** The system shall store AI-generated content linked to a campaign.

---

## 3.6 Templates

**FR-25:** The system shall provide global templates (pre-loaded).
**FR-26:** Users shall be able to create workspace-level templates.
**FR-27:** Users shall be able to copy from a template into a campaign.

---

## 3.7 Notifications & Scheduling

**FR-28:** The system shall schedule notifications based on campaign settings.
**FR-29:** The system shall create scheduled_jobs entries for each scheduled campaign.
**FR-30:** The system shall process due jobs and send notifications via Firebase.
**FR-31:** The system shall allow "send now" actions.
**FR-32:** The system shall track notification status (pending, sending, sent, failed).

---

## 3.8 Analytics & Reporting

**FR-33:** The system shall provide a workspace dashboard showing:

- Total notifications sent
- Daily sent count
- Active campaigns
- Next scheduled notification time

**FR-34:** The system shall provide campaign-level analytics:

- Sent
- Delivered
- Opened
- Clicked (if available)

**FR-35:** The system shall store analytics events per notification.

---

# 4. Non-Functional Requirements

## 4.1 Performance

- **NFR-1:** Support at least 10,000 notifications/min (scalable).
- **NFR-2:** Average API response < 300 ms under normal load.

---

## 4.2 Security

- **NFR-3:** All communication must use HTTPS.
- **NFR-4:** JWT tokens must be signed and have configurable expiry.
- **NFR-5:** Sensitive keys must be encrypted.
- **NFR-6:** RBAC must be enforced on all endpoints.

---

## 4.3 Reliability & Availability

- **NFR-7:** System should target ≥ 99.5% uptime.
- **NFR-8:** Failed sends should be retried (configurable retry policy).

---

## 4.4 Scalability

- **NFR-9:** Support horizontal scaling of API and worker nodes.

---

## 4.5 Usability

- **NFR-10:** UI must provide clear error messages for failed actions (e.g., Firebase misconfig).

---
